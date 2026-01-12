# 积分商城 OpenAPI 接入指南

> 版本: v1.0.0

本文档帮助您快速接入积分商城 OpenAPI。

---

## 目录

1. [概述](#1-概述)
2. [接入准备](#2-接入准备)
3. [快速开始](#3-快速开始)
4. [请求规范](#4-请求规范)
5. [响应格式](#5-响应格式)
6. [典型接入场景](#6-典型接入场景)

---

## 1. 概述

### 1.1 产品介绍

积分商城 OpenAPI 为企业提供积分兑换、商品管理、订单处理等能力的开放接口，支持快速对接企业现有系统。

### 1.2 接口基础信息

| 环境 | Base URL | 说明 |
|------|----------|------|
| 沙箱环境 | `http://43.133.43.2/openapi/points` | 测试环境，用于开发调试 |
| 生产环境 | 待定 | 正式环境，数据真实有效 |

### 1.3 通信协议

| 项目 | 说明 |
|------|------|
| 协议 | HTTP（沙箱）/ HTTPS（生产） |
| 请求方法 | GET / POST / PUT / DELETE |
| 数据格式 | JSON |
| 字符编码 | UTF-8 |

---

## 2. 接入准备

### 2.1 获取 API 密钥

联系管理员获取 API 密钥对：

| 密钥类型 | 格式示例 | 用途 |
|----------|----------|------|
| **API Key** | `pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345` | 公开标识，用于识别调用方 |
| **Secret** | `sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH` | 私密密钥，用于生成签名 |

> **重要提示**
> - Secret 仅在创建时显示一次，请务必安全保存
> - 切勿将 Secret 暴露在前端代码、日志或版本控制中

### 2.2 密钥格式说明

```
API Key 格式: pk_live_ + 32位随机字符 (URL-safe Base64)
Secret 格式:  sk_live_ + 48位随机字符 (URL-safe Base64)

示例:
API Key: pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345
Secret:  sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH
```

---

## 3. 快速开始

### 3.1 第一个请求

以下示例展示如何获取商品列表：

**请求：**
```http
GET /openapi/points/mall/products?page=1&page_size=10 HTTP/1.1
Host: 43.133.43.2
Content-Type: application/json
X-API-Key: pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345
X-Timestamp: 1704067200
X-Signature: a1b2c3d4e5f67890abcdef1234567890abcdef1234567890abcdef1234567890
```

**响应：**
```json
{
    "code": 0,
    "message": "success",
    "data": {
        "items": [
            {
                "id": "1",
                "name": "积分兑换商品A",
                "price": 100,
                "stock": 50,
                "status": "ON_SALE"
            }
        ],
        "total": 1,
        "page": 1,
        "page_size": 10
    }
}
```

### 3.2 认证头说明

每个请求必须携带以下 3 个 HTTP Headers：

| Header 名称 | 类型 | 必填 | 说明 |
|-------------|------|------|------|
| `X-API-Key` | string | 是 | 您的 API Key |
| `X-Timestamp` | string | 是 | 当前 Unix 时间戳（秒级，10位数字） |
| `X-Signature` | string | 是 | HMAC-SHA256 签名（64位十六进制小写） |

签名算法详见 [认证机制](./authentication.md)。

### 3.3 cURL 快速测试

```bash
#!/bin/bash

API_KEY="pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345"
SECRET="sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH"
BASE_URL="http://43.133.43.2/openapi/points"

# 获取当前时间戳
TIMESTAMP=$(date +%s)

# 生成签名: HMAC-SHA256(secret, timestamp + apiKey)
SIGN_STRING="${TIMESTAMP}${API_KEY}"
SIGNATURE=$(echo -n "$SIGN_STRING" | openssl dgst -sha256 -hmac "$SECRET" | cut -d' ' -f2)

# 发送请求
curl -X GET "${BASE_URL}/mall/products?page=1&page_size=10" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${API_KEY}" \
  -H "X-Timestamp: ${TIMESTAMP}" \
  -H "X-Signature: ${SIGNATURE}"
```

---

## 4. 请求规范

### 4.1 请求头

| Header | 必填 | 说明 |
|--------|------|------|
| `Content-Type` | 是 | 固定值 `application/json` |
| `X-API-Key` | 是 | API Key |
| `X-Timestamp` | 是 | Unix 时间戳（秒） |
| `X-Signature` | 是 | 请求签名 |
| `X-Request-Id` | 否 | 请求唯一标识（用于追踪） |

### 4.2 请求体规范

- 使用 JSON 格式
- 字符编码 UTF-8
- 字段命名采用 **snake_case**（下划线命名法）
- 空值字段可以省略

**示例：**
```json
{
    "product_id": "PROD_001",
    "quantity": 1,
    "user_id": "U123456",
    "address_id": "ADDR_001"
}
```

### 4.3 分页参数

列表接口通常支持以下分页参数：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | int | 1 | 页码，从 1 开始 |
| `page_size` | int | 10 | 每页数量，最大 100 |

---

## 5. 响应格式

### 5.1 成功响应

```json
{
    "code": 0,
    "message": "success",
    "data": {
        "order_id": "ORD_20240101_001",
        "status": "PENDING",
        "created_at": "2024-01-01T12:00:00Z"
    }
}
```

### 5.2 错误响应

```json
{
    "code": 40001,
    "message": "Invalid API Key",
    "reason": "INVALID_API_KEY"
}
```

### 5.3 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | int | 状态码，0 表示成功，非 0 表示失败 |
| `message` | string | 状态描述信息 |
| `data` | object | 响应数据（成功时返回） |
| `reason` | string | 错误码标识（失败时返回） |

### 5.4 分页响应

列表接口的响应格式：

```json
{
    "code": 0,
    "message": "success",
    "data": {
        "items": [...],
        "total": 100,
        "page": 1,
        "page_size": 10
    }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `items` | array | 数据列表 |
| `total` | int | 总记录数 |
| `page` | int | 当前页码 |
| `page_size` | int | 每页数量 |

---

## 6. 典型接入场景

### 6.1 H5 商城接入

适用于企业 H5 页面嵌入积分商城功能。

**接入流程：**

```
1. 服务端生成票据
   POST /openapi/points/auth/ticket/generate

2. H5 页面使用票据验证
   GET /openapi/points/auth/ticket/verify?ticket=xxx

3. 获取会话 Token 后调用商城接口
   GET /openapi/points/mall/products

4. 用户下单
   POST /openapi/points/mall/orders
```

### 6.2 后台管理对接

适用于企业后台系统管理商品和订单。

**典型操作：**

```
1. 商品管理
   GET/POST/PUT/DELETE /openapi/points/goods/*

2. 订单管理
   GET /openapi/points/orders
   POST /openapi/points/orders/{id}/ship

3. 卡密管理（虚拟商品）
   POST /openapi/points/goods/cards/import/{goodsId}
```

### 6.3 Webhook 事件处理

接收订单状态变更等异步通知。

**配置流程：**

1. 在租户配置中设置 Webhook URL
2. 实现 Webhook 接收端点
3. 验证 Webhook 签名
4. 处理事件并返回确认

详见 [Webhook 指南](./webhook-guide.md)。

---

## 下一步

- [认证机制](./authentication.md) - 了解签名算法详情
- [SDK 示例](./sdk-examples.md) - 获取各语言的示例代码
- [错误码参考](./error-codes.md) - 查看完整错误码列表
