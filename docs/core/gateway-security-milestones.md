# ClawJacked 漏洞：原因、修復與安全性演進歷程

## 概述

「ClawJacked」是指系列發生在 2026 年初，針對 OpenClaw Gateway 的關鍵安全性漏洞鏈。攻擊者可透過惡意網頁、跨來源 WebSocket 連線或注入惡意記憶，試圖「劫持」Agent 的執行權限與敏感資訊。

本文件記錄了從 `v2026.02.14` 到 `v2026.02.25` 的完整修復歷程，旨在為開發者提供安全性設計的背景參考。

---

## 漏洞分析：為什麼會發生 ClawJacked？

核心問題在於 **「權限邊界模糊」** 與 **「來源信用過度擴張」**：

1. **記憶中毒 (Memory Poisoning)**：Agent 預設信任召回的內容為「事實」，攻擊者可透過注入指令來引發間接指令攻擊。
2. **跨來源連線 (Origin Takeover)**：Gateway 的 WebSocket 未嚴格限制 Origin，惡意網站可在後台發起連線並操作受害者已配對過的 Session。
3. **身分冒充與繞過 (Bypass)**：利用 IPv6 multicast/mapped-IPv4 地址、符號連結（Symlink）或 Shell 元字符，繞過 SSRF 檢查與 `execApproval` 批准機制。

---

## 修復歷程與技術細節

### 🛑 階段一：防禦外部注入 (v2026.02.14)
*   **記憶體去毒化**：將召回記憶（Recalled Memories）強制標記為 `untrusted` 內容。
*   **Webhook 強制驗證**：要求所有 Inbound Webhook（如 Telegram, Twilio）必須配置 `webhookSecret` 並進行簽名檢查，否則拒絕啟動。
*   **路徑邊界保護**：為 `apply_patch` 與 FS 工具強制執行工作區根目錄（Workspace-root）邊界檢查。

### 🛡️ 階段二：系統結構加固 (v2026.02.15)
*   **雜湊算法升級**：將沙箱緩存雜湊從 SHA-1 升級為 **SHA-256**。
*   **容器隔離強化**：明確禁止危險的 Docker 設定（如 host network、unconfined seccomp），防範容器逃逸。
*   **自動化脫敏**：在日誌與狀態回應中自動脫敏 Bot Token 與內部系統路徑。

### ⚓ 階段三：全面主權防衛 (v2026.02.25)
*   **WebSocket 跨來源防護**：
    *   對瀏覽器連線強制執行 **Origin Check** 與 **Password Throttling**。
    *   封鎖靜默自動配對（Silent Auto-pairing）路徑。
*   **指令路徑綁定**：將 `system.run` 的批准（Approval）與正規化後的執行絕對路徑與精確 argv 綁定，徹底防堵利用 Symlink 或空格路徑的繞過。
*   **反應事件授權 (Reaction Ingress)**：所有渠道（Signal, Discord, Slack 等）的反應事件在處理前，必須通過與訊息同等的 DM/Group 授與檢查。
*   **SSRF 防護完善**：將 IPv6 multicast 字面量（`ff00::/8`）納入封鎖範圍。

---

## 經驗教訓與最佳實踐

1. **零信任原則 (Zero Trust)**：不再區分「內部記憶」或「外部訊息」，所有輸入在轉換為 Action 前皆必須視為不受信任。
2. **顯式選擇 (Explicit Opt-in)**：敏感功能（如 `autoCapture`）應預設關閉，並由使用者明確開啟。
3. **路徑正規化**：檔案與指令的路徑應在驗證前進行正規化（Normalization），以防止編碼或連結攻擊。
4. **安全失敗 (Fail-Closed)**：當安全性配置（如 Token）缺失時，系統應選擇停止服務（Fail-closed），而非降級運行。

---

## 相關連結
- [OpenClaw v2026.02.14 發佈說明](../../release-notes/2026-02-14.md)
- [OpenClaw v2026.02.15 發佈說明](../../release-notes/2026-02-15.md)
- [OpenClaw v2026.02.25 發佈說明](../../release-notes/2026-02-25.md)
