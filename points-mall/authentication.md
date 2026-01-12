# 认证机制

本文档详细说明积分商城 OpenAPI 的认证机制和签名算法。

---

## 目录

1. [认证概述](#1-认证概述)
2. [认证参数](#2-认证参数)
3. [签名算法](#3-签名算法)
4. [签名示例](#4-签名示例)
5. [常见问题](#5-常见问题)

---

## 1. 认证概述

### 1.1 认证方式

积分商城 OpenAPI 采用 **HMAC-SHA256** 签名认证，确保请求的真实性和完整性。

### 1.2 认证流程

```
┌─────────────────────────────────────────────────────────────┐
│                        客户端                               │
├─────────────────────────────────────────────────────────────┤
│  1. 获取当前 Unix 时间戳（秒级）                            │
│     → timestamp = 1704067200                                │
│                                                             │
│  2. 构造签名字符串                                          │
│     → signString = timestamp + apiKey                       │
│     → "1704067200pk_live_xxx..."                            │
│                                                             │
│  3. 计算 HMAC-SHA256 签名                                   │
│     → signature = HMAC-SHA256(secret, signString)           │
│     → "a1b2c3d4..." (64位小写十六进制)                      │
│                                                             │
│  4. 发送请求，携带认证头                                    │
│     → X-API-Key: pk_live_xxx...                             │
│     → X-Timestamp: 1704067200                               │
│     → X-Signature: a1b2c3d4...                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        服务端                               │
├─────────────────────────────────────────────────────────────┤
│  1. 检查时间戳是否在有效窗口内（±5分钟）                    │
│  2. 验证 API Key 是否有效                                   │
│  3. 使用相同算法重新计算签名                                │
│  4. 比对签名是否一致                                        │
│  5. 返回响应或认证错误                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 认证参数

### 2.1 请求头

所有 API 请求必须携带以下 HTTP Headers：

| Header 名称 | 类型 | 必填 | 说明 |
|-------------|------|------|------|
| `X-API-Key` | string | 是 | 您的 API Key（公开） |
| `X-Timestamp` | string | 是 | 当前 Unix 时间戳（秒级） |
| `X-Signature` | string | 是 | HMAC-SHA256 签名 |

### 2.2 时间戳要求

| 项目 | 说明 |
|------|------|
| 格式 | Unix 时间戳，秒级，10位数字 |
| 有效期 | 服务器时间 ±5 分钟 |
| 用途 | 防止重放攻击 |

**正确格式：**
```
1704067200          ✓ 正确（秒级，10位）
```

**错误格式：**
```
1704067200000       ✗ 错误（毫秒级，13位）
2024-01-01T00:00:00Z ✗ 错误（ISO格式字符串）
```

> **建议**: 确保您的服务器时钟与 NTP 服务器同步

---

## 3. 签名算法

### 3.1 算法说明

采用 **HMAC-SHA256** 算法生成请求签名。

### 3.2 签名步骤

#### Step 1: 构造签名字符串

将时间戳和 API Key **直接拼接**（无分隔符）：

```
SignString = Timestamp + ApiKey
```

**示例：**
```
Timestamp: 1704067200
ApiKey:    pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345

SignString = "1704067200pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345"
```

#### Step 2: 计算 HMAC-SHA256

使用 **Secret** 作为密钥，对签名字符串进行 HMAC-SHA256 计算：

```
Signature = HMAC-SHA256(Secret, SignString)
```

> **注意**: Secret 是私密的，不在请求中传输

#### Step 3: 转换为十六进制

将计算结果转换为 **小写十六进制字符串**（64 字符）：

```
最终签名: a1b2c3d4e5f6... (全小写，64字符)
```

### 3.3 伪代码

```
function generateSignature(apiKey, secret, timestamp):
    signString = timestamp + apiKey          // 拼接
    hmacResult = HMAC-SHA256(secret, signString)  // 计算
    return hexEncode(hmacResult).toLowerCase()    // 转十六进制小写
```

---

## 4. 签名示例

### 4.1 测试数据

```
API Key:   pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345
Secret:    sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH
Timestamp: 1704067200
```

### 4.2 计算过程

```
Step 1 - 签名字符串:
"1704067200pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345"

Step 2 - HMAC-SHA256:
HMAC-SHA256(
    key:  "sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH",
    data: "1704067200pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345"
)

Step 3 - 最终签名:
64位十六进制小写字符串
```

### 4.3 各语言实现

#### Python

```python
import hmac
import hashlib
import time

def generate_signature(api_key: str, secret: str, timestamp: str) -> str:
    """生成 HMAC-SHA256 签名"""
    sign_string = timestamp + api_key
    signature = hmac.new(
        secret.encode('utf-8'),
        sign_string.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    return signature  # 已经是小写十六进制

# 使用示例
api_key = "pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345"
secret = "sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH"
timestamp = str(int(time.time()))

signature = generate_signature(api_key, secret, timestamp)
print(f"Signature: {signature}")
```

#### Java

```java
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;

public class SignatureGenerator {

    public static String generateSignature(String apiKey, String secret, String timestamp)
            throws Exception {
        String signString = timestamp + apiKey;

        Mac mac = Mac.getInstance("HmacSHA256");
        SecretKeySpec secretKeySpec = new SecretKeySpec(
            secret.getBytes(StandardCharsets.UTF_8),
            "HmacSHA256"
        );
        mac.init(secretKeySpec);

        byte[] hash = mac.doFinal(signString.getBytes(StandardCharsets.UTF_8));
        return bytesToHex(hash);
    }

    private static String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder();
        for (byte b : bytes) {
            sb.append(String.format("%02x", b));  // 小写十六进制
        }
        return sb.toString();
    }
}
```

#### Go

```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "fmt"
    "strconv"
    "time"
)

func generateSignature(apiKey, secret, timestamp string) string {
    signString := timestamp + apiKey

    h := hmac.New(sha256.New, []byte(secret))
    h.Write([]byte(signString))

    return hex.EncodeToString(h.Sum(nil))  // 小写十六进制
}

func main() {
    apiKey := "pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345"
    secret := "sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH"
    timestamp := strconv.FormatInt(time.Now().Unix(), 10)

    signature := generateSignature(apiKey, secret, timestamp)
    fmt.Println("Signature:", signature)
}
```

#### JavaScript / Node.js

```javascript
const crypto = require('crypto');

function generateSignature(apiKey, secret, timestamp) {
    const signString = timestamp + apiKey;
    return crypto
        .createHmac('sha256', secret)
        .update(signString)
        .digest('hex');  // 小写十六进制
}

// 使用示例
const apiKey = 'pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345';
const secret = 'sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH';
const timestamp = Math.floor(Date.now() / 1000).toString();

const signature = generateSignature(apiKey, secret, timestamp);
console.log('Signature:', signature);
```

#### cURL / Bash

```bash
#!/bin/bash

API_KEY="pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345"
SECRET="sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH"

# 获取当前时间戳（秒级）
TIMESTAMP=$(date +%s)

# 构造签名字符串
SIGN_STRING="${TIMESTAMP}${API_KEY}"

# 计算 HMAC-SHA256 签名
SIGNATURE=$(echo -n "$SIGN_STRING" | openssl dgst -sha256 -hmac "$SECRET" | cut -d' ' -f2)

echo "Timestamp: $TIMESTAMP"
echo "Signature: $SIGNATURE"
```

---

## 5. 常见问题

### Q1: 签名验证总是失败怎么办？

**排查步骤：**

1. **检查时间戳格式**
   ```
   正确: 1704067200      (秒级，10位数字)
   错误: 1704067200000   (毫秒级，13位)
   错误: "2024-01-01"    (日期字符串)
   ```

2. **检查签名字符串拼接顺序**
   ```
   正确: timestamp + apiKey
   错误: apiKey + timestamp
   ```

3. **检查签名输出格式**
   ```
   正确: a1b2c3d4...     (小写十六进制)
   错误: A1B2C3D4...     (大写十六进制)
   错误: obHD1E...       (Base64)
   ```

4. **检查 Secret 是否正确**
   - Secret 仅在创建时显示一次
   - 确认没有多余的空格或换行符

5. **检查服务器时钟**
   - 确保与 NTP 同步
   - 误差需在 ±5 分钟内

### Q2: 时间戳过期错误如何处理？

错误响应：
```json
{
    "code": 40103,
    "message": "Timestamp expired",
    "reason": "TIMESTAMP_EXPIRED"
}
```

**解决方案：**

1. 检查服务器时间是否正确
2. 使用 NTP 同步时间：`ntpdate pool.ntp.org`
3. 在代码中使用统一的时间源

### Q3: API Key 和 Secret 有什么区别？

| 项目 | API Key | Secret |
|------|---------|--------|
| 用途 | 标识调用方身份 | 生成请求签名 |
| 安全性 | 可在请求中明文传输 | **绝对不能泄露** |
| 格式 | `pk_live_` 开头 | `sk_live_` 开头 |
| 找回 | 可在控制台查看 | 仅创建时显示一次 |

### Q4: 如何验证签名实现是否正确？

使用以下测试向量验证：

```
输入:
  API Key:   pk_live_test1234567890abcdefghij
  Secret:    sk_live_test1234567890abcdefghijklmnopqrstuvwxyz
  Timestamp: 1704067200

签名字符串:
  "1704067200pk_live_test1234567890abcdefghij"

验证方法:
  使用在线 HMAC-SHA256 工具计算，对比结果是否一致
```

---

## 相关文档

- [SDK 示例](./sdk-examples.md) - 完整的各语言客户端实现
- [错误码参考](./error-codes.md) - 认证错误码详解
