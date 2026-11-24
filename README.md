# UXUY DEX API

UXUY DEX API 为钱包、交易聚合器、Bot 等业务方提供链上兑换（Swap）的**报价**与**交易构建**能力：调用一次接口即可得到最优路由、目标合约与可直接广播的 calldata（Solana 为待签名指令链），签名与广播由业务方用自有钱包完成。

- **协议**：JSON-RPC 2.0，`POST`，`Content-Type: application/json`
- **鉴权**：按 App 维度的 JWT（`RS256` / `ES256`），见[鉴权](swap/jwt-auth.md)
- **报价**：服务端基于本地池状态计算；EVM 本地无路由或数据不达标时自动回落链上询价，业务方无需区分。Solana 全部基于本地状态（无在线回落）
- **执行**：不托管资产、不代签、不代发交易；本服务只产出 calldata / 指令链

## 支持的链

请求 URL 中的 `<chain>` 取下表 `chain` 列：

| `chain` | 类型 | chainId / chainIndex | 询价 | 交易构建 |
|---|---|---|---|---|
| `bnbchain` | EVM | 56 | ✅ | ✅ calldata |
| `ethereum` | EVM | 1 | ✅ | ✅ calldata |
| `base` | EVM | 8453 | ✅ | ✅ calldata |
| `xlayer` | EVM | 196 | ✅ | ✅ calldata |
| `anubis` | EVM | 6714 | ✅ | ✅ calldata |
| `solana` | Solana | `chainIndex` = 501 | ✅ | ✅ 指令链 |

- 链名大小写不敏感（`BNBChain` 与 `bnbchain` 等价）。
- `anubis`（chainId 6714）为 EVM 兼容 L1，原生 gas 代币为 gasDAI（DAI 锚定）；接口中原生币仍以零地址表示。
- EVM 的 chainId 用于请求参数 `chainIn` / `chainOut`。
- Solana 请求如需显式传 `chainIndex`，填 `"501"`。
- 其他 EVM 链可按需开通，联系 BIPAL 团队。

## 接入流程

```
① 申请凭证 ──▶ ② 鉴权 ──▶ ③ 询价 ──▶ ④ 授权 / 组包 ──▶ ⑤ 签名广播
 (AppId+密钥对)   (JWT)     (报价+calldata)   (仅 EVM 需授权)     (业务方执行)
```

1. **申请凭证**：向 BIPAL 团队申请 AppId 与 JWT 密钥对，获取测试/生产地址。
2. **鉴权**：用私钥签发 JWT，请求头携带 `Authorization: Bearer <JWT>`。
3. **询价**：EVM 调用 `dex_offlineQuote`，Solana 调用 `dex_offlineQuoteSol`。
4. **授权 / 组包**：EVM 对输入代币授权给返回的 `calldata.to`；Solana 用 `dex_offlineSwapSol` 取指令链。
5. **执行**：业务方签名广播，链上完成兑换。

完整步骤与代码骨架见[接入流程](swap/quickstart.md)。

## 文档导航

| 文档 | 内容 |
|---|---|
| [接入流程](swap/quickstart.md) | 从申请凭证到首笔交易上链，含 EVM / Solana 代码骨架与联调清单 |
| [鉴权](swap/jwt-auth.md) | 密钥对生成、公钥注册、JWT 签发与排错 |
| [EVM 询价与交易](swap/evm-offline-quote-api.md) | `dex_offlineQuote` / `dex_offlineQuotes` 参数与响应、授权、发交易、失败处理 |
| [Solana 询价与兑换](swap/solana-offline-swap-api.md) | `dex_offlineQuoteSol` / `dex_offlineSwapSol` 参数与响应、指令签名广播 |

## 通用约定

### 请求格式

```json
{"jsonrpc":"2.0","id":1,"method":"<方法名>","params":[ ... ]}
```

- `id` 由业务方指定，响应原样回传，用于请求与响应配对。
- 成功时结果在 `result`；接口层失败返回 HTTP 200 + JSON-RPC 错误对象 `{"error":{"code":-32000,"message":"…"}}`，`message` 会点名出错的参数。
- 鉴权失败返回 **HTTP 401**，响应体为纯文本错误信息；请求过频返回 **HTTP 429**。业务方应分别处理「HTTP 非 200」与「JSON-RPC error」两类失败。

### 地址与金额

| 项目 | EVM | Solana |
|---|---|---|
| 地址格式 | hex（`0x` 开头） | base58 |
| 原生币 | 零地址 `0x0000000000000000000000000000000000000000` | `11111111111111111111111111111111` |
| 请求金额 | 人类可读十进制字符串（`"1"` = 1 枚） | 最小单位整数字符串（`"1000000"` = 1 枚 6 位精度代币） |
| 响应金额 | 未除精度的整数（JSON number） | 最小单位整数字符串 |
| 滑点 | `slippage`，单位 1e-6（`5000` = 0.5%） | `slippagePercent`，百分比字符串（`"0.5"` = 0.5%） |

> **大整数精度**：EVM 响应中的 `amount` / `minAmount` / `calldata.value` 等字段是未除精度的整数，数值可能超过 2<sup>53</sup>。请使用支持大整数的 JSON 解析（如 `json-bigint`），或先按文本取出这些字段再转 `BigInt`，不要直接用 `Number` 累加或比较。

### 环境

| 环境 | 地址 |
|---|---|
| 生产 | `https://gwapi.ourdex.com/<chain>/<appId>` |
| 测试 | 由 BIPAL 团队单独提供 |

测试环境与生产环境的 AppId、密钥对、接口地址互不通用。

## 技术支持

- 接入申请、链与合约开通、限额与费率调整：联系 BIPAL 团队
- 请求报错请附：接口名、`chain`、`appId`、请求体、响应体、时间（含时区）
