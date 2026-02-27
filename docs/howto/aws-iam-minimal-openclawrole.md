# OpenClaw AWS IAM 最小權限配置（OpenClawRole）

本文提供 **OpenClaw 最小 IAM 權限配置**範例。OpenClaw 本身不直接呼叫 Amazon Bedrock — 模型存取透過 OpenClaw 的 provider 設定處理。此角色的唯一用途是讓 openclaw 在啟動時從 AWS Secrets Manager 讀取 API key。

---

## 1) 角色：OpenClawRole

此配置以 **OpenClawRole** 為例，透過 AWS IAM Identity Center（SSO）permission set 管理。

---

## 2) AWS 託管策略（Managed Policies，選用）

若需要帳單與監控查詢，可附加：

- `AWSBillingReadOnlyAccess` — 帳單/成本唯讀
- `CloudWatchReadOnlyAccess` — CloudWatch 唯讀

> 這兩個 policy 對 openclaw gateway 本身**非必要**，僅供成本查詢工具使用。

---

## 3) 內嵌策略（Inline Policy）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OpenClawSecretsRead",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:<region>:<account-id>:secret:openclaw/*"
    }
  ]
}
```

> 將 `<region>` 與 `<account-id>` 替換為實際值。建議 Resource 限定 `openclaw/*` 前綴，避免過度授權。

---

## 4) 備註與建議

- 若 secrets 存放在多個 region，需為每個 region 各加一條 Statement
- 不需要 `secretsmanager:CreateSecret` / `PutSecretValue` — 那些是初始建立 secret 時用的，日常運作不需要
- 不需要任何 `bedrock:*` 權限

---

## 5) 給 Agent 的 Prompt（產生 Role 建立指南）

```text
請根據這份文件，用 AWS CLI 產生「OpenClawRole」的建立操作指南，內容需包含：
1) 建立 permission set（AWS IAM Identity Center）
2) 附加 inline policy（JSON 如文件所示：OpenClawSecretsRead）
3) 選用：附加 AWSBillingReadOnlyAccess、CloudWatchReadOnlyAccess
4) 將 permission set 指派給對應的 SSO user 並 provision
5) 建立後的驗證步驟：用 aws secretsmanager get-secret-value 測試是否成功

注意：在我確認之前，不要直接對 AWS 執行任何寫入操作；先輸出所有指令讓我 review。
文件：https://github.com/thepagent/claw-info/blob/main/docs/howto/aws-iam-minimal-openclawrole.md
```

---

## 6) 參考

- [OpenClaw External Secrets 文件](../external_secrets.md)
- [AWS Secrets Manager 權限清單](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awssecretsmanager.html)
