# 错误码参考

本文档列出积分商城 OpenAPI 的所有错误码及解决方案。

---

## 目录

1. [错误响应格式](#1-错误响应格式)
2. [认证错误](#2-认证错误)
3. [请求错误](#3-请求错误)
4. [业务错误](#4-业务错误)
5. [系统错误](#5-系统错误)
6. [频率限制](#6-频率限制)

---

## 1. 错误响应格式

当请求失败时，API 返回以下格式的错误响应：

```json
{
    "code": 40001,
    "message": "Invalid API Key",
    "reason": "INVALID_API_KEY"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | int | 错误码（非 0） |
| `message` | string | 错误描述（人类可读） |
| `reason` | string | 错误标识（程序可用） |

---

## 2. 认证错误

HTTP 状态码：**401 Unauthorized**

| 错误码 | reason | 说明 | 解决方案 |
|--------|--------|------|----------|
| 40101 | `MISSING_HEADERS` | 缺少必要的认证头 | 检查是否携带 X-API-Key、X-Timestamp、X-Signature |
| 40102 | `INVALID_API_KEY` | API Key 无效 | 检查 API Key 是否正确、是否过期 |
| 40103 | `TIMESTAMP_EXPIRED` | 时间戳已过期 | 检查服务器时钟是否同步（±5分钟） |
| 40104 | `INVALID_SIGNATURE` | 签名验证失败 | 检查签名算法实现是否正确 |
| 40105 | `API_KEY_DISABLED` | API Key 已禁用 | 联系管理员启用或创建新密钥 |
| 40106 | `API_KEY_EXPIRED` | API Key 已过期 | 更新密钥有效期或创建新密钥 |
| 40107 | `INVALID_TIMESTAMP` | 时间戳格式无效 | 使用 Unix 秒级时间戳（10位数字） |

### 签名验证失败排查

当收到 `INVALID_SIGNATURE` 错误时，按以下步骤排查：

1. **检查签名字符串拼接顺序**
   ```
   正确: timestamp + apiKey
   错误: apiKey + timestamp
   ```

2. **检查时间戳格式**
   ```
   正确: 1704067200      (秒级，10位)
   错误: 1704067200000   (毫秒级，13位)
   ```

3. **检查签名输出格式**
   ```
   正确: a1b2c3d4...     (小写十六进制，64字符)
   错误: A1B2C3D4...     (大写十六进制)
   错误: obHD1OY=...     (Base64 编码)
   ```

4. **检查 Secret 是否正确**
   - 确认没有多余空格或换行符
   - 确认使用的是正确的 Secret

---

## 3. 请求错误

HTTP 状态码：**400 Bad Request**

| 错误码 | reason | 说明 | 解决方案 |
|--------|--------|------|----------|
| 40001 | `INVALID_PARAMS` | 请求参数错误 | 检查参数格式和必填项 |
| 40002 | `MISSING_PARAMS` | 缺少必填参数 | 补充必填参数 |
| 40003 | `INVALID_JSON` | JSON 格式无效 | 检查请求体 JSON 格式 |
| 40004 | `PARAM_OUT_OF_RANGE` | 参数值超出范围 | 检查参数值是否在有效范围内 |
| 40005 | `INVALID_FIELD_VALUE` | 字段值无效 | 检查字段值格式或枚举值 |

### 参数错误示例

```json
{
    "code": 40001,
    "message": "Invalid parameter: page_size must be between 1 and 100",
    "reason": "INVALID_PARAMS"
}
```

---

## 4. 业务错误

HTTP 状态码：**400/404/409**

### 4.1 资源不存在 (404)

| 错误码 | reason | 说明 |
|--------|--------|------|
| 40401 | `PRODUCT_NOT_FOUND` | 商品不存在 |
| 40402 | `ORDER_NOT_FOUND` | 订单不存在 |
| 40403 | `CATEGORY_NOT_FOUND` | 分类不存在 |
| 40404 | `ADDRESS_NOT_FOUND` | 地址不存在 |
| 40405 | `TENANT_NOT_FOUND` | 租户不存在 |
| 40406 | `USER_NOT_FOUND` | 用户不存在 |
| 40407 | `CARD_NOT_FOUND` | 卡密不存在 |

### 4.2 业务冲突 (409)

| 错误码 | reason | 说明 |
|--------|--------|------|
| 40901 | `DUPLICATE_ORDER` | 重复订单 |
| 40902 | `ORDER_STATUS_CONFLICT` | 订单状态冲突（如已完成的订单不能再发货） |
| 40903 | `PRODUCT_OFF_SALE` | 商品已下架 |
| 40904 | `STOCK_NOT_ENOUGH` | 库存不足 |
| 40905 | `POINTS_NOT_ENOUGH` | 积分不足 |
| 40906 | `CARD_ALREADY_SOLD` | 卡密已售出 |
| 40907 | `ADDRESS_LIMIT_EXCEEDED` | 地址数量超限 |

### 4.3 业务限制 (400)

| 错误码 | reason | 说明 |
|--------|--------|------|
| 40010 | `ORDER_LIMIT_EXCEEDED` | 订单数量超限 |
| 40011 | `PRODUCT_LIMIT_EXCEEDED` | 商品购买数量超限 |
| 40012 | `INVALID_ORDER_STATUS` | 无效的订单状态 |
| 40013 | `INVALID_PRODUCT_TYPE` | 无效的商品类型 |
| 40014 | `SHIPPING_ADDRESS_REQUIRED` | 实物商品需要收货地址 |

---

## 5. 系统错误

HTTP 状态码：**500 Internal Server Error**

| 错误码 | reason | 说明 | 解决方案 |
|--------|--------|------|----------|
| 50001 | `INTERNAL_ERROR` | 服务器内部错误 | 稍后重试，如持续失败请联系技术支持 |
| 50002 | `SERVICE_UNAVAILABLE` | 服务不可用 | 稍后重试 |
| 50003 | `DATABASE_ERROR` | 数据库错误 | 稍后重试，如持续失败请联系技术支持 |
| 50004 | `THIRD_PARTY_ERROR` | 第三方服务错误 | 稍后重试 |

---

## 6. 频率限制

HTTP 状态码：**429 Too Many Requests**

| 错误码 | reason | 说明 |
|--------|--------|------|
| 42901 | `RATE_LIMIT_EXCEEDED` | 请求频率超限 |

### 6.1 限制规则

| 限制类型 | 限制值 | 说明 |
|----------|--------|------|
| 每秒请求数 (QPS) | 100 | 单个 API Key |
| 每分钟请求数 | 3000 | 单个 API Key |
| 每日请求数 | 100,000 | 单个 API Key |

### 6.2 响应头

超出限制时，响应头包含限制信息：

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1704067260
```

| Header | 说明 |
|--------|------|
| `X-RateLimit-Limit` | 请求限制数 |
| `X-RateLimit-Remaining` | 剩余请求数 |
| `X-RateLimit-Reset` | 限制重置的 Unix 时间戳 |

### 6.3 处理建议

```python
import time

def request_with_retry(func, max_retries=3):
    """带重试的请求"""
    for i in range(max_retries):
        try:
            response = func()
            if response.status_code == 429:
                # 获取重置时间
                reset_time = int(response.headers.get('X-RateLimit-Reset', 0))
                wait_time = max(reset_time - time.time(), 1)
                time.sleep(wait_time)
                continue
            return response
        except Exception as e:
            if i == max_retries - 1:
                raise
            # 指数退避
            time.sleep((2 ** i) + random.random())
    raise Exception("Max retries exceeded")
```

---

## 错误码速查表

| 范围 | 类别 | HTTP 状态码 |
|------|------|-------------|
| 40101-40107 | 认证错误 | 401 |
| 40001-40005 | 请求参数错误 | 400 |
| 40010-40014 | 业务限制错误 | 400 |
| 40401-40407 | 资源不存在 | 404 |
| 40901-40907 | 业务冲突 | 409 |
| 42901 | 频率限制 | 429 |
| 50001-50004 | 系统错误 | 500 |

---

## 相关文档

- [认证机制](./authentication.md) - 解决认证错误
- [接入指南](./integration-guide.md) - 请求规范说明
