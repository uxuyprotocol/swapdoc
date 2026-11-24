# 接入流程

从零到首笔交易上链共五步。EVM 与 Solana 的前三步完全一致，第 4 步起按链分叉。

```
① 申请凭证 ──▶ ② 鉴权 ──▶ ③ 询价 ──┬─▶ ④A 授权（仅 ERC20）──▶ ⑤A 发送交易      【EVM】
                                    └─▶ ④B 组装指令链 ─────────▶ ⑤B 签名广播      【Solana】
```

## 第 1 步：申请 AppId 与密钥对

联系 **BIPAL 团队**，提供以下信息：

| 需要提供 | 说明 |
|---|---|
| 接入链 | 如 `bnbchain`、`base`、`solana`，可多选 |
| 业务场景 | 钱包 / 聚合器 / Bot / 其他，便于评估限额与费率 |
| 预计量级 | 日请求数、单笔金额上限 |
| 联系人 | 技术与商务联系人、紧急联系方式 |

团队会发放：

| 凭证 | 用途 | 使用位置 |
|---|---|---|
| **AppId** | 标识业务方 App | 请求 URL 路径：`/<chain>/<appId>` |
| **JWT 密钥对** | 签发 JWT | 私钥仅保存在业务方服务端 |
| **jwtId** | 标识该公钥 | 写入 JWT 的 **header** 字段 `jti` |
| 测试环境地址 | 联调用 | 与生产隔离的独立 `<baseUrl>` |

密钥对可自行生成后提交公钥（推荐，私钥不出本地），也可由团队生成后通过安全渠道交付。生成与注册方式见[鉴权](jwt-auth.md)。

> **AppId ≠ jwtId**：AppId 在 URL 路径里，jwtId 在 JWT 内部。把 AppId 当 `jti` 使用会直接返回 401。

## 第 2 步：鉴权并打通首个请求

用私钥签发 JWT（有效期建议 5 分钟），携带 `Authorization: Bearer <JWT>` 调用一个只读接口验证链路是否打通：

```bash
TOKEN="<按「鉴权」文档签发的 JWT>"

curl -s -X POST 'https://gwapi.ourdex.com/bnbchain/<appId>' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"dex_version","params":[]}'
```

返回 `result` 即为连通；返回 401 时对照[鉴权](jwt-auth.md)末尾的常见错误表排查。

## 第 3 步：询价

EVM：

```bash
curl -s -X POST 'https://gwapi.ourdex.com/bnbchain/<appId>' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"dex_offlineQuote","params":[{
        "sender":"0x...用户地址",
        "tokenIn":"0x55d398326f99059fF775485246999027B3197955",
        "tokenOut":"0x0E09FaBB73Bd3Ade0a17ECC321fD13a19e81cE82",
        "amountIn":"1",
        "slippage":5000,
        "withTx":true
      }]}'
```

Solana：

```bash
curl -s -X POST 'https://gwapi.ourdex.com/solana/<appId>' \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"dex_offlineQuoteSol","params":[{
        "fromTokenAddress":"11111111111111111111111111111111",
        "toTokenAddress":"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "amount":"100000000",
        "slippagePercent":"0.5"
      }]}'
```

EVM 的 `result` 是路由数组，取 `result[0]`（已按可兑换数量排序）；Solana 的 `result` 直接就是最优路由对象。参数与响应字段详见[EVM 询价与交易](evm-offline-quote-api.md)、[Solana 询价与兑换](solana-offline-swap-api.md)。

## 第 4 步（EVM 线）：授权输入代币

从询价结果取 `calldata`：

| 字段 | 用途 |
|---|---|
| `calldata.to` | 交易目标合约，同时也是 **approve 的 spender** |
| `calldata.data` | 交易的 `data` |
| `calldata.value` | 交易的 `msg.value`（原生币买入时非 0） |

- `tokenIn` 是 **ERC20**：先 `approve(calldata.to, ≥ amountIn)`，一次性授权较大额度可减少后续交易笔数。
- `tokenIn` 是 **原生币**（零地址）：无需授权，按 `calldata.value` 发送即可。

