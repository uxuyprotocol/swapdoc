# API 认证：JWT（RS256 / ES256）

JSON-RPC 网关按 App 维度进行鉴权。启用 JWT 验证的 App，请求必须携带
`Authorization: Bearer <JWT>`。

- **请求地址**：`https://gwapi.ourdex.com/<chain>/YOUR-APP-ID`
- **支持算法**：**RS256**（RSA）和 **ES256**（EC P-256）；其他算法均会被拒绝。

```
┌──────────────┐   ①openssl 生成密钥对    ┌──────────────┐
│  业务方(客户端) │ ──── 公钥注册 ────────▶ │  gdexUI Keys  │
│              │                          │  (服务端存公钥) │
│              │ ◀─── jwtId(=jti) ────── │              │
│              │                          └──────────────┘
│  ②私钥签发 JWT │  header 必带 jti（claims 可选）
└──────┬───────┘
       │ ③POST /<chain>/<appId>
       │   Authorization: Bearer <JWT>
       ▼
   gdex 网关：按 header jti 查公钥 → 验签 → claims.Valid()（exp）
```

仓库内**没有**密钥对生成脚本。gdexUI 的 Keys 页只用于查询、粘贴注册和删除公钥（`cmd/gdexUI/web_app.go`）；密钥对需使用 OpenSSL 或任意 JWT 库生成。`cmd/gdexUI/dexio/client.go` 中内置的 `RSPrivateKey` 是 gdexUI 自身访问主服务的固定身份，**勿用于业务 App**。

参考实现：

- **校验**：`plugins/uxuyio/app.go`（`AppValidator` / `doJwtValidator`）
- **签发**：`cmd/gdexUI/dexio/client.go`（`setJwt`）

## 1. 生成密钥对（openssl）

```bash
# RS256 (RSA 2048)
openssl genrsa -out rs256_key.pem 2048
openssl rsa -in rs256_key.pem -pubout -out rs256_pub.pem

# ES256 (EC prime256v1 / P-256)
openssl ecparam -name prime256v1 -genkey -noout -out es256_key.pem
openssl ec -in es256_key.pem -pubout -out es256_pub.pem
```

- `*_key.pem` 是**私钥**，留在业务方侧用于签发 JWT，绝不外传
- `*_pub.pem` 是公钥，用于下一步注册
- PKCS#1（`BEGIN RSA PRIVATE KEY`）与 PKCS#8（`BEGIN PRIVATE KEY`）格式均可解析

## 2. 注册公钥，拿到 jwtId

gdexUI → 对应 App → Keys 页粘贴公钥 PEM 注册（App 验证方式选 jwt，即 `JwtVerify=true`）。

注册时服务端从公钥派生唯一 ID（`plugins/uxuyio/app.go` `AddJwt`）：

```
jti = MD5( Base64Std( SHA256( X509.MarshalPKIXPublicKey(公钥) ) ) )
```

Keys 列表里展示的 **jwtId 就是签发 JWT 要用的 jti**，直接抄即可，无需自行计算。

> **App Id ≠ jti，两者是完全不同的东西：**
>
> - **App Id**：创建 APP 时自己指定的（App Center → Apps 列表 "App Id" 列），用在请求 URL `/<chainName>/<appId>` 里
> - **jti**（= 该 APP → Keys 页 "Jwt Id" 列）：注册公钥时服务端按上式算出的，签发 JWT 时放进 **token header**
>
> 把 App Id 当 jti 填进 JWT header，服务端在该 APP 的公钥表里查不到，直接 401（`jwt auth id should be empty`）。

需要程序化计算 jti（自动化注册/自检场景）时，以下两种方式等价，均与服务端 `AddJwt` 逐字节一致：

**Go**（与服务端同一套标准库调用）：

```go
// 用法: go run jti.go rs256_pub.pem
package main

import (
	"crypto/md5"
	"crypto/sha256"
	"crypto/x509"
	"encoding/base64"
	"encoding/pem"
	"fmt"
	"os"
)

func main() {
	pemBytes, err := os.ReadFile(os.Args[1])
	if err != nil {
		panic(err)
	}
	block, _ := pem.Decode(pemBytes)
	if block == nil {
		panic("not a PEM file")
	}
	pub, err := x509.ParsePKIXPublicKey(block.Bytes) // openssl -pubout 输出即 PKIX(SPKI) 格式
	if err != nil {
		panic(err)
	}
	der, err := x509.MarshalPKIXPublicKey(pub)
	if err != nil {
		panic(err)
	}
	sum := sha256.Sum256(der)
	fingerprint := base64.StdEncoding.EncodeToString(sum[:])
	fmt.Printf("%x\n", md5.Sum([]byte(fingerprint))) // = Keys 页展示的 jwtId（jti）
}
```

**openssl 一行流**（RS256/ES256 通用）：

```bash
openssl pkey -pubin -in rs256_pub.pem -outform DER \
  | openssl dgst -sha256 -binary | base64 | tr -d '\n' | md5   # macOS；Linux 末段换成 md5sum
```

注意 `tr -d '\n'` 不能省：`base64` 输出带尾部换行，而服务端是对**不含换行**的 base64 字符串做 MD5，带上换行算出的值必然对不上（RS256/ES256 双算法实测与 Go 版逐字节一致）。

注意：

