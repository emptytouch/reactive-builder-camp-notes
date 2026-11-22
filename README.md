# Reactive Builder Camp · 共学笔记与实践

Web3 Career Build「Reactive Builder Camp」两周共学（2026-11-16 ~ 11-29）+ 冲刺期（11-30 ~ 12-05）的学习仓库。

目标：两周内跑通 **源链事件 → Reactive react() 判断 → 目标链回调** 的完整跨链自动化链路，并在此基础上做出一份可提交 Reactive Omni Buildathon 的作品。

## 我在学什么

Reactive Network 让合约自己「动起来」：任意链上发生事件（转账、成交、价格变化、定时），Reactive 合约自动收到并判断条件，满足就在目标链上发起交易。不用自己跑机器人、租服务器盯链。

一句话抽象：**某条链上发生了 X，满足条件 Y，就在某条链上做 Z。**

## 进度

### 第 1 周｜学会写跨链自动化

| 日期 | 日 | 主题 | 任务 | 笔记 |
|---|---|---|---|---|
| 11/16 | D1 | 为什么需要 Reactive · 开营 | 用例拆解卡 | [notes/2026-11-16.md](notes/2026-11-16.md) |
| 11/17 | D2 | 源链、目标链与资金模型 | 读懂费用 | [notes/2026-11-17.md](notes/2026-11-17.md) |
| 11/18 | D3 | 场景 1 监控：第一个 Reactive 合约 | 第一次部署 | notes/2026-11-18.md |
| 11/19 | D4 | 场景 2 从监控到动作：跨链回调 | 跨链回调三步走（必做） | notes/2026-11-19.md |
| 11/20 | D5 | 场景 5 定时任务与动态订阅 | 定时或动态订阅 | notes/2026-11-20.md |
| 11/21 | D6 | 选题 | 项目方向卡 + DoraHacks 报名 | notes/2026-11-21.md |
| 11/22 | D7 | 缓冲日 | 补进度 | notes/2026-11-22.md |

### 第 2 周｜做项目

| 日期 | 日 | 主题 | 任务 | 笔记 |
|---|---|---|---|---|
| 11/23 | D8 | 拆解项目：事件 / 条件 / 动作 | — | notes/2026-11-23.md |
| 11/24 | D9 | 跑通你的项目 | 跑通你的项目（必做） | notes/2026-11-24.md |
| 11/25 | D10 | 加保护：只在该触发时触发 | — | notes/2026-11-25.md |
| 11/26 | D11 | 写测试：正常 + 出问题 | 给项目加保护 | notes/2026-11-26.md |
| 11/27 | D12 | 工程化：代码 / 文档 / 可复现 | — | notes/2026-11-27.md |
| 11/28 | D13 | 提交包 | 提交包 | notes/2026-11-28.md |
| 11/29 | D14 | Demo Day · 结营 | Demo Day 展示 | notes/2026-11-29.md |

结业条件：完成「跨链回调三步走」+「跑通你的项目」两个必做任务，核心任务累计 ≥ 600 分（共 12 个核心任务 / 满分 1040）。

### 冲刺

