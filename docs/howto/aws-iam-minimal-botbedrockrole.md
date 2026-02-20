# OpenClaw AWS IAM 最小權限配置（BotBedrockRole）

本文提供一份可分享的 **OpenClaw 最小 IAM 權限配置**範例，目標是讓 OpenClaw 能：

- 讀取帳單與成本資訊（唯讀）
- 讀取 CloudWatch 指標/日誌（唯讀）
- 呼叫 Amazon Bedrock model inference（Invoke）
- 使用 Amazon Bedrock AgentCore Browser（建立/連線/關閉 session，含 Live View 與 profile persistence）

> 注意：請依你的風險偏好調整 `Resource` 範圍與條件（Condition）。本文以「先跑通」為主，使用較寬鬆的 `"*"`。

---

## 1) 角色：BotBedrockRole

此配置以 **BotBedrockRole** 為例。

---

## 2) AWS 託管策略（Managed Policies）

建議附加 2 個 AWS managed policies：

- `AWSBillingReadOnlyAccess`
  - 用途：帳單/成本唯讀
- `CloudWatchReadOnlyAccess`
  - 用途：CloudWatch 唯讀

---

## 3) 內嵌策略（Inline Policy）

將以下 JSON 作為 role 的 inline policy（或改為 customer managed policy 後再 attach 也可以）：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BedrockInvoke",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AgentCoreBrowserMinimal",
      "Effect": "Allow",
      "Action": [
        "bedrock-agentcore:StartBrowserSession",
        "bedrock-agentcore:ConnectBrowserAutomationStream",
        "bedrock-agentcore:StopBrowserSession",
        "bedrock-agentcore:ConnectBrowserLiveViewStream",
        "bedrock-agentcore:GetBrowserProfile",
        "bedrock-agentcore:SaveBrowserSessionProfile"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 4) 備註與建議

- 若你只需要 **browser automation**（不需要 Live View），可以移除：
  - `bedrock-agentcore:ConnectBrowserLiveViewStream`
- 若你不需要 **profile persistence**（cookies/localStorage 跨 session），可以移除：
  - `bedrock-agentcore:GetBrowserProfile`
  - `bedrock-agentcore:SaveBrowserSessionProfile`
- 建議後續逐步收斂：
  - `Resource` 改為特定 ARN
  - 加入條件（例如限制 region、來源 VPC、tag 條件等）

---

## 5) 參考

- Amazon Bedrock AgentCore（service prefix: `bedrock-agentcore`）權限清單：
  - https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonbedrockagentcore.html
