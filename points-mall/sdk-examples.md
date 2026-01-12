# SDK 示例

本文档提供多种编程语言的完整 SDK 实现示例。

---

## 目录

1. [Python](#1-python)
2. [Java](#2-java)
3. [Go](#3-go)
4. [JavaScript / Node.js](#4-javascript--nodejs)
5. [cURL / Bash](#5-curl--bash)

---

## 1. Python

### 完整客户端实现

```python
import hmac
import hashlib
import time
import requests
from typing import Optional, Dict, Any


class PointsMallClient:
    """积分商城 OpenAPI 客户端"""

    def __init__(
        self,
        api_key: str,
        secret: str,
        base_url: str = "http://43.133.43.2/openapi/points"
    ):
        self.api_key = api_key
        self.secret = secret
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()

    def _generate_signature(self, timestamp: str) -> str:
        """生成 HMAC-SHA256 签名"""
        sign_string = timestamp + self.api_key
        signature = hmac.new(
            self.secret.encode('utf-8'),
            sign_string.encode('utf-8'),
            hashlib.sha256
        ).hexdigest()
        return signature

    def _get_headers(self) -> Dict[str, str]:
        """构造请求头"""
        timestamp = str(int(time.time()))
        return {
            "Content-Type": "application/json",
            "X-API-Key": self.api_key,
            "X-Timestamp": timestamp,
            "X-Signature": self._generate_signature(timestamp)
        }

    def _request(
        self,
        method: str,
        path: str,
        params: Optional[Dict] = None,
        json: Optional[Dict] = None
    ) -> Dict[str, Any]:
        """发送请求"""
        url = f"{self.base_url}{path}"
        response = self.session.request(
            method=method,
            url=url,
            params=params,
            json=json,
            headers=self._get_headers()
        )
        return response.json()

    # ==================== 商城接口 ====================

    def get_products(
        self,
        page: int = 1,
        page_size: int = 10,
        category_id: Optional[str] = None,
        lang: str = "zh"
    ) -> Dict[str, Any]:
        """获取商品列表"""
        params = {
            "page": page,
            "page_size": page_size,
            "lang": lang
        }
        if category_id:
            params["category_id"] = category_id
        return self._request("GET", "/mall/products", params=params)

    def get_product(self, product_id: str, lang: str = "zh") -> Dict[str, Any]:
        """获取商品详情"""
        return self._request(
            "GET",
            f"/mall/products/{product_id}",
            params={"lang": lang}
        )

    def get_categories(self) -> Dict[str, Any]:
        """获取商品分类列表"""
        return self._request("GET", "/mall/categories")

    def get_user_info(self) -> Dict[str, Any]:
        """获取用户信息"""
        return self._request("GET", "/mall/user/info")

    def create_order(
        self,
        product_id: str,
        quantity: int,
        address_id: Optional[str] = None
    ) -> Dict[str, Any]:
        """创建订单"""
        payload = {
            "product_id": product_id,
            "quantity": quantity
        }
        if address_id:
            payload["address_id"] = address_id
        return self._request("POST", "/mall/orders", json=payload)

    def get_orders(
        self,
        page: int = 1,
        page_size: int = 10,
        status: Optional[str] = None
    ) -> Dict[str, Any]:
        """获取订单列表"""
        params = {"page": page, "page_size": page_size}
        if status:
            params["status"] = status
        return self._request("GET", "/mall/orders", params=params)

    def get_order(self, order_id: str) -> Dict[str, Any]:
        """获取订单详情"""
        return self._request("GET", f"/mall/orders/{order_id}")

    def confirm_order(self, order_id: str) -> Dict[str, Any]:
        """确认收货"""
        return self._request("PUT", f"/mall/orders/{order_id}/confirm")

    # ==================== 地址管理 ====================

    def get_addresses(self) -> Dict[str, Any]:
        """获取收货地址列表"""
        return self._request("GET", "/mall/addresses")

    def create_address(
        self,
        name: str,
        phone: str,
        province: str,
        city: str,
        district: str,
        address: str,
        is_default: bool = False
    ) -> Dict[str, Any]:
        """创建收货地址"""
        payload = {
            "name": name,
            "phone": phone,
            "province": province,
            "city": city,
            "district": district,
            "address": address,
            "is_default": is_default
        }
        return self._request("POST", "/mall/addresses", json=payload)


# ==================== 使用示例 ====================

if __name__ == "__main__":
    # 初始化客户端
    client = PointsMallClient(
        api_key="pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345",
        secret="sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH"
    )

    # 获取商品列表
    result = client.get_products(page=1, page_size=10)
    print("商品列表:", result)

    # 获取商品详情
    # result = client.get_product("PROD_001")
    # print("商品详情:", result)

    # 创建订单
    # result = client.create_order(
    #     product_id="PROD_001",
    #     quantity=1,
    #     address_id="ADDR_001"
    # )
    # print("订单创建结果:", result)
```

---

## 2. Java

### 完整客户端实现

```java
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;

public class PointsMallClient {

    private final String apiKey;
    private final String secret;
    private final String baseUrl;
    private final HttpClient httpClient;

    public PointsMallClient(String apiKey, String secret) {
        this(apiKey, secret, "http://43.133.43.2/openapi/points");
    }

    public PointsMallClient(String apiKey, String secret, String baseUrl) {
        this.apiKey = apiKey;
        this.secret = secret;
        this.baseUrl = baseUrl;
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(30))
            .build();
    }

    /**
     * 生成 HMAC-SHA256 签名
     */
    private String generateSignature(String timestamp) throws Exception {
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
            sb.append(String.format("%02x", b));
        }
        return sb.toString();
    }

    /**
     * 发送 GET 请求
     */
    public String get(String path) throws Exception {
        return get(path, null);
    }

    public String get(String path, Map<String, String> params) throws Exception {
        String url = baseUrl + path;
        if (params != null && !params.isEmpty()) {
            StringBuilder queryString = new StringBuilder("?");
            params.forEach((k, v) -> queryString.append(k).append("=").append(v).append("&"));
            url += queryString.substring(0, queryString.length() - 1);
        }

        String timestamp = String.valueOf(Instant.now().getEpochSecond());
        String signature = generateSignature(timestamp);

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .header("Content-Type", "application/json")
            .header("X-API-Key", apiKey)
            .header("X-Timestamp", timestamp)
            .header("X-Signature", signature)
            .GET()
            .build();

        HttpResponse<String> response = httpClient.send(
            request,
            HttpResponse.BodyHandlers.ofString()
        );
        return response.body();
    }

    /**
     * 发送 POST 请求
     */
    public String post(String path, String jsonBody) throws Exception {
        String timestamp = String.valueOf(Instant.now().getEpochSecond());
        String signature = generateSignature(timestamp);

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + path))
            .header("Content-Type", "application/json")
            .header("X-API-Key", apiKey)
            .header("X-Timestamp", timestamp)
            .header("X-Signature", signature)
            .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
            .build();

        HttpResponse<String> response = httpClient.send(
            request,
            HttpResponse.BodyHandlers.ofString()
        );
        return response.body();
    }

    /**
     * 发送 PUT 请求
     */
    public String put(String path, String jsonBody) throws Exception {
        String timestamp = String.valueOf(Instant.now().getEpochSecond());
        String signature = generateSignature(timestamp);

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + path))
            .header("Content-Type", "application/json")
            .header("X-API-Key", apiKey)
            .header("X-Timestamp", timestamp)
            .header("X-Signature", signature)
            .PUT(HttpRequest.BodyPublishers.ofString(jsonBody != null ? jsonBody : ""))
            .build();

        HttpResponse<String> response = httpClient.send(
            request,
            HttpResponse.BodyHandlers.ofString()
        );
        return response.body();
    }

    // ==================== 使用示例 ====================

    public static void main(String[] args) throws Exception {
        PointsMallClient client = new PointsMallClient(
            "pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345",
            "sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH"
        );

        // 获取商品列表
        String result = client.get("/mall/products", Map.of(
            "page", "1",
            "page_size", "10"
        ));
        System.out.println("商品列表: " + result);

        // 创建订单
        // String orderResult = client.post("/mall/orders",
        //     "{\"product_id\":\"PROD_001\",\"quantity\":1}");
        // System.out.println("订单结果: " + orderResult);
    }
}
```

---

## 3. Go

### 完整客户端实现

```go
package main

import (
	"bytes"
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"strconv"
	"time"
)

// PointsMallClient 积分商城客户端
type PointsMallClient struct {
	apiKey  string
	secret  string
	baseURL string
	client  *http.Client
}

// NewPointsMallClient 创建客户端实例
func NewPointsMallClient(apiKey, secret string) *PointsMallClient {
	return &PointsMallClient{
		apiKey:  apiKey,
		secret:  secret,
		baseURL: "http://43.133.43.2/openapi/points",
		client:  &http.Client{Timeout: 30 * time.Second},
	}
}

// GenerateSignature 生成 HMAC-SHA256 签名
func (c *PointsMallClient) GenerateSignature(timestamp string) string {
	signString := timestamp + c.apiKey
	h := hmac.New(sha256.New, []byte(c.secret))
	h.Write([]byte(signString))
	return hex.EncodeToString(h.Sum(nil))
}

// request 发送 HTTP 请求
func (c *PointsMallClient) request(method, path string, params url.Values, body interface{}) (map[string]interface{}, error) {
	// 构建 URL
	reqURL := c.baseURL + path
	if params != nil && len(params) > 0 {
		reqURL += "?" + params.Encode()
	}

	// 构建请求体
	var bodyReader io.Reader
	if body != nil {
		jsonBody, err := json.Marshal(body)
		if err != nil {
			return nil, err
		}
		bodyReader = bytes.NewBuffer(jsonBody)
	}

	// 创建请求
	req, err := http.NewRequest(method, reqURL, bodyReader)
	if err != nil {
		return nil, err
	}

	// 设置认证头
	timestamp := strconv.FormatInt(time.Now().Unix(), 10)
	signature := c.GenerateSignature(timestamp)

	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-API-Key", c.apiKey)
	req.Header.Set("X-Timestamp", timestamp)
	req.Header.Set("X-Signature", signature)

	// 发送请求
	resp, err := c.client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	// 解析响应
	respBody, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, err
	}

	var result map[string]interface{}
	if err := json.Unmarshal(respBody, &result); err != nil {
		return nil, err
	}

	return result, nil
}

// GetProducts 获取商品列表
func (c *PointsMallClient) GetProducts(page, pageSize int) (map[string]interface{}, error) {
	params := url.Values{
		"page":      {strconv.Itoa(page)},
		"page_size": {strconv.Itoa(pageSize)},
	}
	return c.request("GET", "/mall/products", params, nil)
}

// GetProduct 获取商品详情
func (c *PointsMallClient) GetProduct(productID string) (map[string]interface{}, error) {
	return c.request("GET", "/mall/products/"+productID, nil, nil)
}

// CreateOrder 创建订单
func (c *PointsMallClient) CreateOrder(productID string, quantity int, addressID string) (map[string]interface{}, error) {
	body := map[string]interface{}{
		"product_id": productID,
		"quantity":   quantity,
	}
	if addressID != "" {
		body["address_id"] = addressID
	}
	return c.request("POST", "/mall/orders", nil, body)
}

// GetOrders 获取订单列表
func (c *PointsMallClient) GetOrders(page, pageSize int) (map[string]interface{}, error) {
	params := url.Values{
		"page":      {strconv.Itoa(page)},
		"page_size": {strconv.Itoa(pageSize)},
	}
	return c.request("GET", "/mall/orders", params, nil)
}

// GetOrder 获取订单详情
func (c *PointsMallClient) GetOrder(orderID string) (map[string]interface{}, error) {
	return c.request("GET", "/mall/orders/"+orderID, nil, nil)
}

// ConfirmOrder 确认收货
func (c *PointsMallClient) ConfirmOrder(orderID string) (map[string]interface{}, error) {
	return c.request("PUT", "/mall/orders/"+orderID+"/confirm", nil, nil)
}

// ==================== 使用示例 ====================

func main() {
	client := NewPointsMallClient(
		"pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345",
		"sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH",
	)

	// 获取商品列表
	result, err := client.GetProducts(1, 10)
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}
	fmt.Printf("商品列表: %+v\n", result)

	// 创建订单
	// result, err = client.CreateOrder("PROD_001", 1, "ADDR_001")
	// if err != nil {
	// 	fmt.Printf("Error: %v\n", err)
	// 	return
	// }
	// fmt.Printf("订单结果: %+v\n", result)
}
```

---

## 4. JavaScript / Node.js

### 完整客户端实现

```javascript
const crypto = require('crypto');
const axios = require('axios');

class PointsMallClient {
    /**
     * @param {string} apiKey - API Key
     * @param {string} secret - Secret
     * @param {string} baseUrl - Base URL
     */
    constructor(apiKey, secret, baseUrl = 'http://43.133.43.2/openapi/points') {
        this.apiKey = apiKey;
        this.secret = secret;
        this.baseUrl = baseUrl.replace(/\/$/, '');
        this.client = axios.create({
            baseURL: this.baseUrl,
            timeout: 30000
        });
    }

    /**
     * 生成 HMAC-SHA256 签名
     * @param {string} timestamp - Unix 时间戳
     * @returns {string} 签名
     */
    generateSignature(timestamp) {
        const signString = timestamp + this.apiKey;
        return crypto
            .createHmac('sha256', this.secret)
            .update(signString)
            .digest('hex');
    }

    /**
     * 获取请求头
     * @returns {Object} 请求头
     */
    getHeaders() {
        const timestamp = Math.floor(Date.now() / 1000).toString();
        return {
            'Content-Type': 'application/json',
            'X-API-Key': this.apiKey,
            'X-Timestamp': timestamp,
            'X-Signature': this.generateSignature(timestamp)
        };
    }

    /**
     * 发送请求
     * @param {string} method - HTTP 方法
     * @param {string} path - 路径
     * @param {Object} params - 查询参数
     * @param {Object} data - 请求体
     * @returns {Promise<Object>} 响应数据
     */
    async request(method, path, params = null, data = null) {
        const response = await this.client.request({
            method,
            url: path,
            params,
            data,
            headers: this.getHeaders()
        });
        return response.data;
    }

    // ==================== 商城接口 ====================

    /**
     * 获取商品列表
     */
    async getProducts(page = 1, pageSize = 10, categoryId = null) {
        const params = { page, page_size: pageSize };
        if (categoryId) params.category_id = categoryId;
        return this.request('GET', '/mall/products', params);
    }

    /**
     * 获取商品详情
     */
    async getProduct(productId) {
        return this.request('GET', `/mall/products/${productId}`);
    }

    /**
     * 获取商品分类
     */
    async getCategories() {
        return this.request('GET', '/mall/categories');
    }

    /**
     * 获取用户信息
     */
    async getUserInfo() {
        return this.request('GET', '/mall/user/info');
    }

    /**
     * 创建订单
     */
    async createOrder(productId, quantity, addressId = null) {
        const data = { product_id: productId, quantity };
        if (addressId) data.address_id = addressId;
        return this.request('POST', '/mall/orders', null, data);
    }

    /**
     * 获取订单列表
     */
    async getOrders(page = 1, pageSize = 10, status = null) {
        const params = { page, page_size: pageSize };
        if (status) params.status = status;
        return this.request('GET', '/mall/orders', params);
    }

    /**
     * 获取订单详情
     */
    async getOrder(orderId) {
        return this.request('GET', `/mall/orders/${orderId}`);
    }

    /**
     * 确认收货
     */
    async confirmOrder(orderId) {
        return this.request('PUT', `/mall/orders/${orderId}/confirm`);
    }

    // ==================== 地址管理 ====================

    /**
     * 获取地址列表
     */
    async getAddresses() {
        return this.request('GET', '/mall/addresses');
    }

    /**
     * 创建地址
     */
    async createAddress(addressData) {
        return this.request('POST', '/mall/addresses', null, addressData);
    }
}

// ==================== 使用示例 ====================

async function main() {
    const client = new PointsMallClient(
        'pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345',
        'sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH'
    );

    try {
        // 获取商品列表
        const products = await client.getProducts(1, 10);
        console.log('商品列表:', products);

        // 创建订单
        // const order = await client.createOrder('PROD_001', 1, 'ADDR_001');
        // console.log('订单结果:', order);
    } catch (error) {
        console.error('Error:', error.response?.data || error.message);
    }
}

main();

module.exports = PointsMallClient;
```

---

## 5. cURL / Bash

### 通用脚本

```bash
#!/bin/bash

# ==================== 配置 ====================

API_KEY="pk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ012345"
SECRET="sk_live_aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEFGH"
BASE_URL="http://43.133.43.2/openapi/points"

# ==================== 签名函数 ====================

generate_signature() {
    local timestamp=$1
    local sign_string="${timestamp}${API_KEY}"
    echo -n "$sign_string" | openssl dgst -sha256 -hmac "$SECRET" | cut -d' ' -f2
}

# ==================== 请求函数 ====================

api_get() {
    local path=$1
    local timestamp=$(date +%s)
    local signature=$(generate_signature $timestamp)

    curl -s -X GET "${BASE_URL}${path}" \
        -H "Content-Type: application/json" \
        -H "X-API-Key: ${API_KEY}" \
        -H "X-Timestamp: ${timestamp}" \
        -H "X-Signature: ${signature}"
}

api_post() {
    local path=$1
    local data=$2
    local timestamp=$(date +%s)
    local signature=$(generate_signature $timestamp)

    curl -s -X POST "${BASE_URL}${path}" \
        -H "Content-Type: application/json" \
        -H "X-API-Key: ${API_KEY}" \
        -H "X-Timestamp: ${timestamp}" \
        -H "X-Signature: ${signature}" \
        -d "$data"
}

api_put() {
    local path=$1
    local data=$2
    local timestamp=$(date +%s)
    local signature=$(generate_signature $timestamp)

    curl -s -X PUT "${BASE_URL}${path}" \
        -H "Content-Type: application/json" \
        -H "X-API-Key: ${API_KEY}" \
        -H "X-Timestamp: ${timestamp}" \
        -H "X-Signature: ${signature}" \
        -d "$data"
}

# ==================== 使用示例 ====================

# 获取商品列表
echo "获取商品列表:"
api_get "/mall/products?page=1&page_size=10"
echo ""

# 获取商品详情
# echo "获取商品详情:"
# api_get "/mall/products/PROD_001"
# echo ""

# 创建订单
# echo "创建订单:"
# api_post "/mall/orders" '{"product_id":"PROD_001","quantity":1}'
# echo ""

# 确认收货
# echo "确认收货:"
# api_put "/mall/orders/ORDER_001/confirm" '{}'
# echo ""
```

### 单次请求示例

```bash
# 获取商品列表
API_KEY="pk_live_xxx"
SECRET="sk_live_xxx"
TIMESTAMP=$(date +%s)
SIGN_STRING="${TIMESTAMP}${API_KEY}"
SIGNATURE=$(echo -n "$SIGN_STRING" | openssl dgst -sha256 -hmac "$SECRET" | cut -d' ' -f2)

curl -X GET "http://43.133.43.2/openapi/points/mall/products?page=1&page_size=10" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${API_KEY}" \
  -H "X-Timestamp: ${TIMESTAMP}" \
  -H "X-Signature: ${SIGNATURE}"
```

---

## 相关文档

- [认证机制](./authentication.md) - 签名算法详解
- [错误码参考](./error-codes.md) - 错误处理
- [接入指南](./integration-guide.md) - 快速开始
