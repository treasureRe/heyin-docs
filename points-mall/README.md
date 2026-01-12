# 积分商城 OpenAPI 文档

> 版本: v1.0.0

欢迎使用积分商城 OpenAPI。本文档提供完整的 API 接入指南。

---

## 快速链接

| 文档 | 说明 |
|------|------|
| [接入指南](./integration-guide.md) | 快速开始、接入流程 |
| [认证机制](./authentication.md) | HMAC-SHA256 签名算法详解 |
| [SDK 示例](./sdk-examples.md) | Python、Java、Go、JavaScript、cURL 示例代码 |
| [错误码参考](./error-codes.md) | 完整错误码列表及解决方案 |
| [Webhook 指南](./webhook-guide.md) | 异步通知对接说明 |
| [OpenAPI 规范](../openapi/openapi.external.yaml) | Swagger/OpenAPI 3.0 规范文件 |

---

## 环境信息

| 环境 | Base URL | 说明 |
|------|----------|------|
| 沙箱环境 | `http://43.133.43.2/openapi/points` | 测试环境，用于开发调试 |
| 生产环境 | 待定 | 正式环境，数据真实有效 |

---

## API 分组概览

### 1. 商城服务 (Mall)

面向 H5 商城用户的接口。

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取商品列表 | GET | `/openapi/points/mall/products` | 支持分页、分类筛选 |
| 获取商品详情 | GET | `/openapi/points/mall/products/{productId}` | 含多语言信息 |
| 获取分类列表 | GET | `/openapi/points/mall/categories` | 商品分类树 |
| 获取用户信息 | GET | `/openapi/points/mall/user/info` | 用户积分等信息 |
| 创建订单 | POST | `/openapi/points/mall/orders` | 积分兑换下单 |
| 获取订单列表 | GET | `/openapi/points/mall/orders` | 用户订单记录 |
| 获取订单详情 | GET | `/openapi/points/mall/orders/{orderId}` | 含物流、卡密信息 |
| 确认收货 | PUT | `/openapi/points/mall/orders/{orderId}/confirm` | 完成订单 |

**收货地址管理：**

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取地址列表 | GET | `/openapi/points/mall/addresses` | 用户收货地址 |
| 创建地址 | POST | `/openapi/points/mall/addresses` | 新增收货地址 |
| 更新地址 | PUT | `/openapi/points/mall/addresses/{id}` | 修改地址信息 |
| 删除地址 | DELETE | `/openapi/points/mall/addresses/{id}` | 删除地址 |
| 获取默认地址 | GET | `/openapi/points/mall/addresses/default` | 默认收货地址 |
| 设置默认地址 | PUT | `/openapi/points/mall/addresses/{id}/default` | 设为默认 |

### 2. 商品管理 (Goods Management)

后台管理接口，用于商品维护。

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取商品列表 | GET | `/openapi/points/goods` | 管理后台商品列表 |
| 创建商品 | POST | `/openapi/points/goods` | 新增商品 |
| 获取商品详情 | GET | `/openapi/points/goods/{id}` | 商品完整信息 |
| 更新商品 | PUT | `/openapi/points/goods/{id}` | 修改商品信息 |
| 删除商品 | DELETE | `/openapi/points/goods/{id}` | 删除商品 |

**虚拟卡密管理：**

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取卡密列表 | GET | `/openapi/points/goods/cards/{goodsId}` | 商品关联的卡密 |
| 批量上传卡密 | POST | `/openapi/points/good/cards/batch/{goodsId}` | 手动输入批量添加 |
| Excel导入卡密 | POST | `/openapi/points/goods/cards/import/{goodsId}` | Excel文件导入 |
| 删除卡密 | DELETE | `/openapi/points/cards/{cardId}` | 删除单个卡密 |
| 更新卡密状态 | PUT | `/openapi/points/cards/{cardId}/status` | 设置可用/无效 |

### 3. 订单管理 (Order Management)

后台管理接口，用于订单处理。

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取订单列表 | GET | `/openapi/points/orders` | 支持多条件筛选 |
| 获取订单详情 | GET | `/openapi/points/orders/{id}` | 订单完整信息 |
| 更新订单 | PUT | `/openapi/points/orders/{id}` | 修改订单信息 |
| 删除订单 | DELETE | `/openapi/points/orders/{id}` | 删除订单记录 |
| 发货 | POST | `/openapi/points/orders/{id}/ship` | 填写物流信息 |

