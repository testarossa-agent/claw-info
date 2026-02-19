# 訊息路由與 Channel Plugins（頻道外掛）

> 狀態：完整文件（v1）。本文件深入探討 OpenClaw 的訊息路由體系、Channel 插件架構、`message tool` 與 `sessions_send` 工具的差異、回覆標籤、主題/訊息線、投遞模式、安全邊界，並提供範例與排障指南。

---

## 1. TL;DR（500 字精華）

| 核心概念 | 說明 |
|----------|------|
| **路由模型** | 事件流：`Channel → Gateway Router → Session / Agent → Tool Calls`，完全由 Gateway 統一調度 |
| **Channel Plugins** | 每個平台（Telegram、Discord、WhatsApp）都是獨立外掛，實現 `ChannelProvider` interface，透過 WebSocket/Webhook/Long Polling 接入 |
| **`message tool`** | Agent 回覆使用者：發送訊息到 session 的 target（telegram 會用原 chat_id + message_id 回覆） |
| **`sessions_send`** | 跨 session 通訊：從一個 session 主動發訊息到另一個 session 的 target，常用於 cron / isolated agent 結果通知 |
| **回覆標籤（reply_tag）** | 用於建立「訊息線程」，讓 Gateway 知道這條訊息是回覆哪一個 upstream message |
| **主題（Topics）** | Telegram 頻道討論串 / Discord thread，用 topic_id（Discord）或 message_id（Telegram）標記 |
| **投遞模式（Delivery Modes）** | none（不投递）、announce（到預設 target）、target（指定 target），僅 isolated session 支援 |
| **安全邊界** | sessions_send 受 tools.allow/deny 控制；message tool 受 sandbox policy 與 elevated 權限影響 |
| **範例場景** | 日常 reply：message；cron 結果通知：sessions_send；跨 agent 通訊：sessions_send |

---

## 2. 為什麼這很重要

OpenClaw 的核心是「多 channel、多 session、多 agent」的訊息中樞。

---

## 3. 核心概念

### 3.1 Session 與 Target

| 概念 | 說明 | 範例 |
|------|------|------|
| **Session** | 一個「對話實體」：一次對話、一個 cron job、一個 isolated agent 執行 | Telegram DM 的 chat_id；Discord thread 的 thread_id |
| **Target** | Session 的「預設回覆目的地」：通常等於 channel + chat_id/thread_id | target = "telegram:123456789"（DM） |
| **Session Key** | Gateway 內部唯一識別：{agent_id}:{source_channel}:{chat_id/thread_id} | main:telegram:123456789 |

### 3.2 Channel Provider Interface（簡化版）

Channel plugins 必須實作的介面（概念化）：

```typescript
interface ChannelProvider {
  start(): Promise<void>;
  stop(): Promise<void>;
  onEvent(callback: (event: ChannelEvent) => void): void;
  sendMessage(params: SendMessageParams): Promise<MessageId>;
}
```

Gateway 會為每個 channel 建立實體，收到事件後解析成統一格式再推給 router。

---

## 4. 架構與心智模型

### 4.1 訊息從「收到」到「發出」的完整流動

```
使用者 → Channel Provider → Gateway Router → Session / Agent → Tool Calls → Gateway → Channel Plugin
```

---

## 5. `message tool` vs `sessions_send`

### 5.1 `message tool`（預設 reply）

**用途**：Agent 回覆**當前 session 的使用者**（預設 target）

**範例**：

```yaml
tool: message
params:
  text: "這是 Agent 回覆"
  target: "telegram:123456789"  # 可選：預設為 session.target
  reply_tag: "157"              # 可選：被回覆的 message_id
  topic_id: "456!"              # 可選：Discord thread_id
  parse_mode: "HTML"            # 可選：HTML / Markdown
  disable_web_page_preview: true
```

**Gateway 處理**：
1. 解析 `params.target`，若未指定 → 用 `session.target`
2. 記入 `session.state.pending_messages`
3. Tool 完成後 flush 到 channel

### 5.2 `sessions_send`（跨 session 通訊）

**用途**：從一個 session 主動向**另一個 session 的 target 發送訊息**

**範例**：

```yaml
tool: sessions_send
params:
  text: "Cron job #123 完成：已檢查 15 個 issues"
  target: "telegram:987654321"  # 指定 target（可以與當前 session 不同）
  reply_tag: "100"              # 可選：回覆某條訊息
  topic_id: "200!"              # 可選：Discord thread_id
```

**關鍵差異**：

| 對比維度 | `message` | `sessions_send` |
|---------|----------|----------------|
| 預設 target | session 的 target | 必須 Explicit 指定 target |
| 常用場景 | Agent 回覆當前使用者 | cron / isolated agent / 多 agent 通訊 |
| policy 影響 | 受 sandbox policy 與 elevated 影響 | 受 `tools.allow/deny` 控制 |
| session 關聯 | 屬於當前 session 的 tool call | 可跨 session（session A 調用 → session B 收到） |
| Gateway 處理 | 延後 flush 到 session.target | 立刻執行（找出 target 所在 session 或直接 send） |

