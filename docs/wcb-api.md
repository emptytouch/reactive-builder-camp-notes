# WCB 学习平台接口备忘（供每日拉取课程内容用）

来源：2026-11-18 对 <https://web3career.build> 的前端代码与接口实测。

## 公开可读（无需登录）

| 内容 | 入口 |
|---|---|
| 课程详情 + Markdown 正文 + 课程周安排 | 页面 HTML 的 `self.__next_f` 流中的 `T<hexlen>,` 块：`21`=课程详情，`22`=第0周，`23`=场景手册，`24`=第1周，`25`=第2周 |
| 课程详情（JSON） | `GET /api/trpc/program.getById?input={"json":{"idOrSlug":"reactive-builder-camp"}}` |
| 赛道列表 | `GET /api/trpc/tracks.listForProgram?input={"json":{"programId":"cmw0iw7x30mgpk501h3vukxtm"}}` → 返回 `cmw0iw88h0mguk501wt9humbk` / Reactive Builder |

注：`T` 后的长度是**十六进制字符数**，按这个长度切分能完整取回中文正文。

## 需要 GitHub 登录（401 UNAUTHORIZED）

以下接口没有公开版本（已验证 `tasks.list` / `tasks.listForProgram` / `tasks.listPublic` 等都不存在）：

- `tasks.listForLearner` —— **进行中的任务列表**（含状态、分值、提交要求）
- `tasks.myTotalPoints` —— 我的学分
- `announcements.listForLearner` —— 公告
- `icl.getMyProgramStatus` —— 我的共学状态
- `assistants.getStudentTasks` 等助教视角接口

## 在已登录的浏览器里拉取任务列表

打开学习页，F12 → Console，粘贴：

```js
fetch('/api/trpc/tasks.listForLearner?input=' + encodeURIComponent(JSON.stringify({
  json: { programId: 'cmw0iw7x30mgpk501h3vukxtm', locale: 'zh' }
})), { credentials: 'same-origin' })
  .then(r => r.json())
  .then(d => console.log(JSON.stringify(d?.result?.data?.json ?? d, null, 2)))
```

把输出贴给我即可，我会翻译成中文任务清单（状态 / 分值 / 截止 / 提交要求）并更新 `docs/camp-map.md`。**不要贴 cookie、token 或任何登录凭据。**

## 结论

旧写法抓取课程大纲没问题；**进行中的任务必须走已登录会话**，我不会也不会要求用户提供账号凭据。
