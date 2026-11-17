# D2｜源链、目标链与资金模型（中文精讲）

> 对应官方原文：<https://dev.reactive.network/origins-and-destinations>、<https://dev.reactive.network/economy>
> 今天要搞懂两件事：**① 事件从哪条链来、回调打到哪条链；② 这套东西怎么收费、什么时候会悄悄停摆。**

---

## 一、先记四个词（英文看不懂就先背这张表）

| 英文 | 中文 | 一句话 |
|---|---|---|
| Origin | 源链 | 出事的那条链，Reactive 从它读事件日志 |
| Destination | 目标链 | 回调落地的那条链，状态在这改 |
| Callback Proxy | 回调代理 | 目标链上的一个合约，回调交易必须经过它，你的合约靠它验明正身 |
| Reactive Signer | Reactive 签名者 | Reactive 侧真正往目标链发交易的那个地址 |
| balance | 余额 | 合约自己账上的钱 |
| reserves | 预留 / 预付额度 | 存在系统合约里的信用额度，用来抵扣费用 |
| debt | 欠款 | 已经欠系统合约、还没付的费用 |
| revert | 回滚 | 执行失败。**注意：失败照样收费** |
| depositTo | 存入 | 通过系统合约给某个合约记账（先进 reserves） |

---

## 二、源链 / 目标链：谁都能当，但主网和测试网不能混

一个 Reactive 合约可以**同时订阅多条源链**的事件，也可以**往多条目标链**发回调，甚至能在 Solidity 里按条件挑目标链。所以这不是「一对一管道」，更像「一个监听中心 + 多个出口」。

**铁律：主网配主网，测试网配测试网。** 源链是主网，目标链也必须是主网；混用直接不成立。

### 主网（Origin / Destination 都支持）

| 链 | chainId | Callback Proxy 地址 |
|---|---|---|
| Ethereum | 1 | `0xFCE1F192F489c988392aBC16E91e359d540d94Fe` |
| Arbitrum | 42161 | `0xaee8d2deBaB5bE353bfec167C58a549923892e11` |
| Base | 8453 | `0xCc780DdE33f56f55dE778D42c930959374a4a068` |
| BSC | 56 | `0xfc41eE9d3A836b41399BEcce83b06B3e5650f3BB` |
| Avalanche | 43114 | `0x8bdfC7e564B90b31c49Cf1079d4c44400F4e14B4` |
| Unichain | 130 | `0x754120EC7a6eF81E4c5c835AA603C63Da64693E6` |
| HyperEVM | 999 | `0x754120EC7a6eF81E4c5c835AA603C63Da64693E6` |
| Robinhood | 4663 | `0x476F44b11F5C942c58f4c8E3d0300E5710743585` |
| Reactive 主网 | 1597 | `0x8888…8888` |

### 测试网（我们这两周只在测试网玩）

| 链 | chainId | Callback Proxy 地址 |
|---|---|---|
| **Reactive Lasna**（Reactive 侧固定用它） | 5318007 | `0x8888…8888`，RPC `https://lasna-rpc.rnk.dev/` |
| Ethereum Sepolia | 11155111 | `0xAa14d9Cd0f788Ca23d11805986C1d8bd035D23b4` |
| Base Sepolia | 84532 | `0x476F44b11F5C942c58f4c8E3d0300E5710743585` |
| Unichain Sepolia | 1301 | `0x775c9B990ADe401060B78D0e98684eEf8d2466C1` |

> 从这张表能直接读出两件事：① 同一个代理地址会出现在不同链上（如 `0x476F…3585` 既是主网 Robinhood 也是 Base Sepolia），所以**校验回调时绝不能只比对地址，要连 chainId 一起认**；② 我们练习的目标链就是 Sepolia / Base Sepolia / Unichain Sepolia 三选一，Reactive 侧统一 Lasna。

**回调怎么验真**：目标链上的合约检查两件事——`msg.sender` 是你那条链的 Callback Proxy，且原始发送方是 Reactive Signer。这也是为什么 D4 要求 `onlyServiceProvider` + `onlyCallbackSender` 两道校验，而不是自己糊一个 `require(msg.sender == owner)`。

---

## 三、三个数字：balance / reserves / debt

这是今天真正的核心。三者的关系只有两条记账规则，记住就全通了：

1. **收费先扣 reserves，扣完了才产生 debt。**（charges consume reserves before debt accrues）
2. **存款先清 debt，剩下的才进 reserves。**（deposits clear debt before adding reserves）

