# OpenClaw Cron 調度系統

## 概述

OpenClaw Cron 是一個內建的強大定時任務系統，讓您能夠以宣告式方式規劃 Agent 任務的執行時間。

**首次引入版本：** `2026.1.8`

---

## 解決的問題

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       傳統定時任務的痛點                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  痛點 1：環境不一致                                                         │
│    • Crontab 在不同機器上配置分散                                           │
│    • 環境變數與路徑難以管理                                                 │
│    • 跨系統部署困难                                                         │
│                                                                             │
│  痛點 2：狀態無從追蹤                                                       │
│    • 無法得知任務是否成功執行                                               │
│    • 錯誤日誌分散在各處                                                     │
│    • 無法重試失敗任務                                                       │
│                                                                             │
│  痛點 3：缺乏 Agent 整合                                                    │
│    • Crontab 只能執行 shell 指令                                           │
│    • 無法調用 Agent 的工具與知識                                            │
│    • 難以實現複杂業務邏輯                                                   │
│                                                                             │
│  痛點 4：安全性考量                                                         │
│    • Root 權限任務風險                                                      │
│    • 敏感資料暴露於環境變數                                                 │
│    • 無法精細控制 Agent 權限                                                │
│                                                                             │
│  痛點 5：監控與排錯困難                                                     │
│    • 無法查看歷史執行記錄                                                   │
│    • 錯誤訊息難以定位                                                       │
│    • 缺乏執行時隔離                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   OpenClaw Cron 帶來的價值                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ✓ 環境一致性：單一配置檔案，所有環境相同                                   │
│  ✓ 狀態追蹤：實時查看任務執行結果                                           │
│  ✓ Agent 整合：調用 Agent 的知識與工具                                      │
│  ✓ 代碼即配置：Git 維護配置，CI/CD 憑證                                     │
│  ✓ 安全隔離：可選 Sandbox 模式，權限最小化                                 │
│  ✓ 監控排錯：完整執行記錄，錯誤自動重試                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 架構

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OpenClaw Cron 架構圖                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐                                                         │
│   │   Gateway    │                                                         │
│   │  Cron Job    │                                                         │
│   │  Scheduler   │                                                         │
│   └──────┬───────┘                                                         │
│          │                                                                  │
│          ▼                                                                  │
│   ┌──────────────┐      ┌─────────────────────────────────────────┐        │
│   │ Schedule     │─────▶│  job.kind = systemEvent                 │        │
│   │   Engine     │      │  - Injects text into main session       │        │
│   │              │      │  - For simple reminders/notifications   │        │
│   └──────────────┘      └─────────────────────────────────────────┘        │
│                                                                             │
│          │                                                                  │
│          ▼                                                                  │
│   ┌──────────────┐      ┌─────────────────────────────────────────┐        │
│   │  Trigger     │─────▶│  job.kind = agentTurn                   │        │
│   │              │      │  - Spawns isolated sub-agent session    │        │
│   │              │      │  - Runs with dedicated agent            │        │
│   └──────────────┘      └─────────────────────────────────────────┘        │
│                                                                             │
│          │                                                                  │
│          ▼                                                                  │
│   ┌──────────────┐                                                         │
│   │   Session    │                                                         │
│   │  Delivery    │                                                         │
│   └──────┬───────┘                                                         │
│          │                                                                  │
│          ▼                                                                  │
│   ┌──────────────┐                                                         │
│   │   Channel    │                                                         │
│   │  Plugin      │                                                         │
│   └──────────────┘                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 核心概念

### 調度類型 (Schedule Types)

| 類型 | 描述 | 適用場景 |
|------|------|---------|
| `at` | 單次執行，在指定時間點執行 | 一次性任務、提醒 |
| `every` | 週期性執行，以固定間隔執行 | 定期檢查、輪詢 |
| `cron` | Cron 表達式，精確控制時間 | 複雜排程、系統任務 |

#### at - 單次執行

```yaml
schedule:
  kind: at
  at: "2026-02-20T10:00:00Z"  # ISO-8601 UTC timestamp
```

#### every - 週期性執行

```yaml
schedule:
  kind: every
  everyMs: 3600000  # 每小時 (毫秒)
  anchorMs: 1700000000000  # 可選：起始時間戳
```

#### cron - Cron 表達式

```yaml
schedule:
  kind: cron
  expr: "0 9 * * 1-5"  # 工作日早上 9 點
  tz: "America/New_York"  # 可選：時區
```

**支援的 Cron 格式：**
```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6, Sunday = 0)
│ │ │ │ │
* * * * *
```

### 貼 payload 類型 (Payload Types)

