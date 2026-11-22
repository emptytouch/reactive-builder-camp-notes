# D2 任务提交件｜读懂费用：实测 Lasna 系统合约参数

执行时间：2026-11-22 23:03（GMT+8）｜RPC：`https://lasna-rpc.rnk.dev/`｜系统合约：`0x8888888888888888888888888888888888888888`

## 三条 cast 命令及真实输出

```bash
$ cast call 0x8888888888888888888888888888888888888888 "_gasPriceCoeffPer1000()" --rpc-url https://lasna-rpc.rnk.dev/ | cast to-dec
1000

$ cast call 0x8888888888888888888888888888888888888888 "_extraGas()" --rpc-url https://lasna-rpc.rnk.dev/ | cast to-dec
100000

$ cast call 0x8888888888888888888888888888888888888888 "_maxChargeGas()" --rpc-url https://lasna-rpc.rnk.dev/ | cast to-dec
50000
```

三个参数：**系数 1000（即 C = 1000/1000 = 1）**、**固定 gas 附加 100000**、**支付 gas 预算 50000**。注意第三个不进费用公式，只限制 `pay()` 能烧多少 gas。

## 1. 一次 `react()` 的费用怎么算

我的理解：`fee = p_base × C × (g_used + K)`。

- `p_base` 取 `tx.gasprice` 和 `block.basefee` 里**大的那个**，不是随便一个。
- `C = 1000 / 1000 = 1`，Lasna 现在没加价。
- `g_used` 是我这次 `react()` 真烧掉的 gas。
- `K = 100000` 是**白送的固定成本**——哪怕我什么都不做，账单也先挂 10 万 gas。

所以真正要命的不是公式，是那个 K：订阅越宽、触发越频繁，10 万 gas 就被乘得越多。粗算一下，gas price 100 gwei 时，一次简单 react() 约 0.015 lREACT。这就是为什么得给自动化设次数上限、处理完就退订。

## 2. balance / reserves / debt 分别是什么？为什么有一点 debt 就停

| 名词 | 我的理解 |
|---|---|
| balance | 合约自己钱包里的钱，归合约支配，但要主动交出去才能抵债 |
| reserves | 预付给系统合约（或回调代理）的信用额度，扣费时先花它 |
| debt | 已经欠下但还没结清的费用，记在系统合约账上 |

记账只有两条规则：**收费先吃 reserves，吃光了才欠债；存款先清债，剩下的才进 reserves**。所以一个合约不会同时有 reserves 和 debt，只有三种状态：有额度 / 归零 / 欠债。

「一点 debt 就停」的原因是：系统合约在 `trigger()` 里**最先查债务，在任何 gas 消耗之前**。只要 debt > 0，它就不调用你的 `react()`，代理也不投递你的回调。这不是执行失败报错，而是**根本不给你执行机会**，所以你在浏览器上只看到「没反应」，没有任何失败提示——静默停摆。

还有个坑：直接给合约转账只增加 balance，不会清债，钱会卡在合约里。要救回来得用 `depositTo(合约地址)`（推荐，直接进 reserves 先清债），或者合约自己暴露了 `coverDebt()` 才能用 balance 还。另外 `pay()` 是「发生收费时」被回调的，合约都停摆了就不会有收费，所以它救不了已经欠债的合约。

## 3. `react()` 回滚了还收费吗？对设计意味着什么

**收，而且全额收，不退。** 系统合约会捕获 revert、记一条 `ReactiveContractReverted` 事件，然后照常扣费。回调失败同理记 `CallbackFailure` 并收费。唯一例外是回调打到没有代码的地址，会在收费前被拒。

这直接改了我的写法：

- **过滤条件用 `return`，不要用 `revert`**。金额低于阈值、topic 不匹配、次数用完，都直接 `return`——不做事，但少烧 gas。
- 别把 revert 当流程控制；它不便宜，也不是「免费拒绝」。
- 把贵的判断放后面，先做最便宜的检查（topic0、opCode、阈值）。
- 两侧都要留钱：Reactive 侧用 REACT/lREACT，目标链用原生币；不然回调失败照样扣。
- 设 `remaining` 次数上限，处理完退订，别让固定 K 一直烧。
- 回调合约两道校验 `onlyServiceProvider` + `onlyCallbackSender` 必须写，别靠 revert 来「发现问题」——那也是要付钱的。

一句话：**在 Reactive 上，revert 不是错误处理的免费出口，而是一次全价收费的失败。**