11/30 ~ 12/05 按 Demo Day 反馈打磨，在 [DoraHacks](https://dorahacks.io/hackathon/reactive-omni-buildathon/detail) 正式提交。营内建议 12/05 23:59（北京时间）前交齐，官方截止 12/06 04:00。

## 链上交易索引

每条链路三段，提交包要求能核验。跑通一笔就补一行。

| # | 场景 | 源链事件 | Reactive 执行 | 目标链回调 | 时间 |
|---|---|---|---|---|---|
| 1 | | | | | |

## 目录约定

```
.
├─ README.md           # 本文件：目标、进度、交易索引
├─ AGENTS.md           # 给 AI 编程工具的项目上下文（Omni 写法 / 红线 / 约定）
├─ docs/
│  ├─ camp-map.md      # 课程全景：每日学习内容 + 任务 + 场景手册 + 评分标准
│  ├─ ai-playbook.md   # AI 协作手册：每日循环 + 分工 + 可复制 prompt
│  └─ lessons/         # 每日中文精讲（官方英文文档的翻译 + 讲解 + 自测）
├─ notes/              # 每日笔记，一天一个 md，命名 YYYY-MM-DD.md
└─ .gitignore
```

- 想看「这 14 天到底学什么、每天交什么」→ [docs/camp-map.md](docs/camp-map.md)
- 想看「AI 怎么配合我每天推进」→ [docs/ai-playbook.md](docs/ai-playbook.md)

第 2 周开始做项目后，本仓只保留学习与练习记录；参赛作品另开独立仓库（`reactive-<项目名>`），保持作品仓提交历史干净——这是「作品在活动期间构建」最直接的证据。

## 每日中文精讲

官方文档全英文，这里按天做中文讲解（术语对照 + 重点 + 坑 + 自测题），笔记和打卡从这些内容里提炼。

| 日 | 主题 | 精讲 |
|---|---|---|
| D1 | 为什么需要 Reactive | [docs/lessons/D01-why-reactive.md](docs/lessons/D01-why-reactive.md) |
| D2 | 源链、目标链与资金模型 | [docs/lessons/D02-origins-destinations-economy.md](docs/lessons/D02-origins-destinations-economy.md) |
| D3 | 第一个 Reactive 合约：订阅 Sepolia 的代币转账 | [docs/lessons/D03-first-reactive-contract.md](docs/lessons/D03-first-reactive-contract.md) |
| D4 | 从监控到动作：跨链回调三步走（必做） | [notes/d4-callback-task.md](notes/d4-callback-task.md)（三笔链路交易 + 失败实验） |
| D5 | 定时任务与动态订阅（Cron / 运行时增删订阅） | [docs/lessons/D05-cron-and-dynamic-subscription.md](docs/lessons/D05-cron-and-dynamic-subscription.md)｜任务提交件 [notes/d5-cron-task.md](notes/d5-cron-task.md) |

## 学习资源

- 官方模板仓（clone 即用，Foundry）：<https://github.com/ethpanda-org/reactive-builder-camp>
  - [速查表](https://github.com/ethpanda-org/reactive-builder-camp/blob/main/docs/cheatsheet.md)｜[FAQ](https://github.com/ethpanda-org/reactive-builder-camp/blob/main/docs/faq.md)
- [Reactive Contracts](https://dev.reactive.network/reactive-contracts)｜[Callback Contracts](https://dev.reactive.network/callback-contracts)｜[Economy](https://dev.reactive.network/economy)｜[Origins & Destinations](https://dev.reactive.network/origins-and-destinations)
- Omni 新写法示例：<https://github.com/Reactive-Network/reactive-smart-contract-demos/tree/omni/src/micro-demos>
- [Reactive 主网迁移完成（官方博客）](https://blog.reactive.network/reactive-mainnet-migration-complete/)

## 给 AI 的上下文（重要）

AI 训练数据里大量是 Reactive 旧写法，让 AI 写代码前先贴这段：

> 我在用 Reactive Network 的 Omni 新版本（2026-10-28 主网迁移后）。请只参考 https://dev.reactive.network/reactive-contracts 、https://dev.reactive.network/callback-contracts 、https://dev.reactive.network/economy 和 https://github.com/Reactive-Network/reactive-smart-contract-demos/tree/omni/src/micro-demos 。不要使用 `vm` 修饰符、ReactVM、`emit Callback` 或系统合约 `0x...fffFfF`（只有 Cron 事件来自它）。react() 必须加 onlySystem，回调用 requestCallbackV_1_0。

## 安全须知

- 全程使用**测试专用钱包**和测试网（Lasna / Sepolia），不要用存有资产的主钱包。
- **绝不**向任何人或 AI 工具提供私钥、助记词，也不要写进笔记、代码或任务材料。本仓库公开，`.env` 已在 `.gitignore` 中。
- 打卡笔记会公开展示在 GitHub 上。