```js
// ethers v6
const erc20 = new ethers.Contract(tokenIn, [
  'function allowance(address owner, address spender) view returns (uint256)',
  'function approve(address spender, uint256 amount) returns (bool)',
], signer);

const need = BigInt(amountInWei);
if (await erc20.allowance(wallet.address, calldata.to) < need) {
  await (await erc20.approve(calldata.to, ethers.MaxUint256)).wait();
}
```

> 授权对象必须是 `calldata.to`（链上真正执行 `transferFrom` 的合约），**不是** DEX 的 router、pool 或 factory 地址。授权错地址会在发送交易时 revert `UP: token allowance is not enough`。

## 第 5 步（EVM 线）：发送并确认交易

```js
const tx = await signer.sendTransaction({
  to: calldata.to,
  data: '0x' + calldata.data,     // 接口返回的 data 不带 0x 前缀
  value: BigInt(calldata.value),
});
const receipt = await tx.wait();
if (receipt.status !== 1) throw new Error('swap reverted');
```

- `sender`（询价参数）必须是实际签名地址：合约按 `msg.sender` 校验余额与授权。
- calldata 内含截止时间，默认 **5 分钟**（Ethereum 主网 30 分钟），超时上链会 revert `CommonBase: transaction too old`，此时重新询价。
- 成交低于 `minAmount` 会整笔 revert（典型报错 `UxuySwap: swapped amount less than minAmountOut`，部分 DEX 直接透传其自身错误）；确认报价与滑点是否符合预期。

## 第 4 步（Solana 线）：组装指令链

调用 `dex_offlineSwapSol`（继承询价参数，`userWalletAddress` 必填，建议 `simulate:true`）：

- 返回 `instructionLists`（待组装指令）与 `unsignedTransaction`（已组装好的 v1 交易，base64）。
- 拿 `unsignedTransaction` 反序列化后签名最省事；需要自定义指令时用 `instructionLists` + `tx.computeBudget` 自组。
- `simulateResult.ok` 为 `false` 时不要广播，重新询价或换路。

## 第 5 步（Solana 线）：签名并广播

```js
// @solana/web3.js（需支持 v1 / SIMD-0385 交易反序列化的版本）
const tx = VersionedTransaction.deserialize(Buffer.from(res.unsignedTransaction, 'base64'));
tx.sign([userKeypair]);              // 用户钱包
if (feePayerKeypair) tx.sign([feePayerKeypair]);  // 传了 feePayer 时

await connection.sendRawTransaction(tx.serialize(), { maxSupportedTransactionVersion: 1 });
```

Solana 无 approve 环节；输入代币的 ATA 创建、wrap/unwrap、平台费转账均由指令链覆盖。

## 联调检查清单

| 检查项 | 说明 |
|---|---|
| `chain` 路径 | 使用已开通的链名（当前为 `bnbchain` / `ethereum` / `base` / `xlayer` / `solana`） |
| AppId 与 jwtId 配对 | 公钥注册在哪个 App 下，就只能用该 App 的 AppId |
| `jti` 位置 | 在 JWT **header**（token 第一段）里，不在 payload 里 |
| 金额单位 | EVM 人类可读字符串，Solana 最小单位整数字符串 |
| 原生币写法 | EVM 零地址，Solana `111…1` |
| 大整数解析 | EVM 响应中的数量用 `BigInt` 解析，避免精度丢失 |
| `sender` | 必须与实际签名地址一致 |
| 授权 spender | `calldata.to`，不是 DEX router |
| Solana 预执行 | 生产环境建议 `simulate:true` 且 `ok=true` 再广播 |
| 交易版本 | Solana 广播与查询统一带 `maxSupportedTransactionVersion: 1` |

## 下一步

- [鉴权：密钥对、公钥注册与 JWT 签发](jwt-auth.md)
- [EVM 询价与交易：完整参数、授权与失败处理](evm-offline-quote-api.md)
- [Solana 询价与兑换：指令链、签名与广播](solana-offline-swap-api.md)
