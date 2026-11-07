# dsh-conductor

> 单入口委托 · 后台执行 · 独立复核 —— 给 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）的「委托协议」。

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![topic: dsh-plugin](https://img.shields.io/badge/topic-dsh--plugin-blue)](https://github.com/topics/dsh-plugin)
[![status](https://img.shields.io/badge/status-P0%20pilot%20passed-brightgreen)](docs/pilot-p0.md)
[![plugin](https://img.shields.io/badge/plugin-MVP%20in%20progress-orange)](docs/roadmap.md)

## 为什么做它

重度使用 Agent 之后，瓶颈不再只是模型能力，而是**委托不可靠**：

- 派发没有统一入口，结果要在多个 Agent 界面之间人肉搬运；
- 任务失败不暴露，成功也可能只是“进程退出码为 0”的假成功；
- 多路并行产出的结论缺少独立复核，严重性没人校准；
- 中断之后不知道做到哪一步，也没法安全续跑；
- 做完之后没有可审计的 run 台账，复盘和汇报全靠回忆。

`dsh-conductor` 在 DSH 的对话入口与后台 Agent 之间补一层协议：**你只跟主 agent 说话，它按任务契约派活、按证据回收、按负向用例复核、把需要你拍板的事集中起来，并把全过程记成可恢复的 run 台账。**

## 它做什么

| 模块 | 职责 |
|---|---|
| **契约 Contract** | 结构化任务卡：目标 / 范围 / 交付路径 / 证据锚点 / 验收标准（含负向用例）/ 失败判定 / 预算 / 待决项 |
| **派发 Dispatch** | 后台运行 dsh headless（一期），后续扩 Codex / Claude；每步独立目录与日志 |
| **回收 Collect** | 产物落盘自动回收；**退出码 + 超时 + 结构化结果**三重校验，缺一即 `failed` |
| **复核 Review** | 独立 reviewer：按证据锚点抽查、校准严重性、跑“应当拒绝”的负向用例，输出「已证实 / 待验证 / 分歧」 |
| **决策 Decide** | 需要人拍板的事项集中成聊天内卡片，支持“先别改 / 暂停 / 让 dsh 再看一遍” |
| **台账 Ledger** | `runs/<id>/`（spec / spans / report / heartbeat）+ 可重放；中断后可续跑 |

### 一次委托长什么样

```text
你：评审这份方案，重点看合规、进度和工具链，先给结论和修改建议。

主 agent：已生成任务契约（3 路并行 + 1 路独立复核，负向用例 2 条）。
         [run #42 已启动] 预计 8 分钟；需要你决定的事项会单独列出。

（稍后）

主 agent：发现 3 个优先问题：2 个有直接证据（附原文锚点），1 个待负责人确认。
         复核降级了 1 条“环境不可达”（证据不足）。失败的一路已标记原因，不并入结论。
         [待你决定] ① 先改阶段门定义 ② 等启动会 ③ 让 dsh 再核一遍附录
```

## 设计原则

1. **契约先行**：没有写清交付物、证据与验收标准的任务不进入派发。
2. **证据锚点**：每条结论必须能定位到「文件 + 行号 + 原文引用」，机器自动校验。
3. **负向验收**：检查类任务必须包含“应当拒绝 / 应当失败”的用例，不能只测 happy path。
4. **fail-closed**：退出码、超时、结构化结果任一缺失即判失败；失败如实标记，不包装成功。
5. **可恢复、可审计**：run 台账 append-only 可重放；不把心跳当成恢复能力已实现。
6. **本地优先**：台账与产物默认落工作区，不发送到任何服务器。

## P0 影子试点（已完成）

在真实只读评审任务上验证了「派发 → 回收 → 复核 → 失败暴露」链路：

| 指标 | 结果 |
|---|---|
| 三路并行分析 + 一路独立复核 | **4/4 通过**（退出码、超时、结构化结果、证据锚点校验） |
| 派发墙钟 | 138.9 秒（三路并行；复核 121.0 秒） |
| 失败注入 | **3/3**（非零退出、超时、结构化缺失均判 `failed`） |
| 证据负向用例 | **2/2**（不存在路径、越界行号均被拒绝） |
| 独立复核产出 | 确认 29 条、待验证 1 条、分歧 1 条（并下调了执行者的严重度） |
| 未验证 | 恢复/续跑、写操作回滚、权限强制边界、跨 harness、成本口径（n=1） |

完整报告：[docs/pilot-p0.md](docs/pilot-p0.md)

## 目录

```text
dsh-conductor/
├── docs/
│   ├── design.md        # 产品设计：契约、派发、回收、复核、台账
│   ├── pilot-p0.md      # P0 影子试点报告（真实运行数据）
│   └── roadmap.md       # M0–M6 路线图
├── templates/
│   ├── contract.schema.json   # 任务契约 JSON Schema（v0.1）
│   └── playbooks/             # 6 个正向工作流模板
│       ├── review-panel.md
│       ├── contract-parallel.md
│       ├── negative-path.md
│       ├── root-entry.md
│       ├── evidence-grade.md
│       └── assumptions.md
└── LICENSE / README.md
```

## 路线图

- [x] M0 契约与影子试点 harness
- [x] P0 真实只读评审试点（4/4 通过，失败注入 3/3）
- [ ] M1 dsh headless adapter（派发 → 心跳 → 回收 → 三重校验）
- [ ] M2 `/review` 独立复核与证据锚点校验
- [ ] M3 `/run status` / `/run resume` 台账与续跑
- [ ] M4 `/decide` 聊天内决策队列
- [ ] M5 Codex / Claude adapter
- [ ] M6 npm 发布 + 插件市场收录

详见 [docs/roadmap.md](docs/roadmap.md)。

## 安装（M6 后可用）

```bash
# 需要 Node.js >= 22 与 dsh >= 0.2
dsh plugin add dsh-conductor
```

在 MVP 完成前，本仓库以设计、契约与试点证据为主，暂不提供可安装版本。

## 参与

欢迎 issue 讨论：真实委托场景、契约字段、复核隔离强度、失败判定口径。仓库已加 `dsh-plugin` topic。

## License

[MIT](LICENSE)

---

### English

`dsh-conductor` is a delegation protocol for DeepSeek Harness: you talk to one main agent, it dispatches work to background agents under an explicit task contract, collects artifacts fail-closed, runs an independent review with evidence anchors and negative test cases, queues decisions for you, and records everything in a resumable run ledger. P0 pilot passed (4/4 valid results, failure injection 3/3). Plugin MVP is in progress.
