# Reactive Builder Camp 课程全景（学习内容 + 任务 + 评分）

> 来源：<https://web3career.build/zh/programs/reactive-builder-camp?tab=learning>（2026-11-17 抓取）
> 主办：ETHPanda × LXDAO（Web3 Career Build）× Reactive Network｜共学 11/16–11/29，冲刺至 12/05

## 0. 一句话

Reactive Network 让智能合约自己「动起来」：合约直接监听任意链上的事件，满足条件后自动在另一条链上发交易——把「盯盘 + 下单」写进合约，不用自己跑机器人、租服务器、24 小时盯链。

```
源链（Origin）            Reactive Network             目标链（Destination）
合约 emit 事件  ──订阅匹配──▶  react(LogRecord)  ──回调请求──▶  Callback Proxy ──▶ 你的合约函数
```

它解决的问题：价格到了要有人去卖，跨链价差出现要有人去搬，钱包被盗要有人第一时间转走资产。过去这些都得自己写 bot。

## 1. 对接的黑客松：Reactive Omni Buildathon（总奖金 $5,500，以 REACT 支付）

| 赛道 | 方向 | 奖金 |
|---|---|---|
| 赛道 1｜Every Second Counts | 对速度敏感的 DeFi：跨链套利、止损止盈、金库再平衡、流动性迁移 | 一等奖 $2,000 + 第二名 $250 |
| 赛道 2｜Set It and Forget It | 个人自动化：资产追踪、LP 到价提醒、自定义再平衡，零合约经验也能做 | 一等奖 $2,000 + 第二名 $250 |
| 赛道 3｜Reactive Meets AI | AI 参与系统运行：解读事件、生成决策、由链上事件触发执行 | 一等奖 $1,000 |

