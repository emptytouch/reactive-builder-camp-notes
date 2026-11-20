# D5｜定时任务与动态订阅（中文精讲）

> 今日任务：**定时或动态订阅**（课程第 1 周 D5，11/20）
> 官方原文：<https://dev.reactive.network/reactive-contracts> → Subscribing To CRON Events / Changing Subscriptions At Runtime
> 场景手册：场景 5｜定时任务：定投、巡检、到点执行

一句话：**让合约按“每 N 个区块”自己醒来，并且能在运行时改订阅——但每个 tick 都要付钱。**

---

## 一、术语对照

| 英文 | 中文 | 一句话 |
|---|---|---|
| Cron event | 定时事件 | 由网络的 legacy 系统合约按区块间隔发出 |
| tick | 一次触发 | 每到一个间隔就 wake up 一次，`react()` 被调一次 |
| legacy system contract | 旧系统合约 | `0x0000000000000000000000000000000000fffFfF`，**只用于 Cron 订阅** |
| block interval | 区块间隔 | 不是墙钟时间，约等于的秒数只是估算 |
| dynamic subscription | 动态订阅 | 在 `react()` 或外部函数里 `subscribe` / `unsubscribe` |
| pause / resume | 暂停 / 恢复 | 用退订和重新订阅实现 |

> ⚠️ 复习 D2 的坑：系统合约是 `0x8888…8888`，**只有 Cron 事件来自 `0x…fffFfF`**。两者别混。Cron 订阅里 `contract_` 位置填的是 `0x…fffFfF`，但订阅/退订动作依然通过 `SYSTEM`（`0x8888…8888`）调用。

---

## 二、Cron 的五个固定 topic0（背表，别自己算）

| 事件 | 间隔 | 大约时间 | topic 0 |
|---|---|---|---|
| `Cron1` | 每 1 个区块 | ~1 秒 | `0xf02d6ea5c22a71cffe930a4523fcb4f129be6c804db50e4202fb4e0b07ccb514` |
| `Cron10` | 每 10 个区块 | ~10 秒 | `0x04463f7c1651e6b9774d7f85c85bb94654e3c46ca79b0c16fb16d4183307b687` |
| `Cron100` | 每 100 个区块 | ~2 分钟 | `0xb49937fb8970e19fd46d48f7e3fb00d659deac0347f79cd7cb542f0fc1503c70` |
| `Cron1000` | 每 1000 个区块 | ~17 分钟 | `0xe20b31294d84c3661ddc8f423abb9c70310d0cf172aa2714ead78029b325e3f4` |
| `Cron10000` | 每 10000 个区块 | ~3 小时 | `0xd214e1d84db704ed42d37f538ea9bf71e44ba28bc1cc088b2f5deca654677a56` |

这些是**协议硬编码常量**，不是事件签名 keccak 出来的——别拿名字去 `cast keccak`，算出来一定不对。

订阅方式与普通事件几乎一样，只有 `contract_` 换成 legacy 地址：

```solidity
address internal constant LEGACY_SYSTEM_ADDR = 0x0000000000000000000000000000000000fffFfF;
uint256 internal constant CRON100_TOPIC_0 =
    0xb49937fb8970e19fd46d48f7e3fb00d659deac0347f79cd7cb542f0fc1503c70;

function _subscribeToCron() internal {
    SYSTEM.subscribe(
        block.chainid,          // 注意：这里用当前链
        LEGACY_SYSTEM_ADDR,
        CRON100_TOPIC_0,
        REACTIVE_IGNORE,
        REACTIVE_IGNORE,
        REACTIVE_IGNORE
    );
}
```

### Cron 事件的载荷

每个 Cron 事件带一个参数 `number`（触发时的区块号），**不是 indexed**，所以它在 `log_.data` 里，是一个 `uint256`。但官方文档说得很直接：在 `react()` 里通常直接读 `block.number` 就行，不用管载荷。

### 路由：多个订阅怎么区分

Cron 的 tick 和别的事件走**同一个 `react()`**，靠 `topic0` 分辨：

```solidity
function react(LogRecord calldata log_) external onlySystem {
    if (log_.topic0 == CRON100_TOPIC_0) {
        _onTick();          // 定时逻辑
    } else if (log_.topic0 == TRANSFER_TOPIC0) {
        _onTransfer(log_);
    }
}
```

---

## 三、最容易被忽略的四个坑