| 類型 | 描述 | sessionTarget |
|------|------|--------------|
| `systemEvent` | 將文字注入會話 | `main` |
| `agentTurn` | 啟動獨立 Agent 會話 | `isolated` |

#### systemEvent - 會話事件

```yaml
payload:
  kind: systemEvent
  text: "Read HEARTBEAT.md if it exists. Follow it strictly."
```

**特點：**
- 將文字作為系統訊息注入
- 適用於簡單提醒、通知
- 主要用于 `main` session

#### agentTurn - Agent 任务

```yaml
payload:
  kind: agentTurn
  message: "Please check the latest GitHub issues and summarize."
  model: "amazon-bedrock/qwen.qwen3-next-80b-a3b"
  thinking: "on"
  timeoutSeconds: 300
```

**特點：**
- 啟動獨立的 sub-agent session
- 可指定不同模型、thinking 設定
- 適用於複雜任務、隔離執行

### 傳送模式 (Delivery Modes)

只有 `isolated` session 支援 `delivery` 設定：

| 模式 | 描述 |
|------|------|
| `none` | 不傳送結果 |
| `announce` | 將結果傳送到 session 的預設頻道 |

```yaml
delivery:
  mode: announce
  channel: "telegram:176096071"
  to: "176096071"
  bestEffort: true
```

---

## 使用範例

### 範例 1：每日早晨提醒

**觸發時間：** 每个工作日早上 8:30 (EST)

```yaml
name: "每日工作開始提醒"
schedule:
  kind: cron
  expr: "30 8 * * 1-5"
  tz: "America/New_York"
payload:
  kind: systemEvent
  text: |
    主公早安！今天的工作已經準備就緒。
    
    已排程的任務：
    - 每日 GitHub issue 摘要
    - 天氣預報
    - 日曆事件提醒
```

### 範例 2：每小時 GitHub 檢查

**觸發時間：** 每小時整點

```yaml
name: "GitHub Issue 監控"
schedule:
  kind: every
  everyMs: 3600000  # 1 小時
payload:
  kind: agentTurn
  message: |
    請檢查 claw-info 倉庫的未解決 issue，
    滙總重要問題並報告狀態。
  model: "amazon-bedrock/qwen.qwen3-next-80b-a3b"
  timeoutSeconds: 120
delivery:
  mode: announce
```

### 範例 3：單次任務 - 定點執行

**觸發時間：** 2026 年 2 月 25 日下午 3 點

```yaml
name: "系統維護通知"
schedule:
  kind: at
  at: "2026-02-25T15:00:00-05:00"
payload:
  kind: systemEvent
  text: "⚠️ 系統即將進行維護，請儲存工作。"
```

### 範例 4：週末備份確認

**觸發時間：** 每週六和週日早上 10:00

```yaml
name: "週末備份確認"
schedule:
  kind: cron
  expr: "0 10 * * 0,6"
payload:
  kind: agentTurn
  message: |
    請檢查昨晚的備份狀態，確認所有重要資料已成功備份。
    報告：備份成功/失敗、 missing backups
  timeoutSeconds: 60
delivery:
  mode: announce
```

---

## 實作細節

### Cron Job 狀態

| 狀態 | 描述 |
|------|------|
| `pending` | 作業已建立，等待執行 |
| `running` | 正在執行中 |
| `completed` | 已成功完成 |
| `failed` | 執行失敗 |
| `skipped` | 被跳過（例如：上一次執行未完成） |

### 錯誤處理

Cron 作業失敗時的行為：

1. **重試機制：**
   - 作業失敗後會自動重試 3 次
   - 每次重試間隔 5 分鐘

2. **失敗通知：**
   - 可設定 `delivery` 至特定頻道
   - 包含錯誤訊息與堆疊追蹤

3. **查看記錄：**
   ```bash
   openclaw cron runs <job_id>
   ```

### 限制與最佳實踐

| 項目 | 限制 | 建議 |
|------|------|------|
| 作業超時 | 最長 1 小時 | 長時間任務使用 `timeoutSeconds` |

--- 

## 常見問題

### Q1: Cron 任務會在 Gateway 重啟後保持嗎？

**A:** 会。Cron 作業狀態會持久化到 Gateway db，重啟後繼續排程。

### Q2: 多個 Gateway 實例會重複執行嗎？

**A:** 不會。Cron Scheduler 是單例的，即使多 Gateway 實例也不會重複執行。

### Q3: 能否動態調整排程？

**A:** 可以。使用 `openclaw gateway restart` 重新加载配置。

---

## 更新紀錄

- **2026-02-16**：建立文件，涵蓋核心概念與範例

---

*最後更新：2026-02-16*
