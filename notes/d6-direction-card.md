# D6｜黑客松项目方向卡（第一版，第 2 周以最终版为准）

> 状态：单人｜赛道 2｜2026-11-22 定稿第一版
> 对接赛事：Reactive Omni Buildathon（https://dorahacks.io/hackathon/reactive-omni-buildathon/detail ）

## 一句话

**监听 Sepolia 上我的测试代币 `Transfer`（`topic1 = 我的热钱包`）→ `react()` 判断单笔或 1 小时内累计转出超过阈值 → 在 Sepolia 回调 Guardian 合约，把事先授权额度内的剩余代币自动转到我预设的冷钱包，并 emit 事件供告警。**

## 赛道

**赛道 2｜Set It and Forget It（个人自动化；旧大纲称 Personal Use Cases）**

选它的理由，说人话：

- 这件事的价值在「**我不用盯着链**」，不在「快 1 秒」。钱包被异常转出后，人肉发现再操作通常要几分钟；链上自动响应是秒级，但对这个场景够用，不需要赛道 1 那种低延迟套利。
- 不碰赛道 3：AI **不在运行时做决定**。阈值、白名单都是确定的规则，引入模型只会增加不确定性和攻击面。
- 赛道 2 明确欢迎零合约经验选手，评审看的是「是否真的解决一个个人会遇到的麻烦」，而不是机制多复杂——这正好是我能讲清楚的。

## 目标用户和真实场景

**谁会用**：跟我一样用热钱包做日常链上操作（领测试币、交互 dApp、小额转账）的普通用户。热钱包私钥在浏览器/电脑上，一旦中毒或误签，资产会被转走。

**现在他们怎么解决**：

1. 大额资产放冷钱包/硬件钱包，热钱包只放少量——但热钱包里的钱被转走仍然是真实损失，且不会有人替你自动反应。
2. 用链下监控 bot / 第三方告警服务——要跑服务器、要盯着通知、要手动去链上补救；掉线就漏，私钥还常放在 bot 里。
3. 多签——安全但日常操作太重，不适合小额高频的普通用户。

**Reactive 让它简单在哪**：监听 + 判断 + 发交易全在合约里，用户不跑服务器、不托管私钥、链上可核验。注意**不能夸大**：它不能撤销已经发生的转账，也不能保证抢在攻击者前面——定位是「事先授权、额度受限的自动止损 + 告警」，不是「防盗保险」。这条边界场景手册写得很清楚，我会写进 README。

## 最小 Demo（第 2 周结束前必须跑通的那条链路）

```
源链 Sepolia（11155111）
  TestToken 0xfb7553…80f5 emit Transfer(from=我的热钱包, to, value)
  → 订阅 chainId + token 地址 + topic0 + topic1 = uint256(uint160(热钱包))
       ↓ ¹
Lasna CampReactive（新部署）
  react(LogRecord) onlySystem
    topic1 精确匹配热钱包；opCode == 3；value = abi.decode(data,(uint256))
    value ≥ 阈值 → 请求回调；低于阈值 return
       ↓ ² requestCallbackV_1_0，payload 首参 address(0) 占位
Sepolia Guardian 回调合约
  onlyServiceProvider + onlyCallbackSender 两道校验
  token.transfer(coldWallet, min(remaining, CAP))  ← 只能转预设冷钱包、单次与累计都限额
  emit GuardianSwept(...)  ← 链下脚本监听推 Telegram
```

- **第 2 周验收标准**：在 Sepolia 拿到三笔真实交易（异常转账 → Lasna `react()` → Guardian 执行），`lastReactBlock` 非 0、`debt = 0`；展示「正常小额转账不触发」和「超过阈值才触发」两个对照。
- **已有基础**：D3/D4 已跑通 ERC-20 Transfer → react() → 回调的三段链路（证据见 [d3-first-deploy-task.md](d3-first-deploy-task.md) 与 [d4-callback-task.md](d4-callback-task.md)）；D5 已跑通 Cron 限额自动退订（[d5-cron-task.md](d5-cron-task.md)）。参赛作品会是在这些基础之上的**新合约 + 新逻辑**，旧练习只当依赖。
- **交付物**：公开仓库、合约地址、三笔哈希、<3 分钟 Demo 视频、README 写清如何复现与 AI 使用范围。

## 单人还是组队

**单人**。我会做合约、部署、测试、README 和 Demo；如果第 2 周时间紧，只在社群找 1 人补前端告警页/视频剪辑，不核心依赖队友。分工写得清楚，避免出现「等队友」的阻塞。

## 边界与风险（提前写，免得 Demo Day 被问住）

- 不能撤销已发生转账，也不保证比攻击者先动手；只操作事先授权、额度受限的测试资产。
- Guardian 合约能动你的币 → 目标地址硬编码为冷钱包，单次/累计额度写死，先 approve 固定额度。
- 每次 `react()` 都收费且有 10 万 gas 底噪 → 阈值过滤用 `return`；同一地址短时间连续事件做冷却（用 `block.timestamp`）；余额耗尽/欠费会静默停摆，Demo 里展示 `debt` 监控与 `depositTo` 充值。
- 当前演示在测试网；主网部署不在本次范围内。
