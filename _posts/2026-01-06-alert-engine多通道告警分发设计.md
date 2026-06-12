---
layout: post
title: "告警引擎多通道告警分发设计"
date: 2026-01-06
categories: [golang, 架构]
---

告警的最终价值在于**通知**——只有将告警信息及时、准确地送达相关人员，问题才能得到响应和解决。本章将深入 alert-engine 的告警分发模块，探讨其如何支持 Webhook、邮件、钉钉等多种通知渠道，以及如何通过接口抽象实现灵活扩展。

## 分发器职责与设计目标

告警分发器（Dispatcher）承担以下职责：

1. **多渠道支持**：支持 Webhook、邮件、钉钉等通知渠道，未来可扩展企业微信、短信、Slack 等
2. **灵活配置**：每条规则可独立配置通知渠道，渠道配置作为规则的一部分存储
3. **可靠投递**：支持重试机制，避免因网络抖动导致告警丢失
4. **消息格式化**：根据不同渠道的特性，生成符合其规范的告警消息格式
5. **安全签名**：支持钉钉等渠道的消息签名验证，防止伪造

## 分发器接口设计

为了支持多渠道和便于扩展，我们定义了清晰的接口：

```go
type Dispatcher interface {
    Notify(ctx context.Context, alert model.Alert, channels []model.NotificationChannel) error
}
```

`NotificationChannel` 定义了通道类型和配置：

```go
type NotificationChannel struct {
    Type   string            // channel type: webhook / email / dingtalk
    Config map[string]string // channel-specific config
}
```

## ChannelDispatcher 实现

`ChannelDispatcher` 是当前的默认实现，支持 Webhook、邮件、钉钉三种渠道。

```go
func (d *ChannelDispatcher) Notify(
    ctx context.Context,
    alert model.Alert,
    channels []model.NotificationChannel,
) error {
    if len(channels) == 0 {
        return nil
    }

    for _, ch := range channels {
        switch ch.Type {
        case "webhook":
            if err := d.sendWebhook(ctx, alert, ch.Config); err != nil {
                return err
            }
        case "email":
            if err := d.sendEmail(ctx, alert, ch.Config); err != nil {
                return err
            }
        case "dingtalk":
            if err := d.sendDingTalk(ctx, alert, ch.Config); err != nil {
                return err
            }
        default:
            return fmt.Errorf("unsupported channel type: %s", ch.Type)
        }
    }
    return nil
}
```

## 一、Webhook 通知

Webhook 是最通用的通知方式——将告警 JSON 推送到指定的 HTTP 端点。

### Webhook 配置

```go
type WebhookConfig struct {
    URL        string            // 全局默认 Webhook URL
    Method     string            // HTTP 方法，默认 POST
    Headers    map[string]string // 自定义请求头
    Secret     string            // 签名密钥，用于 HMAC-SHA256 签名
    Timeout    time.Duration     // 请求超时
    RetryCount int               // 重试次数
    RetryDelay time.Duration     // 重试间隔
}
```

### 设计要点

1. **全局配置 + 通道覆盖**：全局配置作为默认值，规则级别可单独指定 URL、Method、Headers
2. **HMAC 签名**：如果配置了 Secret，会在请求头中添加 `X-Signature`，接收方可用于验签
3. **指数退避（可选）**：当前实现是固定间隔重试，可改进为指数退避

## 二、邮件通知

邮件是传统的告警通知方式，至今仍被广泛使用。

### 邮件配置

```go
type EmailConfig struct {
    SMTPHost string // SMTP 服务器地址
    SMTPPort int    // 端口（25/465/587）
    Username string // 用户名
    Password string // 密码
    From     string // 发件人
    To       string // 默认收件人
    UseTLS   bool   // 是否使用 TLS
}
```

### 设计要点

1. **明文密码风险**：当前将 SMTP 密码以明文存储在配置中，生产环境应使用密钥管理服务（KMS）
2. **邮件格式**：纯文本格式，结构简单但信息完整
3. **TLS 支持**：显式区分 25 端口（无 TLS）和 465 端口（SSL/TLS）

## 三、钉钉通知

钉钉是企业内部最常用的即时通讯工具，其机器人 Webhook 机制允许自定义消息推送。

### 钉钉配置

```go
type DingTalkConfig struct {
    WebhookURL string // 全局默认 Webhook URL
    Secret     string // 加签密钥
    AtMobiles  string // @手机号列表，逗号分隔
    IsAtAll    bool   // @所有人
}
```

### 钉钉签名算法

钉钉的加签机制基于 HMAC-SHA256：

```go
func generateDingTalkSign(secret string, timestamp int64) string {
    stringToSign := fmt.Sprintf("%d\n%s", timestamp, secret)
    h := hmac.New(sha256.New, []byte(secret))
    h.Write([]byte(stringToSign))
    return hex.EncodeToString(h.Sum(nil))
}
```

### 设计要点

1. **签名验证**：钉钉 Webhook 支持加签模式，安全性更高
2. **@功能**：支持 @指定人员或 @所有人
3. **Markdown 格式**：比纯文本更美观，信息层次更清晰

## 四、规则与通知渠道的绑定

在规则定义中，`Channels` 字段直接内嵌了通知渠道配置：

```json
{
  "name": "HighMemoryUsage",
  "expr": "avg(container_memory_usage_bytes) / 1024 / 1024 > 1024",
  "severity": "warning",
  "interval": "60s",
  "channels": [
    {
      "type": "webhook",
      "config": {
        "url": "https://hooks.example.com/alert"
      }
    },
    {
      "type": "dingtalk",
      "config": {
        "webhook_url": "https://oapi.dingtalk.com/robot/send?access_token=xxx",
        "at_mobiles": "13800138000"
      }
    }
  ]
}
```

这种设计让告警通知配置与告警规则同生命周期——创建规则时一并配置通知渠道，删除规则时一并删除，无需额外管理。

## 总结

告警分发模块的设计要点：

- **接口抽象**：统一的 `Dispatcher` 接口，支持多实现
- **配置分层**：全局默认配置 + 规则级覆盖配置
- **多渠道支持**：Webhook（通用）、邮件（传统）、钉钉（企业IM）
- **签名安全**：钉钉 HMAC 签名支持
- **可扩展架构**：新增渠道只需添加新 case，不影响现有代码

通过这套分发机制，alert-engine 能够将告警信息及时送达值班人员手中，完成告警系统的闭环。