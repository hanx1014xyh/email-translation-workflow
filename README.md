# Email Translation Workflow

一个基于 n8n 的邮箱监听与自动翻译工作流，支持英文邮件自动翻译为中文并推送通知，同时支持用中文回复并以英文发出，实现跨语言邮件通信。

## 项目架构

本项目包含两条核心链路：

### 链路一：英文邮件接收 → 中文通知推送

Email Trigger (IMAP) → 英文转中文 (DeepSeek LLM) → Switch路由 → 企微/飞书/钉钉机器人通知


- **邮箱监听**：通过 IMAP 协议实时监听 Gmail 收件箱，自动获取新邮件
- **智能翻译**：调用 DeepSeek 大语言模型，将英文邮件的主题和正文翻译为中文
- **多渠道通知**：通过 Switch 节点实现渠道路由，支持企业微信、飞书、钉钉机器人推送

### 链路二：中文回复 → 英文邮件发送

Webhook (接收中文回复) → 中文转英文 (DeepSeek LLM) → Send Email (SMTP发送)


- **接收回复**：通过 Webhook 接收用户的中文回复内容
- **智能翻译**：调用 DeepSeek 将中文回复翻译为英文
- **邮件发送**：通过 SMTP 将翻译后的英文内容发送给原始发件人

## 整体流程图

### 链路一：接收与通知

[Gmail收件箱] → [IMAP监听] → [DeepSeek英转中] → [Switch路由] → [企微 / 飞书 / 钉钉 机器人]


### 链路二：回复与发送

[机器人中文回复] → [Webhook] → [DeepSeek中转英] → [SMTP发送] → [原始发件人收到回复]


## 多渠道扩展性设计

通知推送采用 Switch + HTTP Request 的架构设计，通过统一的消息路由机制实现多渠道扩展：

| 渠道 | Webhook URL格式 | 消息体格式 |
|------|----------------|-----------|
| 企业微信 | `https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key={KEY}` | `{"msgtype":"text","text":{"content":"..."}}` |
| 飞书 | `https://open.feishu.cn/open-apis/bot/v2/hook/{TOKEN}` | `{"msg_type":"text","content":{"text":"..."}}` |
| 钉钉 | `https://oapi.dingtalk.com/robot/send?access_token={TOKEN}` | `{"msgtype":"text","text":{"content":"..."}}` |

扩展新渠道只需两步：
1. 在 Switch 节点添加新的路由规则
2. 添加对应的 HTTP Request 节点配置 Webhook URL 和消息格式

## 技术栈

- **工作流引擎**：n8n（可视化自动化平台）
- **邮件协议**：IMAP（收件）/ SMTP（发件）
- **翻译模型**：DeepSeek Chat API（兼容 OpenAI 格式）
- **通知渠道**：企业微信 / 飞书 / 钉钉 Webhook 机器人

## 使用方法

### 前置条件

- n8n 实例（Cloud 或 Self-hosted）
- Gmail 账号（需开启 IMAP 和应用专用密码）
- DeepSeek API Key
- 企微/飞书/钉钉机器人 Webhook URL（按需配置）

### 部署步骤

1. 在 n8n 中导入 `workflow.json`
2. 配置 IMAP credential（Gmail 邮箱 + 应用专用密码）
3. 配置 SMTP credential（Gmail 邮箱 + 应用专用密码）
4. 配置 DeepSeek API Key（Header Auth: `Authorization: Bearer {YOUR_KEY}`）
5. 将通知节点中的 Webhook URL 替换为真实的机器人地址
6. 激活 workflow

## 后续优化方向

- **上下文串联**：通过数据库或缓存存储邮件上下文（发件人、主题、邮件ID），实现完整的回复闭环，自动将回复发送给原始发件人
- **多轮对话**：基于邮件 Message-ID 和 In-Reply-To 头部实现邮件线程关联
- **HTML邮件支持**：当前处理纯文本邮件，后续可扩展支持 HTML 格式邮件的解析与翻译
- **错误处理**：添加翻译失败重试、邮件发送失败告警等异常处理机制
- **语言自动检测**：自动识别邮件语言，非英文邮件跳过翻译直接通知