推论一：**一个合约不可能同时有 reserves 和 debt**，只可能是「有 reserves / 零 / 有 debt」三态。
推论二：**balance 是另一套体系**。别人直接往合约转钱只增加 balance，既不会变成 reserves，也不会替你还债。

| | 存在哪 | 谁控制 | 干什么用 |
|---|---|---|---|
| balance | 合约自己账上 | 合约（要主动交出去） | 收费时可以拿出来付 |
| reserves | 系统合约 / 回调代理里 | 系统 | 自动抵扣，最先被消耗 |
| debt | 系统合约 / 回调代理记账 | 系统 | 欠款，必须清零才恢复 |

### 「有 debt 就停」到底有多狠

只要 `debt > 0`，**哪怕只欠一点点**：系统合约不再调用你的 `react()`，回调代理也不再往你的回调合约投递。而且**债务检查排在所有 gas 消耗之前**——所以负债合约是**静默停摆**，不会给你报错、不会在日志里炸，你只会发现「怎么没反应了」。

状态在 Reactscan 上看：`active` / `inactive`。债务从 0 变正会发 `BlacklistContract` 事件（拉黑），清零后发 `WhitelistContract`（恢复）。注意：**在同一笔交易里被 `pay()` 结清的合约，根本不会被拉黑**。

---

## 四、费用公式：一行，但要能读懂

Reactive 侧和 callback 侧**用同一个公式**：

```
fee = p_base · C · (g_used + K)
```

| 符号 | 含义 | 默认值 |
|---|---|---|
| `p_base` | `tx.gasprice` 和 `block.basefee` **取较大者** | 随网络 |
| `C` | 定价系数，读 `_gasPriceCoeffPer1000()` 再 **除以 1000** | Reactive 侧 1000（即 1 倍） |
| `g_used` | 实际消耗的 gas | 看代码 |
| `K` | 固定附加费，读 `_extraGas()` | Reactive 侧 100,000 gas |

**callback 侧没有默认值**，每个代理部署时自己配，必须去代理合约上查。所以别背数字，背命令：

```bash
# Reactive 侧
cast call $SYSTEM_CONTRACT_ADDR "_gasPriceCoeffPer1000()" --rpc-url $REACTIVE_RPC | cast to-dec
cast call $SYSTEM_CONTRACT_ADDR "_extraGas()"              --rpc-url $REACTIVE_RPC | cast to-dec
# 目标链 callback 侧（换成代理地址和该链 RPC）
cast call $CALLBACK_PROXY_ADDR "_gasPriceCoeffPer1000()" --rpc-url $DESTINATION_RPC | cast to-dec
cast call $CALLBACK_PROXY_ADDR "_extraGas()"             --rpc-url $DESTINATION_RPC | cast to-dec
```

**这个公式最坑的一点**：`K` 是固定 100k gas 起的固定开销。所以一次 `react()` 哪怕只干了一点点活，账单里也永远挂着一笔 10 万 gas 的底噪。**订阅越宽 → 触发越频繁 → 这笔底噪被乘上几千次。** 这就是「订阅通配符要加 `_limit`」的经济学理由，不是代码风格问题。

### 两条 gas 上限

- Reactive 交易 gas 上限 **900,000**
- Callback gas limit **最低 100,000**，低于这个数的请求会被**直接忽略**（连失败都算不上，就是没发生）
- 而且请求的 gas **不会全到目标**：代理要先扣 `_extraGas` 和 `_maxChargeGas`，剩下的才转发；剩不够就 `InsufficientGas` 失败

---

## 五、充钱的两条路（这里最容易白扔钱）

| 方式 | 效果 | 什么时候用 |
|---|---|---|
| `depositTo(目标合约)` | 进**系统合约的 reserves**，产生债务前自动抵扣 | 推荐默认选它，尤其是给别人的合约充值 |
| 直接转账给合约 | 只加 **balance**，不清债、不产生 reserves | 合约自己有 `pay()` / `coverDebt()` 时才行 |

```bash
# ✅ 推荐：明确指定目标合约
cast send --rpc-url $REACTIVE_RPC --private-key $PK $SYSTEM_CONTRACT_ADDR \
  "depositTo(address)" $REACTIVE_CONTRACT_ADDR --value 1ether
```

三个必须知道的坑：

