# D4 任务提交件｜三段链路 + 回调鉴权失败实验

执行时间：2026-11-22 23:29（GMT+8）｜测试网：源链与目标链均为 **Ethereum Sepolia（11155111）**，Reactive 侧 **Lasna（5318007）**，无主网混用。

## 一、三笔交易（本任务要交的证据）

| 环节 | 交易链接 |
|---|---|
| ① 源链事件：测试 ERC-20 `Transfer(from, to, 500e18)` | https://sepolia.etherscan.io/tx/0x8e024ae7b4ebf543fb5778b7e03d522e36dbb58e5f73400a34fb68f94b3f4034 |
| ② Reactive `react()` 被执行 | https://lasna.reactscan.net/tx/0xf00ae83fc2476d63a4b2cc6a48ae1ecc42aa8b6767982cf3d7e97e42a361abea |
| ③ 目标链 Callback 被调用，emit `Handled` | https://sepolia.etherscan.io/tx/0x1b0df3555ddf2c042280015f24051f181bcb0f194176d34899e9d11d53177671 |

合约地址：Reactive `0x789312d59c59fab7d3dc5914041ea6308828061f`（Lasna）、Callback `0x3b1f497e6b051a3b391c82d32d0a0f25e1ed6f74`（Sepolia）。

链路：`TestToken.transfer()` → `SYSTEM` 订阅匹配 → `react(LogRecord)` 解出 `value` 判阈值 → `requestCallbackV_1_0` → `Callback Proxy 0xAa14d9Cd0f788Ca23d11805986C1d8bd035D23b4` → `CampCallback.onEvent()` → `Handled`。

## 二、失败实验：钱包直接调用 Callback，被拒绝

**交易链接（链上 status = 0）**：

```
https://sepolia.etherscan.io/tx/0x5b94d01fb14a7c395b700e95a2a106e737763bbcade901eeaea3e6cef4c4cc6b
```

做法：用我自己的钱包（`0x9750…988`）直接调用 `CampCallback.onEvent(address,uint256,uint256,uint256)`，第一参数也填我自己的地址，模拟「有人冒充 Reactive 合约发回调」。

执行前 `cast call` 复现的报错：

```
Error: execution reverted, data: "0xc55ddc97…"
```

`0xc55ddc97` = `NotAuthorized(address sender_, address provider_)`，解码后：

```
sender_   = 0x9750Af96716784390A76420312D57fdadEB4B988   ← 我的钱包
provider_ = 0xAa14d9Cd0f788Ca23d11805986C1d8bd035D23b4   ← Sepolia Callback Proxy
```

**一句话解释为什么被拒**：`onEvent` 带 `onlyServiceProvider`，它要求 `msg.sender` 必须是本链的 Callback Proxy `0xAa14…D23b4`；我的钱包是普通 EOA，不是代理，所以在第一道校验就被 `NotAuthorized` 拦下，状态 0、没有执行函数体。即使绕过第一道，`onlyCallbackSender` 还会检查 payload 注入的 reactive 地址是否等于构造时绑定的 `0x7893…061f`，冒充者同样过不了。

## 三、要求逐条对应

| 要求 | 满足情况 |
|---|---|
| 源链和目标链都用测试网，不和主网混用 | 均是 Ethereum Sepolia（11155111）；Reactive 侧 Lasna（5318007） |
| 可选 Base / Unichain Sepolia（不强制） | 本任务用 Sepolia 同链作源/目标，官方模板默认且已实测；赛道 1 不要求两条链 |
| Callback 继承 `AbstractCallback`，首参 `address`，两道校验 | `CampCallback is AbstractCallback`；`onEvent(address reactive_, ...)` 带 `onlyServiceProvider` + `onlyCallbackSender(reactive_)` |
| Callback Proxy 以 Origins & Destinations 表格为准 | 用的是表格中 Ethereum Sepolia 的 `0xAa14d9Cd0f788Ca23d11805986C1d8bd035D23b4`，不是旧地址 |
| 失败实验 | 上面链上 status=0 的交易 + 解码后的 `NotAuthorized` 报错 |

补充证据：本地 `forge test -vv` 8 个测试全过，其中 `test_callbackRejectsWrongProxy`、`test_callbackRejectsWrongSender` 正好覆盖这两道校验。

## 四、一句话结论

三段链路在测试网上真实跑通（三笔交易可核），且钱包直接调回调被 `onlyServiceProvider` 拒绝（失败交易可核）——说明回调确实只能由 Callback Proxy 投递，链上证据与代码设计一致。
