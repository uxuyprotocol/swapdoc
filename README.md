# UXUY Swap API 文档

本文档介绍 UXUY Swap API 的 EVM 与 Solana 链接口，以及网关 JWT 鉴权方式。

## EVM

- [离线询价](swap/evm-offline-quote-api.md)：了解 `dex_offlineQuote` 的请求参数、路由结果及交易数据。

## Solana

- [离线询价与兑换指令](swap/solana-offline-swap-api.md)：了解 `dex_offlineQuoteSol` 报价和 `dex_offlineSwapSol` 兑换指令的组装流程。

## 鉴权

- [JWT 鉴权（RS256 / ES256）](swap/jwt-auth.md)：了解密钥对生成、公钥注册及请求签名方式。