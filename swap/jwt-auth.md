# 鉴权

UXUY DEX API 按 **App** 维度鉴权：每个请求都必须携带由业务方私钥签发的 JWT。

- **请求地址**：`https://gwapi.ourdex.com/<chain>/<appId>`（测试环境地址由 BIPAL 团队提供）
- **请求头**：`Authorization: Bearer <JWT>`
- **签名算法**：`RS256`（RSA 2048+）或 `ES256`（EC P-256），其他算法一律拒绝

本文以 App 已启用 JWT 校验为前提（正式接入默认启用）；App 未启用时仅凭 URL 中的 AppId 即可访问，是否启用由平台配置决定。

```
业务方                               UXUY 平台
  │  ① 生成密钥对（私钥不出本地）          │
  │  ② 提交公钥 PEM ─────────────────▶  注册公钥，返回 jwtId
  │  ③ 私钥本地签发 JWT                   │
  │     header: {"alg":"RS256","jti":"<jwtId>"}
  │  ④ POST /<chain>/<appId>  ─────────▶  按 jti 查公钥 → 验签 → 校验 exp
  │     Authorization: Bearer <JWT>      │
```

## 三个容易混淆的概念

| 名称 | 含义 | 出现位置 |
|---|---|---|
| **AppId** | App 的唯一标识 | 请求 URL 路径 `/<chain>/<appId>` |
| **jwtId** | 某个公钥在该 App 下的唯一标识（注册公钥时由平台生成） | JWT 的 **header** 字段 `jti` |
| **`jti`** | JWT 标准头字段，值必须等于 jwtId | JWT 第一段（base64 解码后的 JSON） |

> `jti` 必须放在 **JWT 自身的 header**（token 第一段）里，不是 HTTP header，也不是 claims。这是本平台的**非标准**做法：多数 JWT 库默认把 `jti` 写进 payload，直接签名会被拒绝（`jwtId nil`）。

## 第 1 步：生成密钥对

自行生成时（推荐，私钥不出本地）：

```bash
# RS256（RSA 2048）
openssl genrsa -out rs256_key.pem 2048
openssl rsa -in rs256_key.pem -pubout -out rs256_pub.pem

# ES256（EC prime256v1 / P-256）
openssl ecparam -name prime256v1 -genkey -noout -out es256_key.pem
openssl ec -in es256_key.pem -pubout -out es256_pub.pem
```

- `*_key.pem`：私钥，仅保存在业务方服务端，用于签发 JWT
- `*_pub.pem`：公钥，提交给平台注册
- PKCS#1（`BEGIN RSA PRIVATE KEY`）与 PKCS#8（`BEGIN PRIVATE KEY`）均可

## 第 2 步：注册公钥，获取 jwtId

将公钥 PEM 全文提交给 BIPAL 团队，平台注册后返回 **jwtId**：

- 同一公钥重复注册会被拒绝（按公钥指纹去重）
- **公钥注册在哪个 App 下，就只能配该 App 的 AppId 使用**；换 App 需重新注册
- 删除公钥后，该 jwtId 将不再通过校验；App 配置有缓存，最长约 1 小时后完全生效

如需自行核对 jwtId（或自动化注册前预检），计算方式为：

```
jwtId = MD5( Base64Std( SHA256( DER(公钥) ) ) )   # 结果取 32 位小写 hex
```

```bash
# 与平台逐字节一致；macOS 用 md5，Linux 用 md5sum
openssl pkey -pubin -in rs256_pub.pem -outform DER \
  | openssl dgst -sha256 -binary | base64 | tr -d '\n' | md5
```

`tr -d '\n'` 不能省：`base64` 输出带尾部换行，而平台对**不含换行**的 base64 字符串求 MD5。通常直接用平台返回的 jwtId 即可，无需自己计算。

## 第 3 步：签发 JWT

### 三条硬性要求

1. **`jti` 放在 token header**，值为 jwtId，缺了会报 `jwtId nil`
2. **务必自设 `exp`**：服务端对 `exp` 有则校验、无则放行，不带 `exp` 的 token 永不过期；建议有效期 5 分钟
3. **算法只能是 `RS256` / `ES256`**

### Go（golang-jwt/jwt/v4）

