# Webhook 指南

本文档介绍如何接收和处理积分商城的 Webhook 事件通知。

---

## 目录

1. [概述](#1-概述)
2. [配置 Webhook](#2-配置-webhook)
3. [事件类型](#3-事件类型)
4. [Webhook 请求](#4-webhook-请求)
5. [签名验证](#5-签名验证)
6. [响应要求](#6-响应要求)
7. [重试机制](#7-重试机制)
8. [最佳实践](#8-最佳实践)

---

## 1. 概述

### 1.1 什么是 Webhook

Webhook 是一种 HTTP 回调机制，当特定事件发生时（如订单状态变更），积分商城会主动向您配置的 URL 发送 HTTP 请求，通知您事件详情。

### 1.2 适用场景

- 订单状态变更通知（创建、支付、发货、完成）
- 库存变动提醒
- 积分扣减/退还通知
- 异步操作结果通知

### 1.3 工作流程

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   积分商城      │    │    Webhook      │    │   您的服务器    │
│   服务器        │    │    事件发生     │    │                 │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         │  1. 事件触发          │                      │
         │<─────────────────────┤                      │
         │                      │                      │
         │  2. POST /your-webhook-url                  │
         │─────────────────────────────────────────────>│
         │                      │                      │
         │                      │  3. 验证签名          │
         │                      │  4. 处理业务逻辑      │
         │                      │                      │
         │  5. 返回 200 OK                             │
         │<─────────────────────────────────────────────│
         │                      │                      │
```

---

## 2. 配置 Webhook

### 2.1 通过 API 配置

在创建或更新租户配置时设置 Webhook URL：

```http
PUT /openapi/points/tenants/{tenantCode}
Content-Type: application/json

{
    "webhook_url": "https://your-domain.com/webhook/points-mall",
    "webhook_secret": "your-webhook-secret",
    "points_mode": "POINTS_MODE_WEBHOOK"
}
```

### 2.2 配置参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `webhook_url` | string | 是 | Webhook 接收地址，必须是 HTTPS |
| `webhook_secret` | string | 是 | 用于验证 Webhook 签名的密钥 |
| `points_mode` | string | 是 | 积分模式，设置为 `POINTS_MODE_WEBHOOK` |

### 2.3 URL 要求

- 必须使用 HTTPS（生产环境）
- 必须可从公网访问
- 响应时间不超过 5 秒
- 返回 HTTP 2xx 状态码表示成功

---

## 3. 事件类型

### 3.1 订单相关事件

| 事件类型 | 说明 | 触发时机 |
|----------|------|----------|
| `order.created` | 订单创建 | 用户下单成功后 |
| `order.paid` | 订单支付成功 | 积分扣减成功后 |
| `order.shipped` | 订单已发货 | 管理员填写物流信息后 |
| `order.completed` | 订单完成 | 用户确认收货后 |
| `order.cancelled` | 订单取消 | 订单被取消后 |
| `order.refunded` | 订单退款 | 积分退还后 |

### 3.2 积分相关事件

| 事件类型 | 说明 | 触发时机 |
|----------|------|----------|
| `points.deducted` | 积分扣减 | 订单支付时 |
| `points.refunded` | 积分退还 | 订单取消/退款时 |

### 3.3 库存相关事件

| 事件类型 | 说明 | 触发时机 |
|----------|------|----------|
| `stock.low` | 库存预警 | 库存低于阈值时 |
| `stock.empty` | 库存耗尽 | 库存为 0 时 |

---

## 4. Webhook 请求

### 4.1 请求格式

积分商城向您的 Webhook URL 发送 POST 请求：

```http
POST /webhook/points-mall HTTP/1.1
Host: your-domain.com
Content-Type: application/json
X-Webhook-ID: evt_abc123def456
X-Webhook-Timestamp: 1704067200
X-Webhook-Signature: a1b2c3d4e5f67890...

{
    "event_id": "evt_abc123def456",
    "event_type": "order.paid",
    "created_at": "2024-01-01T12:00:00Z",
    "data": {
        "order_id": "ORD_20240101_001",
        "user_id": "U123456",
        "product_id": "PROD_001",
        "product_name": "积分商品A",
        "quantity": 1,
        "points_cost": 100,
        "status": "PAID",
        "created_at": "2024-01-01T11:55:00Z",
        "paid_at": "2024-01-01T12:00:00Z"
    }
}
```

### 4.2 请求头

| Header | 说明 |
|--------|------|
| `Content-Type` | 固定为 `application/json` |
| `X-Webhook-ID` | 事件唯一标识 |
| `X-Webhook-Timestamp` | 事件时间戳（Unix 秒） |
| `X-Webhook-Signature` | 请求签名 |

### 4.3 请求体字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `event_id` | string | 事件唯一标识 |
| `event_type` | string | 事件类型 |
| `created_at` | string | 事件创建时间（ISO 8601） |
| `data` | object | 事件详细数据 |

---

## 5. 签名验证

### 5.1 签名算法

Webhook 签名使用 HMAC-SHA256 算法：

```
SignString = X-Webhook-Timestamp + "." + RequestBody
Signature = HMAC-SHA256(WebhookSecret, SignString)
```

### 5.2 验证步骤

1. 获取请求头中的 `X-Webhook-Timestamp` 和 `X-Webhook-Signature`
2. 构造签名字符串：`timestamp + "." + requestBody`
3. 使用您的 `webhook_secret` 计算 HMAC-SHA256
4. 比对计算结果与请求签名

### 5.3 代码示例

#### Python

```python
import hmac
import hashlib

def verify_webhook_signature(
    timestamp: str,
    body: str,
    signature: str,
    webhook_secret: str
) -> bool:
    """验证 Webhook 签名"""
    sign_string = f"{timestamp}.{body}"
    expected_signature = hmac.new(
        webhook_secret.encode('utf-8'),
        sign_string.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected_signature, signature)

# Flask 示例
from flask import Flask, request, jsonify

app = Flask(__name__)
WEBHOOK_SECRET = "your-webhook-secret"

@app.route('/webhook/points-mall', methods=['POST'])
def webhook_handler():
    # 获取签名信息
    timestamp = request.headers.get('X-Webhook-Timestamp')
    signature = request.headers.get('X-Webhook-Signature')
    body = request.get_data(as_text=True)

    # 验证签名
    if not verify_webhook_signature(timestamp, body, signature, WEBHOOK_SECRET):
        return jsonify({"error": "Invalid signature"}), 401

    # 处理事件
    event = request.get_json()
    event_type = event.get('event_type')

    if event_type == 'order.paid':
        handle_order_paid(event['data'])
    elif event_type == 'order.shipped':
        handle_order_shipped(event['data'])
    # ... 处理其他事件

    return jsonify({"status": "ok"}), 200
```

#### Go

```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "io"
    "net/http"
)

var webhookSecret = "your-webhook-secret"

func verifyWebhookSignature(timestamp, body, signature string) bool {
    signString := timestamp + "." + body
    h := hmac.New(sha256.New, []byte(webhookSecret))
    h.Write([]byte(signString))
    expectedSignature := hex.EncodeToString(h.Sum(nil))
    return hmac.Equal([]byte(expectedSignature), []byte(signature))
}

func webhookHandler(w http.ResponseWriter, r *http.Request) {
    // 获取签名信息
    timestamp := r.Header.Get("X-Webhook-Timestamp")
    signature := r.Header.Get("X-Webhook-Signature")

    body, _ := io.ReadAll(r.Body)

    // 验证签名
    if !verifyWebhookSignature(timestamp, string(body), signature) {
        http.Error(w, "Invalid signature", http.StatusUnauthorized)
        return
    }

    // 处理事件
    var event map[string]interface{}
    json.Unmarshal(body, &event)

    eventType := event["event_type"].(string)
    switch eventType {
    case "order.paid":
        handleOrderPaid(event["data"])
    case "order.shipped":
        handleOrderShipped(event["data"])
    }

    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}
```

---

## 6. 响应要求

### 6.1 成功响应

返回 HTTP 2xx 状态码表示成功接收：

```json
{
    "status": "ok"
}
```

### 6.2 失败响应

返回非 2xx 状态码或超时，系统将进行重试。

### 6.3 响应超时

- 超时时间：5 秒
- 超时后视为失败，触发重试

---

## 7. 重试机制

### 7.1 重试策略

当 Webhook 发送失败时，系统采用指数退避策略重试：

| 重试次数 | 等待时间 |
|----------|----------|
| 第 1 次 | 1 分钟后 |
| 第 2 次 | 5 分钟后 |
| 第 3 次 | 30 分钟后 |
| 第 4 次 | 2 小时后 |
| 第 5 次 | 24 小时后 |

最多重试 5 次，全部失败后事件标记为失败。

### 7.2 手动重试

通过 API 手动重试失败的 Webhook：

```http
POST /openapi/points/webhook/events/{eventId}/retry
```

### 7.3 查询事件

查询 Webhook 事件状态：

```http
GET /openapi/points/webhook/events?status=FAILED&page=1&page_size=10
```

---

## 8. 最佳实践

### 8.1 幂等性处理

同一事件可能被发送多次（因重试机制），请确保处理逻辑是幂等的：

```python
def handle_order_paid(data):
    order_id = data['order_id']

    # 检查是否已处理
    if order_already_processed(order_id):
        return  # 跳过重复处理

    # 处理订单
    process_order(data)

    # 标记已处理
    mark_order_processed(order_id)
```

### 8.2 快速响应

Webhook 处理应尽快返回响应，复杂业务逻辑应异步处理：

```python
from threading import Thread

@app.route('/webhook/points-mall', methods=['POST'])
def webhook_handler():
    # 验证签名...

    event = request.get_json()

    # 异步处理，立即返回
    Thread(target=process_event, args=(event,)).start()

    return jsonify({"status": "ok"}), 200

def process_event(event):
    # 复杂业务逻辑
    pass
```

### 8.3 日志记录

记录所有 Webhook 请求，便于排查问题：

```python
import logging

logger = logging.getLogger('webhook')

@app.route('/webhook/points-mall', methods=['POST'])
def webhook_handler():
    event_id = request.headers.get('X-Webhook-ID')
    event = request.get_json()

    logger.info(f"Received webhook: {event_id}, type: {event.get('event_type')}")

    try:
        process_event(event)
        logger.info(f"Webhook processed successfully: {event_id}")
        return jsonify({"status": "ok"}), 200
    except Exception as e:
        logger.error(f"Webhook processing failed: {event_id}, error: {str(e)}")
        return jsonify({"error": str(e)}), 500
```

### 8.4 安全建议

1. **始终验证签名** - 防止伪造请求
2. **验证时间戳** - 拒绝过旧的请求（如超过 5 分钟）
3. **使用 HTTPS** - 保护数据传输安全
4. **限制来源 IP** - 可选，仅允许积分商城服务器 IP

```python
# 验证时间戳有效性
import time

def is_timestamp_valid(timestamp: str, tolerance: int = 300) -> bool:
    """检查时间戳是否在有效范围内（默认 5 分钟）"""
    try:
        ts = int(timestamp)
        now = int(time.time())
        return abs(now - ts) <= tolerance
    except ValueError:
        return False
```

---

## 相关文档

- [认证机制](./authentication.md) - API 认证说明
- [错误码参考](./error-codes.md) - 错误处理
- [API 索引](./README.md) - 完整 API 列表