---

## 6. 回覆標籤（reply_tag）與主題（Topics）

### 6.1 `reply_tag`（回覆哪條訊息）

**用途**：建立「回覆」關聯（Telegram 的 reply_to_message_id，Discord 的 message_reference）

| Channel | reply_tag 對應欄位 | 限制 |
|---------|-------------------|-----|
| Telegram | reply_to_message_id | 必須是同 chat_id 內的 message |
| Discord | message_reference.message_id | 必須是同 channel 內的 message |
| WhatsApp | 不支援（僅支援 quick reply buttons） | |

### 6.2 主題 / 討論串（topic_id / thread_id）

**用途**：把訊息發送到特定主題討論串

| Channel | 名稱 | 參數 | 範例 |
|---------|-----|------|-----|
| Telegram | Discussion group | topic_id | topic_id: "-1001234567890" |
| Discord | Thread | topic_id | topic_id: "456!" |
| WhatsApp | 不支援 | - | - |

---

## 7. 投遞模式（Delivery Modes）

**僅適用於 isolated session**

| 模式 | 說明 |
|------|------|
| none | 不投遞任何結果（預設） |
| announce | 把 isolated session 的最後一條訊息，投遞到 session 的 target |
| target | 指定任意 target |

### 範例：Cron job 把結果通知到指定頻道

```yaml
payload:
  kind: agentTurn
  message: "請列出 GitHub 的 open issues"
  target: "telegram:987654321@announce"
delivery:
  mode: "announce"
```

---

## 8. 安全邊界與 Policy

### `sessions_send` 的安全控制

- `tools.allow/deny`：控制 session 是否可以呼叫 `sessions_send` tool

### `message tool` 的安全控制

- Sandbox Mode：若 session sandbox.mode != "off"，message tool 可能受限

---

## 9. 常見工作流（Recipes）

### 9.1 日常 Telegram DM 互動

User → Channel → Gateway Router → Agent → message tool → Gateway → Channel → User

### 9.2 Cron job 發送每日摘要到特定頻道

1. Cron Scheduler 唤醒 → 啟動 isolated session
2. Agent 推理 → tool calls...
3. isolated session 結束 → Gateway flush 到 target

### 9.3 多 Agent 工作流（Manager → Worker → User）

Manager → sessions_send → Worker → sessions_send → Main → message tool → User

---

## 10. 故障排除（Troubleshooting）

### 10.1「收到使用者訊息，但沒有回覆」

**檢查清單**：
1. Gateway 有收到 ChannelEvent？
2. Session 找對了嗎？
3. Agent 有決定 call `message tool` 嗎？
4. `message tool` 的 target 正確嗎？
5. Channel Provider 有送出去嗎？

### 10.2 `sessions_send` 收不到訊息

**檢查清單**：
1. target 格式正確嗎？
2. Channel Provider 已正確啟動？
3. target 對應的 channel/chat 有允許 bot 發訊息嗎？

### 10.3 reply_tag 無效

**可能原因**：
1. message_id 已被删除
2. message_id 在不同 chat_id
3. Channel Provider 不支持 reply_to_message_id

### 10.4 訊息亂序 / 收到多條相同訊息

**可能原因**：
1. Channel Provider 重試失敗
2. Multiple sessions 使用同一 target
3. delivery.mode = "announce" + sessions_send 兩者並存

---

## 11. 運維檢查清單

```bash
openclaw gateway status          # Channel Adapter 狀態
openclaw sessions list           # 檢查當前 sessions
openclaw sessions show <key>     # 查看 pending messages
```

---

## 12. 附錄：常見問題（Q&A）

### Q1：`message tool` 與 `sessions_send` 選哪個？

- **用 `message tool`**：Agent 回覆當前對話的使用者
- **用 `sessions_send`**：cron job / isolated agent / 多 agent 通訊

### Q2：Telegram 限制（Rate Limits）

- 每秒最多 30 messages（same chat_id）
- 每分鐘最多 20 messages（same message thread）
- 解法：批次匯總訊息

---

## 13. 更新紀錄

- **2026-02-18**：初版，完整涵蓋 routing model、channel plugin 架構、message vs sessions_send、reply_tag、topics/delivery modes、safety、recipes、troubleshooting。

---

## 相關資源

- docs/core/gateway-lifecycle.md - Gateway 架構與狀態機
- docs/cron.md - Cron 調度與 isolated session
- docs/sandbox.md - Sandbox policy 與安全邊界
- docs/webhook.md - Webhook delivery