```go
package main

import (
	"fmt"
	"os"
	"time"

	"github.com/golang-jwt/jwt/v4"
)

func main() {
	alg, keyPath, jwtID := os.Args[1], os.Args[2], os.Args[3] // RS256|ES256, 私钥PEM, jwtId

	pemBytes, err := os.ReadFile(keyPath)
	if err != nil {
		panic(err)
	}

	var method jwt.SigningMethod
	var key interface{}
	switch alg {
	case "RS256":
		method = jwt.SigningMethodRS256
		key, err = jwt.ParseRSAPrivateKeyFromPEM(pemBytes)
	case "ES256":
		method = jwt.SigningMethodES256
		key, err = jwt.ParseECPrivateKeyFromPEM(pemBytes)
	default:
		panic("only RS256 and ES256 are supported")
	}
	if err != nil {
		panic(err)
	}

	now := time.Now()
	t := jwt.NewWithClaims(method, jwt.MapClaims{
		"exp": now.Add(5 * time.Minute).Unix(),
	})
	t.Header["jti"] = jwtID // 关键：写在 header
	token, err := t.SignedString(key)
	if err != nil {
		panic(err)
	}
	fmt.Println(token)
}
```

### Node.js（jsonwebtoken）

```js
const fs = require('fs');
const jwt = require('jsonwebtoken');

const JWT_ID = 'your-jwt-id';
const privateKey = fs.readFileSync('rs256_key.pem');

const token = jwt.sign(
  { exp: Math.floor(Date.now() / 1000) + 300 },
  privateKey,
  { algorithm: 'RS256', header: { jti: JWT_ID } }   // 关键：写在 header
);
```

### Python（PyJWT 2.x）

```python
import time
import jwt

JWT_ID = "your-jwt-id"
private_key = open("rs256_key.pem").read()

token = jwt.encode(
    {"exp": int(time.time()) + 300},
    private_key,
    algorithm="RS256",
    headers={"jti": JWT_ID},   # 关键：写在 header
)
```

### 自检

把 token 按 `.` 切开，base64url 解码第一段，应看到 `jti`：

```bash
echo "$TOKEN" | cut -d. -f1 | tr '_-' '/+' | base64 -d 2>/dev/null
# {"alg":"RS256","jti":"<jwtId>","typ":"JWT"}
```

## 第 4 步：调用接口

```bash
curl -s -X POST 'https://gwapi.ourdex.com/bnbchain/<appId>' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"dex_version","params":[]}'
```

## 有效期与复用

- 私钥本地签名，签发 JWT 不需要网络请求，性能开销极小
- 建议有效期 5 分钟，并在过期前复用同一 token（按有效期缓存，提前 30–60 秒刷新）
- 服务端校验证签，并对存在的时间类 claims（`exp` / `nbf` / `iat`）做有效性校验，不比对 `iss`；时钟偏差大时请校准 NTP

## 安全要求

- 私钥只在服务端使用；前端、移动端不得内置私钥（应由业务方服务端代签）
- 公私钥文件不要提交到代码仓库；建议放密钥管理服务并定期轮换
- 全部请求走 HTTPS；token 不要出现在 URL、日志或前端存储中
- 公钥泄露不构成风险；私钥泄露时立即联系团队删除该 Key（App 配置缓存最长 1 小时，删除后请同时暂停使用旧私钥签发的令牌）

## 常见错误

| 返回 / 报错 | 原因 | 处理 |
|---|---|---|
| `jwtId nil` | JWT header 里没有 `jti` | 把 jwtId 写进 token header（不是 payload） |
| `jwt auth id should be empty` / 401 | `jti` 与该 AppId 下注册的 jwtId 不一致；Key 已删除；公钥注册在别的 App 下 | 核对 AppId 与 jwtId 的配对关系 |
| `unexpected signing method` | 算法不是 RS256 / ES256 | 改用 RS256 或 ES256 |
| `Token is expired` | `exp` 已过期 | 重新签发 |
| `authentication method is not supported` | 缺少 Bearer、用了 Basic Auth | 使用 `Authorization: Bearer <JWT>` |
| `supports the algorithms RS256 and ES256` | 提交的公钥 PEM 既不是 RSA 也不是 EC 公钥（如误传证书、私钥或格式损坏） | 用 `openssl *_pub.pem` 的输出重新提交 |
| `publicKey existed` | 同一公钥重复注册 | 直接使用已返回的 jwtId |