### 4. 商品分类 (Product Categories)

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取分类列表 | GET | `/openapi/points/product-categories` | 分类列表 |
| 创建分类 | POST | `/openapi/points/product-categories` | 新增分类 |
| 获取分类详情 | GET | `/openapi/points/product-categories/{id}` | 分类信息 |
| 更新分类 | PUT | `/openapi/points/product-categories/{id}` | 修改分类 |
| 删除分类 | DELETE | `/openapi/points/product-categories/{id}` | 删除分类 |

### 5. 物流管理 (Logistics)

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取物流公司列表 | GET | `/openapi/points/logistics-companies` | 可选物流公司 |
| 创建物流公司 | POST | `/openapi/points/logistics-companies` | 新增物流公司 |
| 获取物流公司详情 | GET | `/openapi/points/logistics-companies/{id}` | 物流公司信息 |
| 更新物流公司 | PUT | `/openapi/points/logistics-companies/{id}` | 修改物流公司 |
| 删除物流公司 | DELETE | `/openapi/points/logistics-companies/{id}` | 删除物流公司 |

### 6. 租户管理 (Tenant Management)

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取租户列表 | GET | `/openapi/points/tenants` | 租户配置列表 |
| 创建租户 | POST | `/openapi/points/tenants` | 新增租户配置 |
| 获取租户详情 | GET | `/openapi/points/tenants/{tenantCode}` | 租户完整配置 |
| 更新租户 | PUT | `/openapi/points/tenants/{tenantCode}` | 修改租户配置 |
| 删除租户 | DELETE | `/openapi/points/tenants/{tenantCode}` | 删除租户 |
| 重置密钥 | POST | `/openapi/points/tenants/{tenantCode}/reset-secret` | 重新生成API密钥 |

### 7. 认证服务 (Auth Ticket)

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 生成票据 | POST | `/openapi/points/auth/ticket/generate` | 生成免登票据 |
| 验证票据 | GET | `/openapi/points/auth/ticket/verify` | 验证并建立会话 |
| 刷新会话 | POST | `/openapi/points/auth/session/refresh` | 续期会话Token |

### 8. Webhook 服务

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 确认Webhook | POST | `/openapi/points/webhook/confirm` | 商户确认处理结果 |
| 获取事件列表 | GET | `/openapi/points/webhook/events` | 查询Webhook事件 |
| 获取事件详情 | GET | `/openapi/points/webhook/events/{eventId}` | 单个事件信息 |
| 手动重试 | POST | `/openapi/points/webhook/events/{eventId}/retry` | 重新发送Webhook |

---

## 通用规范

### 请求格式

- **协议**: HTTP/HTTPS
- **数据格式**: JSON
- **字符编码**: UTF-8
- **字段命名**: snake_case（下划线命名法）

### 认证方式

所有 API 请求必须携带以下 HTTP Headers：

```
X-API-Key: {your_api_key}
X-Timestamp: {unix_timestamp}
X-Signature: {hmac_sha256_signature}
```

详见 [认证机制](./authentication.md)。

### 响应格式

**成功响应：**
```json
{
    "code": 0,
    "message": "success",
    "data": { ... }
}
```

**错误响应：**
```json
{
    "code": 40001,
    "message": "Invalid API Key",
    "reason": "INVALID_API_KEY"
}
```

---

## 使用 Swagger UI 查看接口

1. 打开 [Swagger Editor](https://editor.swagger.io/)
2. 点击 `File` -> `Import URL`
3. 导入 `openapi.external.yaml` 文件
4. 即可查看完整的接口文档和请求/响应示例

---

## 使用 Postman 测试

1. 下载 [Postman Collection](../openapi/postman/points-mall.postman_collection.json)
2. 在 Postman 中导入 Collection
3. 配置环境变量：
   - `baseUrl`: `http://43.133.43.2/openapi/points`
   - `apiKey`: 您的 API Key
   - `apiSecret`: 您的 Secret
4. 手动添加认证头（或配置 Pre-request Script）

---

## 联系方式

如有问题，请联系技术支持。
