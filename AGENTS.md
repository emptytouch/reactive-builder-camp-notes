# AGENTS.md — 给编程工具的项目上下文

本仓库是 Reactive Builder Camp（2026-11-16 ~ 12-05）的学习与练习仓。任何 AI 编程工具在本仓库工作前，先读这一页。

## 技术上下文（必读）

我在用 **Reactive Network 的 Omni 新版本**（2026-10-28 主网迁移之后）。请只参考这些来源，不要用训练数据里的旧写法：

- <https://dev.reactive.network/reactive-contracts>
- <https://dev.reactive.network/callback-contracts>
- <https://dev.reactive.network/economy>
- <https://dev.reactive.network/origins-and-destinations>
- <https://github.com/Reactive-Network/reactive-smart-contract-demos/tree/omni/src/micro-demos>

**禁止出现**：

- `vm` 修饰符、ReactVM 相关写法
- `emit Callback`（旧写法）
- 把系统合约 `0x…fffFfF` 当普通回调目标（只有 Cron 事件来自它）

**必须做到**：

- `react()` 必须加 `onlySystem`
- 回调请求使用 `requestCallbackV_1_0`
- Callback 合约继承 `AbstractCallback`，第一个参数是 `address`（由代理注入）
- 回调校验 `onlyServiceProvider` + `onlyCallbackSender` 两道
- 回调合约自身要有余额才能被调用

## 链路

```
源链（Origin）合约 emit 事件
  → Reactive 合约 react(LogRecord)：按 chainId / 合约地址 / topic0 / indexed 参数匹配，条件判断
  → 回调请求 → Callback Proxy（目标链）→ 我的回调合约函数
```

- Reactive 侧测试网：**Lasna**
- 目标链测试网：Sepolia / Unichain Sepolia / Base Sepolia
- 主网与测试网不能混用
- 链上任何时间逻辑用**时间戳**（Base Sepolia 200ms 出块，不能用区块数）

## 费用模型（会影响代码设计）

- 三个数：`balance`（余额）、`reserves`（预付）、`debt`（欠款）。**只要有 debt 就停止执行**。
- 每次 `react()` 都收费，回滚也收费 → 订阅不要过宽，务必加次数上限（`_limit`）与金额上限。

## 仓库约定

- `notes/YYYY-MM-DD.md`：每天一个笔记文件，含「今天做了什么 / 关键概念 / 卡在哪 / 链上痕迹 / 明日待办」。这是我的打卡材料，**不要删改历史笔记**。
- `docs/camp-map.md`：课程全景（内容、任务、评分）
- `docs/ai-playbook.md`：AI 协作方式与 prompt 模板
- 合约与脚本后续按 `contracts/` `script/` `test/` 组织
- 参赛作品另开独立仓库，不在本仓提交

## 硬性红线

1. 绝不写入、读取、索取私钥或助记词；不生成含真实密钥的 `.env`。
2. 不编造交易哈希、合约地址、测试结果。未验证就写「未验证」。
3. 所有练习仅用测试网，不产生实盘操作建议。
4. 我用了 AI 生成的代码，在 README 中注明范围与验证方式（活动要求）。

## 响应风格

- 中文，直接给结论，再给验证方式。
- 报错排查：先给「最可能的 3 个原因 + 每个原因的验证命令」，不要直接重写代码。
- 代码审稿：只报问题，不擅自重构；每条问题说明后果。