- 同一公钥重复注册会报 `publicKey existed`（按指纹去重）
- **公钥注册在哪个 APP 下，就只能配那个 APP 的 appId 使用**：`JwtKeys` 按 App 维度存储，请求 URL 里的 appId 与注册时的 APP 不符 → 该 APP 的表里查不到此 jti → 401
- 删除 Key 后该 jti 的所有 token 立即失效（查不到公钥即拒绝）

## 3. 签发 JWT

### 必须遵守的三点

1. **`jti` 必须放进 JWT 自身的 header；claims 里可带可不带（服务端不读 claims 里的 `jti`）**。注意这里的 header 指 **token 第一段**（base64 解码后的 JSON），**不是 HTTP header**——HTTP 层只需要 `Authorization: Bearer <token>` 一个头，`jti` 编码在 token 内部，抓包看不到。服务端从 `token.Header["jti"]` 查公钥（`doJwtValidator`），缺失直接报 `jwtId nil`。验证方法：把 token 按句号切开、base64 解码第一段，应看到 `{"alg":"RS256","jti":"...","typ":"JWT"}`。这是非标准做法——`pyjwt`/`jose` 等库默认只把 `jti` 写进 payload，直接签出来会被拒
2. claims **建议**带 `exp`（有效期）——服务端（golang-jwt v4）对 `exp` 是**有则校验、无则放行**，不带 `exp` 的 token 永不过期，因此强烈建议自设分钟级有效期（参考实现默认 5 分钟）。服务端只做签名校验 + `claims.Valid()`（exp/nbf），`iss` 不做比对
3. 算法只能是 `RS256` / `ES256`

### Go 示例（`golang-jwt/jwt/v4`，与仓库同款）

```go
package main

import (
	"fmt"
	"os"
	"time"

	"github.com/golang-jwt/jwt/v4"
)

// 用法: go run main.go <RS256|ES256> <私钥PEM路径> <jwtId> [issuer]
func main() {
	alg, keyPath, jwtID := os.Args[1], os.Args[2], os.Args[3]
	issuer := "app"
	if len(os.Args) > 4 {
		issuer = os.Args[4]
	}

	pemBytes, err := os.ReadFile(keyPath)
	if err != nil {
		panic(err)
	}

	var method jwt.SigningMethod
	var key interface{}
	switch alg {
	case "RS256":
		method = jwt.SigningMethodRS256
		if key, err = jwt.ParseRSAPrivateKeyFromPEM(pemBytes); err != nil {
			panic(err)
		}
	case "ES256":
		method = jwt.SigningMethodES256
		if key, err = jwt.ParseECPrivateKeyFromPEM(pemBytes); err != nil {
			panic(err)
		}
	default:
		panic("only RS256 and ES256 are supported")
	}
	token, err := newToken(method, jwtID, issuer, key)
	if err != nil {
		panic(err)
	}
	fmt.Println(token)
}

func newToken(method jwt.SigningMethod, jwtID, issuer string, key interface{}) (string, error) {
	now := time.Now()
	t := jwt.NewWithClaims(method, jwt.MapClaims{
		"jti": jwtID, // 可选：服务端不读 claims 里的 jti，带上无害
		"iss": issuer,
		"ts":  now.Unix(), // 参考实现带的字段，服务端不校验
		"exp": now.Add(5 * time.Minute).Unix(),
	})
	t.Header["jti"] = jwtID // 关键：header 里也要有
	return t.SignedString(key)
}
```

### 直接复用仓库客户端

Go 业务方可以直接用 `cmd/gdexUI/dexio` 包，每次请求自动签发并附带 Bearer 头：

```go
client, err := dexio.NewClient(ctx, "https://gwapi.ourdex.com",
	dexio.WithForceJwt(true),
	dexio.WithJwtId(jwtID),
	dexio.WithJwtIssuer("app"),
	dexio.WithJwtPrivate(dexio.JWT_ALG_RS256, privateKeyPEM), // ES256 换 JWT_ALG_ES256
)
```

## 4. 发请求

URL 仍是 `/<chainName>/<appId>`，token 放 header：

```bash
TOKEN=$(go run main.go RS256 rs256_key.pem YOUR-JWT-ID myapp)

curl -s -X POST 'https://gwapi.ourdex.com/bnbchain/YOUR-APP-ID' \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Content-Type: application/json' \
    -d '{"jsonrpc":"2.0","id":1,"method":"dex_getCoin","params":["0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82"]}'
```

token 过期后重新签发即可（私钥本地签，无需网络请求）；有效期建议与参考实现一致保持分钟级。

## 5. 排错对照

| 报错 | 原因 |
| --- | --- |
| `jwtId nil` | JWT header 里没有 `jti`（见上文第 3 节第 1 点） |
| `jwt auth id should be empty` / 查无此 Key | header `jti` 与**该 APP** 注册的 jwtId 不一致；Key 已被删除；公钥注册在了别的 APP 下（URL 里 appId 用错）；服务端注册未落盘/重启丢失 |
| `unexpected signing method` | alg 不是 RS256/ES256 |
| `token is expired` / 401 | `exp` 过期，重新签发 |
| `authentication method is not supported` | 带了 Basic Auth 或未带 Bearer（App 已开启 JwtVerify） |
| `supports the algorithms RS256 and ES256` | 注册阶段公钥 PEM 格式/类型不对 |