1. **重叠订阅会重复收费。** 一次 `cron()` 调用会发出当前区块号能整除的所有间隔事件。同时订阅 `Cron1` 和 `Cron10`，每第 10 个区块会收到**两笔独立的 reactive 交易**，付两份钱。
2. **每个 tick 都收费，哪怕 `react()` 里直接 `return`。** 所以选**能接受的最长间隔**，并且一定要有暂停开关（退订）。
3. **Cron 是区块节奏，不是墙钟定时。** 官方写的“~2 分钟”只是估算；Base Sepolia 现在 200ms 出块，Cron100 在那里只有约 20 秒，课程明确要求时间逻辑用**时间戳**而不是区块数。要严格墙钟调度，得自己用时间戳判断。
4. **暂停后再恢复，先清债。** `unsubscribe` → 过一阵 → 再 `subscribe`，如果中间欠费，合约收不到任何交易，重新订阅也会静默失败。官方示例是先 `_coverDebt()` 再 `_subscribe()`。

---

## 四、动态订阅：运行时改“听什么”

订阅不只限于构造函数。`react()` 里、外部函数里都能改：

```solidity
function _subscribe() internal {
    SYSTEM.subscribe(
        _originChainId, _originContract, _topic0,
        REACTIVE_IGNORE, REACTIVE_IGNORE, REACTIVE_IGNORE
    );
}

function _unsubscribe() internal {
    SYSTEM.unsubscribe(
        _originChainId, _originContract, _topic0,
        REACTIVE_IGNORE, REACTIVE_IGNORE, REACTIVE_IGNORE
    );
}
```

**六项参数必须完全一致**——`unsubscribe()` 只移除六项全匹配的订阅，任何漂移都会静默失败，老订阅继续扣钱。所以官方建议把两边都包成 helper，参数只写一处。

官方给的“限额 + 暂停/恢复”范式（照抄思路）：

```solidity
function updateLimit(uint256 limit_) external payable onlyOwner {
    _limit = limit_;

    if (limit_ == 0) {
        _unsubscribe();            // limit=0 当作暂停
    } else {
        _coverDebt();              // 先清债，否则活不过来
        _subscribe();
    }
}
```

> 注意这是 `external payable`：调用时顺便打钱进去清债，避免“恢复订阅但仍欠费”的死状态。

---

## 五、今天的任务怎么交（两个选项，二选一）

平台任务名：**定时或动态订阅**。两种都算完成，我建议做 ①，因为简单可验证。

### 选项 ① Cron 定时触发（推荐）

最小做法：在现有 `CampReactive` 的构造函数里**追加一条 Cron100 订阅**，并在 `react()` 里加分支：

```solidity
// 构造函数末尾追加
_subscribeToCron();

// react() 开头加分支
if (log_.topic0 == CRON100_TOPIC_0) {
    ticks += 1;
    emit Tick(block.number, ticks);
    return;                       // 不做事也要 return，别 revert
}
```

部署后等 2–4 分钟，查 `ticks()` 是否 > 0、`lastReactBlock` 是否更新、`debt` 是否为 0。把 `Tick` 事件和那笔 Lasna 交易哈希贴进笔记即可。

> 省钱版：用 `Cron1000`（~17 分钟）验证更慢但便宜；测试网练习用 Cron100 足够。

### 选项 ② 动态订阅：达到 N 次自动退订、可恢复

复用 D3 已有的 `remaining` 机制：在 `react()` 里处理完第 N 次后 `SYSTEM.unsubscribe(...)` 退订；再提供一个 `onlyOwner` 的 `resume()`，内部先 `_coverDebt()` 再 `_subscribe()`。用 `cast send` 调一次 `resume()`，观察订阅恢复、事件再次触发。

> 两条路都能写成一篇合格笔记。**别两个都部署**——两份订阅会并行扣费，纯烧测试币。

---

## 六、自测

1. 你同时订阅了 Cron1 和 Cron10，10 个区块内 `react()` 被调几次？付几次钱？
2. `unsubscribe()` 的六个参数有一个写错了，会发生什么？怎么发现？
3. 在 Base Sepolia 上用 `Cron100` 做“每 2 分钟执行一次”，靠谱吗？

<details>
<summary>答案</summary>

1. 不是 10 次，是 **11 次**，付 **11 份**钱。Cron1 每个区块都触发（10 次），Cron10 在第 10 个区块**额外**再触发一笔——一次 `cron()` 会发出当前区块号能整除的所有间隔事件，两个订阅各自独立产生一笔 reactive 交易。
2. 订阅不会被移除，会继续扣费，而且**不报错**。发现方式：看 `react()` 还在被不该触发的事件调用，或用区块浏览器观察系统合约的 `Subscribed` 事件仍存在。所以六个参数要包 helper 统一写。
3. 不靠谱。Base Sepolia 200ms 出块，Cron100 只有约 20 秒；而且 Cron 按区块不按墙钟。正确做法是订阅后用 `block.timestamp` 自己判断间隔，或者选更长的 Cron 间隔。

</details>
