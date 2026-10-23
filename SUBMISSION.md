# 提交说明

- 姓名或称呼：（待填写）
- 应聘方向：技术
- 打码手机号：（待填写，格式：前三位 + `****` + 后四位）

## 结果概览

**问题判断**：原实现的额度检查发生在模型调用之后，且依据请求鉴权时的 `context.user.credits` 快照做 check-then-act——额度为 0 仍会发起 `mock-ai.started` 所代表的接口调用；并发请求都读到同一快照并通过检查，`decrement` 无 `> 0` 保护，额度可扣成负数。

**方案**：预留-退还。原子条件更新 `UPDATE ... WHERE id = ? AND credits > 0`（`src/demo-ai-app/planGeneration.ts:67-68`）先占额度——单条语句 + PostgreSQL 行锁，判断与扣减无时间窗，这是防透支的具体一步；额度不足立即 402，模型调用不会发生；预留之后的任何失败（模型抛错或落库失败）退还 `+1` 再报错。前端常显剩余额度并在每次尝试后刷新，402/失败提示说明原因与下一步，新增历史日程展示。

**完成状态**：服务端改造、前端展示、两级自动化验证（核心流程 vitest 7/7 + HTTP 端到端 16/16 断言）全部完成并通过；仅 AC-5 的浏览器人工步骤未在本环境执行（无浏览器），已在验证记录中明确标注并附步骤清单。

## 交付入口

- 设计与拆解：[SPEC.md](SPEC.md)（范围与假设、流程、并发思路、实现步骤、验收案例 AC-1..AC-6，含修订记录）。
- 主要改动：`src/demo-ai-app/planGeneration.ts`（核心流程，预留/退还）、`src/demo-ai-app/operations.ts`（薄封装与错误映射、`getUserCredits` 查询）、`src/demo-ai-app/DemoAppPage.tsx`（余额展示、差异化提示、历史日程）。
- 验证脚本：`src/demo-ai-app/planGeneration.test.ts`（vitest）、`scripts/test-db-setup.sh`（一次性测试库）、`scripts/verify-http.mjs`（HTTP 端到端）。

运行方式（详见 [README.md](README.md)）：

```sh
wasp start db               # Docker 开发库，保持运行
wasp db migrate-dev         # 首次
wasp start                  # 应用与 API（默认 3001）

npm test                    # 核心流程测试（自动重建测试库）
node scripts/verify-http.mjs http://localhost:3001   # HTTP 端到端（含并发）
```

## 验证记录

逐案例的命令、预期与实际结果见 [VERIFICATION.md](VERIFICATION.md)，含两次重复运行输出、AC-6 的 HTTP 层附带证据，以及 AC-5 未执行项的明确标注。基础检查（`wasp compile` + 两个 `tsc --noEmit`）全部通过。

## 待办与交接

- **需提交者本人完成**：填写本文件顶部称呼与打码手机号；按 `TASK.md` 要求 fork 本仓库、从分支 `tech/credits-reserve-refund` 提 PR，标题格式 `技术｜称呼｜138****5678`。
- AC-5 浏览器人工步骤：见 [VERIFICATION.md](VERIFICATION.md) 末尾清单，预计 10 分钟内完成。
- 明确不做（题目豁免）：崩溃窗口额度丢失/退还不补偿；支付、跨账号防刷等。完整假设与取舍见 [SPEC.md §2](SPEC.md)。
- `package-lock.json` 在工作区有一处环境产生的未提交改动（`wasp install` 所致），未纳入本分支提交；如需可自行提交。