规则以 [DoraHacks 官方页](https://dorahacks.io/hackathon/reactive-omni-buildathon/detail) 与[赛道页](https://dorahacks.io/hackathon/reactive-omni-buildathon/tracks)为准。提交仓库须公开开源，作品须在活动期间构建（旧成果只能当依赖，不能直接拿既有项目参赛）。

## 2. 时间轴

| 阶段 | 时间 | 做什么 |
|---|---|---|
| 准备 | 报名后 – 11/15 | 测试钱包、Foundry、领测试币、建 GitHub 学习仓库；可先 clone 模板跑通测试 |
| 第 1 周 | 11/16 – 11/22 | 学会写跨链自动化，周末交项目方向卡 |
| 第 2 周 | 11/23 – 11/29 | 做自己的黑客松项目，11/29 Demo Day |
| 冲刺 | 11/30 – 12/05 | 按 Demo Day 反馈打磨，DoraHacks 正式提交（官方截止北京时间 12/06 04:00） |

每天约 1–1.5 小时。不设强制出勤，不会被移出群，所有任务在 12/05 23:59（北京时间）前补交都算。

## 3. 每日学习内容与任务（D1–D14）

### 第 0 周｜开营前准备（环境）

1. **测试专用钱包**：新建只用于本期的钱包（MetaMask 新账户），不要用主钱包。
2. **Foundry**：确认 `forge --version`、`cast --version` 有输出。
3. **测试币**：领 Sepolia ETH → 转到 Sepolia 上的 faucet 合约换 Lasna 的 lREACT（**1 ETH → 100 lREACT，单笔别超 5 ETH，超出部分丢失**，文档示例 0.1 ETH）。
4. **GitHub 学习仓库**：公开仓库，至少含 `README.md` + `notes/`，每天的打卡和代码都放这里。
5. **AI 编程工具（可选）**：Claude Code / Codex / Cursor，先把「给 AI 的上下文」存好。

### 第 1 周｜跑通 Reactive

| 日 | 学习目标 | 材料 | 任务 |
|---|---|---|---|
| **D1 11/16** 为什么需要 Reactive · 开营 | 说清它解决的问题（不跑 bot / Keeper，由网络监听并代发交易）；知道适合与不适合做什么 | [Reactive Contracts](https://dev.reactive.network/reactive-contracts)、[Reactive Circuit](https://blog.reactive.network/reactive-circuit-incentive-program-for-working-integrations/) | 用例拆解卡；Session 1 |
| **D2 11/17** 源链、目标链与资金模型 | 哪些链能做源/目标链、主网与测试网不能混用；`balance` / `reserves` / `debt` 三个数——**有欠款就停止执行**；`react()` 回滚照样收费 | [Origins & Destinations](https://dev.reactive.network/origins-and-destinations)、[Economy](https://dev.reactive.network/economy) | 读懂费用（平台任务：**用 cast 查询 Lasna 系统合约参数**，+60） |
| **D3 11/18** 场景 1 监控：第一个合约 | 构造函数里订阅；通配符与 `REACTIVE_IGNORE`；`react()` 加 `onlySystem`、按 opCode 解码；设次数上限；部署到 Lasna 并充值，浏览器上看到 `react()` 被调用 | [Subscriptions / Reacting To Events](https://dev.reactive.network/reactive-contracts)、omni 分支 [Xfers.sol](https://github.com/Reactive-Network/reactive-smart-contract-demos/tree/omni/src/micro-demos/Xfers.sol)、场景手册场景 1 | 第一次部署 |
| **D4 11/19** 场景 2 跨链回调 | 把「看到事件」变成「在另一条链做事」。Callback 合约继承 `AbstractCallback`，**第一个参数必须是 address**（代理注入），加 `onlyServiceProvider` + `onlyCallbackSender` 两道校验；回调合约也要有余额 | [Callback Contracts](https://dev.reactive.network/callback-contracts)、[Requesting Callbacks](https://dev.reactive.network/reactive-contracts)、omni 分支 CallbackSender / CallbackRecipient | **跨链回调三步走（必做）**；20:00 技术 Q&A |
| **D5 11/20** 场景 5 定时与动态订阅 | 订阅 Cron 事件（仍由 `0x…fffFfF` 发出）实现「每 N 个区块执行」；运行时增删订阅；给自动化加次数与额度上限 | [Subscribing To CRON / Changing Subscriptions At Runtime](https://dev.reactive.network/reactive-contracts)、omni 分支 Cron / Xfers | 定时或动态订阅 |
| **D6 11/21** 选题 | 把想法压成一句话：监听什么事件、满足什么条件、在哪条链做什么；判断是否真需要 Reactive | 场景手册「怎么挑」、三个赛道说明 | 项目方向卡 + DoraHacks 报名（可组队） |
| **D7 11/22** 缓冲日 | 补交本周任务 | — | 打卡写本周复盘（最卡的一步 + 怎么解决）+ 本周反馈 |

练习链：Reactive 侧统一 **Lasna**；目标链可选 **Sepolia / Unichain Sepolia / Base Sepolia**（赛道 1 不要求同时用两条链）。

### 第 2 周｜做你的黑客松项目（原则：宁窄不宽，先让一条链路真实跑通）

| 日 | 要点 | 任务 |
|---|---|---|
| **D8 11/23** 拆解项目 | 画三段时序图：源链事件长什么样（topic0 稳定、至少一个 indexed 参数）、`react()` 判断什么、目标链做什么、失败怎么办。建仓库骨架 `contracts/` `script/` `test/` `README.md` | — |
| **D9 11/24** 跑通项目 | 只做一条源链 → 一条目标链，最简单条件，先拿到三笔真实交易。赛道 1 记每段时间戳；赛道 3 先用固定规则跑通再接模型 | **跑通你的项目（必做）** |
| **D10 11/25** 加保护 | 只在该触发时触发（价格阈值、只认指定地址）；同一事件只执行一次（幂等）；单次与累计金额设上限 | —；20:00 Session 3 项目讨论 |
| **D11 11/26** 写测试 | 想清楚出问题会怎样：余额耗尽自动停、目标链交易失败、有人冒充调用回调。用 Foundry 测「正常」+「应该被拦下」各一个 | 给项目加保护 |
| **D12 11/27** 工程化 | 部署脚本一键可复现；README 写清安装、部署、运行、如何验证；用了 AI 的话写明哪些由 AI 生成、你如何验证 | — |
| **D13 11/28** 提交包 | 公开仓库、合约地址、三笔交易哈希（源链 / Reactive / 目标链）、<3 分钟演示视频；赛道 1 附全链路延迟实测，赛道 3 说明 AI 在运行时做什么决定 | 提交包 |
| **D14 11/29** Demo Day | 3 分钟演示 + 2 分钟问答，按 Execution / Integration / Usefulness / Craft 四项给反馈；15:00 Session 4 | Demo Day 展示 + 结营反馈 |

### 冲刺｜11/30 – 12/05

按 Demo Day 反馈补齐材料，在 DoraHacks 正式提交。建议至少提前一天，避免截止前网络或页面问题。后续如在 Reactive Circuit 清单内其他黑客松获奖，可按该计划官方规则申请额外奖励（不属于本次奖金承诺）。

## 4. 场景手册：六个场景

模板统一为：**听什么事件 → react() 里判断什么 → 回调做什么 → 参考代码 → 常见的坑 → 适合哪个赛道**。

| # | 场景 | 难度 | 监听 | 判断 | 回调动作 | 坑 | 赛道 |
|---|---|---|---|---|---|---|---|
| 1 | 链上监控：大额转账 / 代币流量 | 入门 | 源链 ERC-20 `Transfer(from,to,value)`，`address(0)` 表示全部代币 | value 超阈值 / 累计流量超上限 | 只在 Reactive 上 emit，或回调到目标链记录 / 报警 | 订阅全量代币会被打爆，每次 `react()` 都收费，先 `_limit`；订阅只能「等于」匹配，「大于 X」只能在 `react()` 里判 | 2 |
| 2 | 钱包守护：异常转出自动保护 | 入门→进阶 | 代币 + `Transfer` topic0 + topic1 = 你的地址 | 单笔或短时累计转出超阈值 | 回调 Guardian：把剩余代币转冷钱包 / 暂停金库 | Guardian 能动你的币，权限写死只能转到指定冷钱包 | 2 |
| 3 | 止损 / 止盈 | 进阶 | Uniswap V2 池子 `Sync(reserve0,reserve1)` | 用储备算价格，跌破止损 / 涨过止盈 | 用事先授权代币卖出；执行合约 emit 完成事件 → Reactive 标记完成并退订（一次性订单） | 测试网要自己部署代币和池子；卖出设最低成交量（滑点保护） | 1 或 2 |
| 4 | 跨链价差套利 | 挑战 | 两条链两个池子的 `Sync` | 分别记两边最新价（看 `log.chainId`），价差 > 手续费+gas+安全边际，且过冷却时间 | 便宜链买 / 贵链卖，或单链再平衡；**记录三段延迟** | 两笔回调不原子，一边失败要有处理；主网要跟专业 bot 竞争；Base Sepolia 200ms 出块，冷却用时间戳 | 1 |
| 5 | 定时任务：定投 / 巡检 / 到期执行 | 入门→进阶 | Cron 事件（`0x…fffFfF`，如 Cron100 按区块间隔，不保证墙钟周期） | 到点了吗、还剩几次 | 定投买入 / 读健康度补仓 / 到期解锁续费 | Cron 越频繁越费钱，先算一天花多少；欠款后静默停止，要监控 | 2（跨链再平衡可投 1） |
| 6 | 事件驱动的 AI 代理 | 挑战 | 价格变化 / 大额转账 | — | `react()` → 回调 emit「需要决策」→ 链下 AI 读上下文给决定 → 写回链上（白名单动作 + 额度内）→ Reactive 订阅决策事件 → 执行 | 别把私钥交给模型自由支配；AI 输出必须在合约里二次校验 | 3 |

**怎么挑**：第一次写合约 → 场景 1 / 5，赛道 2；有 DeFi 经验 → 场景 3 → 4，赛道 1；想做 AI → 场景 6，赛道 3；时间紧 → 场景 1 或 5 加一条自己真会用的规则。

**场景安全边界**（官方明确写的，写进 README 会加分）：
- 钱包守护不能撤销已发生的转账，不能保证抢在攻击者前搬走资产。
- 止损示例的 `Sync` 只适用 V2 风格池，储备现货价比可被操纵，**不是安全预言机**，触发不保证成交价。
- 跨链套利回调不负责桥接资产，两笔交易不原子；不得承诺套利收益。
- Cron 是区块节奏，不是严格墙钟定时。AI 场景含链下模型，不代表全在链上。
- 所有练习只用测试网（Lasna / Sepolia），不实盘。

## 5. 结业与评分

**结业条件（需同时满足）**
1. 完成「跨链回调三步走」+「跑通你的项目」两个必做任务。
2. 核心任务累计 ≥ **600 分**（核心任务共 12 个、满分 1040；Session 出席、社群互动等附加任务不计入）。

**Demo Day 反馈标准**（对齐黑客松官方评审，各项 1–5 分、权重相同）

| 维度 | 看什么 |
|---|---|
| Execution 执行 | 核心流程是否真跑通，链上能否查到 Reactive 交易及其触发的回调 |
| Integration 集成 | Reactive 是否承担实际功能；赛道 1 要说清为什么需要速度，赛道 3 要说清 AI 如何参与运行 |
| Usefulness 实用性 | 是否解决真实用户的问题 |
| Craft 质量 | 代码是否清晰可读、Demo 能否运行、文档是否足以让他人复现 |

表现突出者优先推荐给 Reactive 官方团队，争取生态支持与孵化。

## 6. 正式提交材料清单

- 公开开源仓库链接
- 已部署的合约地址
- 覆盖「源链事件 → Reactive 执行 → 目标链回调」的**交易哈希**（建议附区块浏览器链接并标明各环节）
- 演示视频（<3 分钟，提交前复核官方表单限制）
- 赛道 1：全链路延迟实测｜赛道 3：AI 在实际运行中做什么决定

> 共学结业、Demo Day 展示、黑客松正式提交是**三件不同的事**；在 WCB 完成任务 ≠ 已在 DoraHacks 报名 / 提交。

## 7. 必记的工程约束

1. **只写 Omni 新写法**（2026-10-28 主网迁移后）：不用 `vm` 修饰符、不用 ReactVM、不 `emit Callback`、不碰系统合约 `0x…fffFfF`（只有 Cron 事件来自它）；`react()` 必须加 `onlySystem`；回调用 `requestCallbackV_1_0`。
2. `main` 分支 demos 是旧写法，**只参考思路**；新写法看 `omni` 分支。
3. Base Sepolia 200 毫秒出块 → 计时、冷却、截止时间一律用**时间戳**，不用区块数（黑客松规则明确要求）。
4. 回调合约第一个参数必须是 `address`（代理注入），并且要有余额才能被调用。
5. 私钥 / 助记词永不外传，不写进代码、笔记、GitHub；打卡笔记是公开的。

## 10. WCB 平台当前任务状态（2026-11-18 截图）

页面标签：`进行中 4`、`即将开始 12`、`已完成 4`，另有已提交/待审核、已驳回、已失效、助教反馈。所有进行中任务均显示「剩余 18 天」（截止 12/05 23:59 北京时间）。

| 状态 | 分值 | 任务名 | 对应日 | 备注 |
|---|---|---|---|---|
| 进行中 | +20 | 参与或回看：Online Session 2｜技术 Q&A | D4 11/19 20:00 | 附加任务，不计入核心 600 分门槛 |
| 进行中 | **+80** | 第一次部署：监控 Sepolia 上的代币转账 | D3 11/18 | 核心任务，分值是核心任务里最高的一档 |
| 进行中 | +60 | 读懂费用：用 **cast 查询 Lasna 系统合约参数** | D2 11/17 | 核心任务。**注意不是写概念，是要把参数查出来** |
| 进行中 | +60 | 用例拆解卡：Reactive 适合解决什么问题 | D1 11/16 | 核心任务，可补交 |

四项合计 220 分。核心任务共 12 个 / 1040 分，结业门槛 600 分，因此 D3 的 80 分 deployment 任务不能跳过。

**关键校正**：之前我按课程大纲把 D2 理解为“读懂费用”概念题，截图显示平台要求的是**用 `cast` 查询 Lasna 系统合约的实际参数**，并把查询结果贴进打卡笔记。因此今天的笔记必须包含 `_gasPriceCoeffPer1000()`、`_extraGas()`、`_maxChargeGas()` 三条命令的**真实输出**。

> 进行中任务的完整 JSON 需要 GitHub 登录才能拉取（`tasks.listForLearner` 返回 401），截图是当前唯一来源；接口边界与拉取命令见 `docs/wcb-api.md`。