1. **直接往系统合约转 REACT，钱会记到你自己地址上**，不是给任何合约充值——「给合约充值」充进了一个永远不执行任何东西的地址。
2. **直接转账给合约，如果合约没暴露 `coverDebt()`，这笔钱就困住了**：它加的是 balance，而 balance 只有合约自己主动交给系统合约才能抵债。不确定就用 `depositTo()`。
3. **`pay()` 有 gas 预算限制（`_maxChargeGas`）**，自定义支付逻辑写太胖会 `PaymentFailure` 且债务还在。所以 `pay()` 要写薄。

标准实现已经帮你做好了：`AbstractReactive` 和 `AbstractCallback` 都继承 `AbstractPayer`，所以 `pay()` 自动结算、`_coverDebt()` 内部可用。你要做的只是决定**要不要把 `coverDebt()` 包成 external**（面向第三方开放就包，并加 `onlyOwner` 或干脆全公开）。

### 失败不退款

- `react()` revert → 捕获、记 `ReactiveContractReverted` 事件、**全额收费**
- callback 失败 → 记 `CallbackFailure` 事件、**全额收费**
- 两者都**不退**。一个每次都 revert 的合约，钱照样被烧光。
- 唯一例外：回调打向**没有代码的地址**，会在收费前被拒。

**结论**：宁可让 `react()` 里条件判断提前 return（不做事但便宜），也不要让它 revert。

### 两侧币种不同

- Reactive 侧用 **REACT**（`--value 1ether` 指 REACT）
- Callback 侧用**目标链原生币**（ETH / BNB / …），**不是 REACT**

练习时最常见的翻车就是：Lasna 上有 REACT，但目标链上的回调合约没钱，回调永远失败还照收费。

---

## 六、今天该动手的 4 条命令

挑一个你自己部署过的合约（还没有就等 D3），把三个数查出来写进笔记：

```bash
# 1. 合约余额
cast balance $REACTIVE_CONTRACT_ADDR --rpc-url $REACTIVE_RPC
# 2. 欠款（>0 就是已停摆）
cast call $SYSTEM_CONTRACT_ADDR "debt(address)" $REACTIVE_CONTRACT_ADDR --rpc-url $REACTIVE_RPC | cast to-dec
# 3. 预留额度
cast call $SYSTEM_CONTRACT_ADDR "reserves(address)" $REACTIVE_CONTRACT_ADDR --rpc-url $REACTIVE_RPC | cast to-dec
# 4. 它欠费给谁（Reactive 合约返回系统合约；回调合约返回回调代理）
cast call $REACTIVE_CONTRACT_ADDR "_SERVICE_PROVIDER()" --rpc-url $REACTIVE_RPC
```

> 小坑：`debts(address)` 是老别名，新代码用 `debt(address)`（属于 `IPayable`）。
> 另一个好用命令：`getCallbacks(你的reactive交易哈希)` 能反查这笔交易请求了哪些回调、目标交易哈希、错误数据。**返回空数组 = 还没记录到，不等于失败**（validator 是投递后才写记录）。

---

## 七、自测（先别翻答案，能答上来就算过）

1. 我的 reactive 合约 balance 有 5 REACT，但 debt 是 0.001 REACT，`react()` 还会被调用吗？为什么？该用什么命令救它？
2. 我直接往系统合约转了 1 REACT，为什么我的合约还是 inactive？
3. 一次 `react()` 里我什么都不做就 revert，会不会省钱？

<details>
<summary>答案</summary>

1. **不会**。debt > 0 就停摆，跟 balance 多少无关——balance 得由合约主动交给系统合约才能抵债。救法：`depositTo(合约地址)` 打钱进去（先清债务，剩下的进 reserves），或调合约的 `coverDebt()`。
2. 因为直接转给系统合约，钱记在**发送者自己地址**上，没给任何合约记账。必须用 `depositTo(目标合约)`。
3. **不省**。revert 会被捕获、记事件、**全额收费**。想省钱应该提前 `return`，不做事但少烧 gas。

</details>

---

## 八、一句话总结（写进打卡的那种）

> Reactive 的费用分两侧：Reactive 侧扣 REACT，目标链侧扣原生币。每个合约有三态账——reserves（预付信用）、debt（欠款）、balance（自有余额），收费先吃 reserves，吃完才欠债，**一有债就静默停摆**，所以要么用 `depositTo` 喂 reserves，要么让合约暴露 `coverDebt` 自己还。公式是 `fee = p_base·C·(g_used + K)`，那个固定 K（默认 10 万 gas）意味着**每次触发都有底噪成本**，订阅越宽越烧钱——这就是必须给自动化设次数上限的经济原因。
