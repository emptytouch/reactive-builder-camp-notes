# D3 任务提交件｜Lasna 部署监控合约，订阅 Sepolia ERC-20 Transfer

执行时间：2026-11-22 23:13（GMT+8）｜基于模板 `starter/src/CampReactive.sol`（Omni 写法），未改模板原有源码。

## 提交三件套

**① 合约地址（Reactscan）**

```
0x23d8de308079caeec9409d7a66f1e4b934d749d0
https://lasna.reactscan.net/address/0x23d8de308079caeec9409d7a66f1e4b934d749d0
```

**② 一笔 `react()` 被成功调用的交易（Lasna）**

```
0x8a3ab28a4bc3113b0397522a960b8255fa27873a6575cfaa7b15429f9ef0257f
https://lasna.reactscan.net/tx/0x8a3ab28a4bc3113b0397522a960b8255fa27873a6575cfaa7b15429f9ef0257f
```

**③ 一句话说明订阅过滤条件**

> 订阅 Sepolia（chainId 11155111）上我部署的测试代币 `0xfb7553…80f5` 的 `Transfer(address indexed from, address indexed to, uint256 value)` 事件（topic0 `0xddf252ad…23b3ef`），topic1 / topic2 / topic3 全部用 `REACTIVE_IGNORE` 不限制；在 `react()` 里拆出 `value`，**只对单笔 ≥ 100 × 10^18（100 token）的转账计数并请求回调**，低于阈值的直接 `return`。

## 四项硬性要求逐条对应

| 要求 | 我怎么做的 | 链上/代码证据 |
|---|---|---|
| 构造函数完成订阅，用 `REACTIVE_IGNORE` 处理不关心的 topic | `CampReactive` 构造函数调用 `SYSTEM.subscribe(11155111, token, TRANSFER_TOPIC0, REACTIVE_IGNORE, REACTIVE_IGNORE, REACTIVE_IGNORE)` | 部署交易 `0xec3b40b4…177`；constructor 里六参数只有 chainId + 合约 + topic0 是具体的 |
| `react()` 带 `onlySystem` | `function react(LogRecord calldata log_) external onlySystem`，非系统合约调用直接 `NotAuthorized` | 模板源码 `starter/src/CampReactive.sol`；能被调用本身说明系统合约认可 |
| 设置次数上限，达到后自动退订 | 构造参数 `limit = 1`；`react()` 里 `remaining -= 1`，归零时 `SYSTEM.unsubscribe(...)` | `remaining() = 0`（见下），二次转账后**不再触发**，85 个区块过去 `lastReactBlock` 不变 |
| 部署时充值，避免欠款停摆 | 部署时 `--value 0.5ether`（lREACT），当前余额 `0.4812867 lREACT`，`debt = 0` | Reactscan 地址页可查余额与状态 `active` |

## 部署与触发参数

```bash
# 构造参数（limit=1，threshold=100e18）
ARGS=$(cast abi-encode "constructor(uint256,address,uint256,uint256,uint256,uint256)" \
  11155111 0xfb7553bbe806f3e13e712b8171666b6e9f6280f5 \
  0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef \
  11155111 100000000000000000000 1)

cast send --rpc-url https://lasna-rpc.rnk.dev/ --private-key $PRIVATE_KEY \
  --gas-limit 1500000 --gas-price 100000000000 --value 0.5ether \
  --create "$(forge inspect CampReactive bytecode)${ARGS#0x}"
# → 合约 0x23d8de308079caeec9409d7a66f1e4b934d749d0

cast send --rpc-url https://lasna-rpc.rnk.dev/ --private-key $PRIVATE_KEY \
  0x23d8de308079caeec9409d7a66f1e4b934d749d0 "setRecipient(address)" $WALLET_ADDR

cast send --rpc-url $SEPOLIA_RPC_URL --private-key $PRIVATE_KEY \
  0xfb7553bbe806f3e13e712b8171666b6e9f6280f5 \
  "transfer(address,uint256)" $WALLET_ADDR 500000000000000000000
# → 触发交易 0x2e2bafcbeb44e8f4f4dcb3c9064c748789deaa44cdbc6313df3bee3efaf34ebc
```

> 备注：`setRecipient` 填的是我自己的钱包地址（EOA，没有代码）。按官方文档，回调打到无代码地址会在收费前被拒，所以不产生无效扣费；这个任务只要求 `react()` 被调用，不要求回调落地。

## 核验输出（真实链上读取）

```
lastReactBlock : 8983443          ← 非 0，react() 真被系统合约调了
remaining      : 0                ← limit=1 已用完，自动 unsubscribe 已执行
debt           : 0                ← 没有欠款，未停摆
balance        : 0.4812867 lREACT ← 部署时充的 0.5 lREACT
```

**自动退订的反证**：之后又发了一笔 `500 → 600 token` 的转账（交易 `0xac516f12…c5984d17`），Lasna 区块从 8983443 涨到 8983528（85 个区块），`lastReactBlock` 仍是 **8983443**、`remaining` 仍是 **0**——订阅确实被取消，不是只是计数归零。

## 一句话结论

合约在构造函数里用 `REACTIVE_IGNORE` 订阅 Sepolia 测试币的 `Transfer`，`react()` 用 `onlySystem` 鉴权并只处理 ≥100 token 的转账，limit=1 用完后自动退订；部署充了 0.5 lREACT，`react()` 已被真实调用一次，二次转账不再触发。
