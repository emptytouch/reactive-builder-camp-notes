# D5 任务提交件｜定时执行：Cron 触发回调 + 最多 N 次自动停止

执行时间：2026-11-22 23:40（GMT+8）｜方案 A（定时执行）｜Reactive 侧 Lasna（5318007），源链/目标链 Ethereum Sepolia（11155111），无主网混用。

> 为快速拿到两次触发，练习用 **Cron10**（约每 10 个区块 / 10 秒一次）；正式场景按场景手册换 `Cron100`（约 2 分钟），同样的 `LIMIT` 退订逻辑不用改。

## 一、合约地址

| 合约 | 链 | 地址 |
|---|---|---|
| `CronReactive`（订阅 + 计数 + 自动退订） | Lasna | https://lasna.reactscan.net/address/0xb3a2fb06bb0d62c973cb0739149cc82c58e87afa |
| `CronCallback`（目标链接收方） | Sepolia | https://sepolia.etherscan.io/address/0xca09f20c4548adef5015e5e5330c2a7b4347ffa6 |

新增源码：`starter/src/CronReactive.sol`、`starter/src/CronCallback.sol`（未改模板原有文件）。

## 二、至少两次触发的交易链接

| 次数 | Lasna 上的 `react()` 交易 | 触发区块 |
|---|---|---|
| 第 1 次 | https://lasna.reactscan.net/tx/0x464d6fc8d1c379f18e0cf7be3ca597ecbdbff033a73a3de2a124e3a4817a0fd0 | 8984791 |
| 第 2 次 | https://lasna.reactscan.net/tx/0x84bfbebcf1af91ddd3b9358f9ff83dfe12de3c4b935e0806f55b6b4f26705558 | 8984801 |

两个区块正好差 **10 个区块**，符合 Cron10 的间隔；两次都发了回调到 Sepolia：

| 次数 | 目标链回调交易 |
|---|---|
| 第 1 次 | https://sepolia.etherscan.io/tx/0x0b8a1bfadc851463ddaab69a9f1ecfe0b005bb1a2b8ce8719df6edc384132034 |
| 第 2 次 | https://sepolia.etherscan.io/tx/0x1de2a38bb9c4c61a8ff8f1c17b7056d6bd1ebe045d6f48b04f29558dda160449 |

链上状态：`ticks() = 2`、`LIMIT() = 2`、目标链 `count() = 2`、`debt = 0`。

## 三、如何防止无限执行或耗尽余额（一段说明）

** hardcoded 上限 + 自动退订 + 早退 + 预充值 + 时间安全** 五层防护：

1. 构造参数 `LIMIT = 2`，`react()` 里 `ticks += 1`，一旦 `ticks >= LIMIT` 立刻调用 `SYSTEM.unsubscribe(block.chainid, 0x…fffFfF, CRON10_TOPIC_0, REACTIVE_IGNORE ×3)` 取消订阅——六项参数与构造时的 `subscribe` 完全一致，避免静默失败。达到上限后再次 tick 会直接 `return`，不再请求回调。
2. `react()` 开头三重早退：`topic0` 不匹配（不是 Cron10）、`recipient == 0`、或 `ticks >= LIMIT` 都 `return`，**不 revert**——Cron 每个 tick 都收费，revert 照样扣钱。
3. 部署时用 `--value 0.5 ether` 预存 lREACT，当前余额 **0.4125496 lREACT**、`debt = 0`，不会因欠款被静默停摆。
4. 反证：第 2 次触发后 Lasna 又前进了 **64 个区块**（约 6 个 Cron10 间隔），`ticks()` 仍停在 2、目标链 `count()` 仍是 2——订阅确实被取消，不是只把计数封顶。
5. 目标链回调合约用 `block.timestamp` 记录 `lastTimestamp`，而不是区块数；换成 Base Sepolia（200ms 出块）时计时不会被区块节奏带偏。回调还有 `onlyServiceProvider` + `onlyCallbackSender(reactive_)` 两道校验，防止别人冒充触发。

真要上生产，还要把 Cron10 换成能接受的最长间隔（Cron100/Cron1000）——**每次 tick 都收费，哪怕 `react()` 直接 return**，间隔越短固定成本乘得越多。

## 四、部署与核验命令（可复现）

```bash
set -a; source .env; set +a

# ① Lasna：limit = 2，预存 0.5 lREACT
ARGS=$(cast abi-encode "constructor(uint256,uint256)" 11155111 2)
cast send --rpc-url "$LASNA_RPC_URL" --private-key "$PRIVATE_KEY" \
  --gas-limit 2000000 --gas-price 100000000000 --value 0.5ether \
  --create "$(forge inspect CronReactive bytecode)${ARGS#0x}"

# ② Sepolia：回调合约绑定 CronReactive，带 0.02 ETH 付回调 gas
ARGS=$(cast abi-encode "constructor(address,address)" "$CALLBACK_PROXY" 0xb3a2fb06bb0d62c973cb0739149cc82c58e87afa)
cast send --rpc-url "$SEPOLIA_RPC_URL" --private-key "$PRIVATE_KEY" \
  --gas-limit 4000000 --gas-price 1000000000 --value 0.02ether \
  --create "$(forge inspect CronCallback bytecode)${ARGS#0x}"

# ③ 绑定 recipient，然后等两次 Cron10 tick
cast send --rpc-url "$LASNA_RPC_URL" --private-key "$PRIVATE_KEY" \
  0xb3a2fb06bb0d62c973cb0739149cc82c58e87afa "setRecipient(address)" 0xca09f20c4548adef5015e5e5330c2a7b4347ffa6

# ④ 核验
cast call --rpc-url "$LASNA_RPC_URL" 0xb3a2fb06bb0d62c973cb0739149cc82c58e87afa 'ticks()(uint256)'
cast call --rpc-url "$SEPOLIA_RPC_URL" 0xca09f20c4548adef5015e5e5330c2a7b4347ffa6 'count()(uint256)'
```

## 五、一句话结论

Cron10 定时触发 `react()` 两次并成功回调到 Sepolia，达到 `LIMIT=2` 后合约自动退订——64 个区块过去再无新触发，且全程 `debt=0`、余额充足，无限执行和余额耗尽都被硬上限挡住了。
