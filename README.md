# Agent Harness README

解析 `AGENTS.md` 与 `CLAUDE.md` 的关系与架构设计。

> **理论来源**:
> - [*Effective harnesses for long-running agents*](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)（Anthropic, 2025-11-26）—— 双代理方案：Initializer + Coding Agent，解决长程任务中的上下文丢失问题
> - [*Harness design for long-running application development*](https://www.anthropic.com/engineering/harness-design-long-running-apps)（Anthropic Engineering）—— GAN 风格三代理架构：Planner / Generator / Evaluator，自评偏差问题与 Grading Criteria

---

## 整体架构：四阶段 SDD + GAN 三代理

### GAN 模式流程图（来源：Anthropic）

```
用户 prompt（1–4 句话）
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                     PLANNER                              │
│  展开 prompt → 产品规格文档（spec.md）                     │
│  识别 AI 功能机会，设计 Visual Design Language            │
└─────────────────────────┬───────────────────────────────┘
                          │ specs/[NNN]/
                          ▼
┌─────────────────────────────────────────────────────────┐
│              GENERATOR ←──────→ EVALUATOR                │
│                                                         │
│  生成器构建功能      评审师测试                          │
│  一次 Sprint         浏览器 Playwright                   │
│  一条 git commit     打分：Design / Originality         │
│                     / Craft / Functionality             │
│        ▲            │                        │         │
│        │            │                        ▼         │
│        └──── 反馈循环（FAIL 则返回 Generator 修复）──────┘
└─────────────────────────────────────────────────────────┘
```

### 完整四阶段 SDD 流程

```
用户prompt
    │
    ▼
Orchestrator（编排器）───▶ Planner（规划师）
                            │
                            ▼
                       Generator（生成器）
                            │
                            ▼
                       Evaluator（评审师）◀── Playwright MCP
```

基于 **spec-kit 规范驱动开发（SDD）** + **GAN 风格三代理循环**。

---

## 文件职责矩阵

| 文件 | 归属 | 职责 |
|------|------|------|
| `AGENTS.md` | 团队共享 | 定义四个阶段（PLAN/TASK/IMPLEMENT/CHECK）的规则、产物与约束 |
| `CLAUDE.md` | Claude Code 专用 | 定义三个子代理（planner/generator/evaluator）的 prompt 模板、工具、模型选择 |
| `.claude/agents/planner.md` | Claude Code | Planner 子代理的完整 prompt |
| `.claude/agents/generator.md` | Claude Code | Generator 子代理的完整 prompt |
| `.claude/agents/evaluator.md` | Claude Code | Evaluator 子代理的完整 prompt |
| `.claude/agents/orchestrator.md` | Claude Code | Orchestrator（入口编排器）的完整 prompt |
| `.claude/settings.json` | Claude Code | MCP 服务器配置、Hook 规则 |
| `.claude/hooks/` | Claude Code | Session 级别的自动化脚本 |

---

## 四个阶段（Phase）

### Phase 1 — PLAN
**产物**: `specs/[NNN]/spec.md` + `plan.md` + `contracts/` + `data-model.md` + `research.md`

**流程**:
```
/speckit.specify  →  spec.md   （用户需求：what + why）
/speckit.plan     →  plan.md   （技术架构：how）
```

**约束**:
- spec.md 禁止出现实现细节（无技术栈、无函数名）
- plan.md 必须包含 Visual Design Language 章节（前端时）

---

### Phase 2 — TASK
**产物**: `specs/[NNN]/tasks.md`

**流程**:
```
/speckit.tasks    →  tasks.md   （从 plan.md + contracts/ + data-model.md 派生）
```

**任务格式**:
```markdown
- [ ] T-001: 任务描述
      Criterion: 验收标准（可观测，非"功能正常"）
      Size: S/M/L
      [P]  parallelizable
      Dependencies: T-XXX
```

---

### Phase 3 — IMPLEMENT
**产物**: `sprint-contract.md` + git commit

**流程**:
```
Generator 选择下一个任务
    │
    ▼
Generator 起草 sprint-contract.md
    │
    ▼
Evaluator 审查 contract → 协商直到 CONTRACT APPROVED
    │
    ▼
Generator 实现 → 自验 → git commit
    │
    ▼
写 eval-trigger.txt → 触发 CHECK
```

**约束**:
- 必须先有 CONTRACT APPROVED 才能开始写代码
- 每次 Session 必须运行 startup ritual（`cat claude-progress.txt` + `git log` + `bash init.sh` + smoke test）

---

### Phase 4 — CHECK
**产物**: `eval-result-{N}.md`

**评审维度**:

| 维度 | 权重 | 阈值 | 说明 |
|------|------|------|------|
| Design quality | 30% | ≥ 7/10 | 视觉身份统一，有意为之 |
| Originality | 30% | ≥ 6/10 | 拒绝 AI 垃圾（紫渐变白卡片等） |
| Craft | 20% | ≥ 7/10 | 排版、间距、对比度技术正确 |
| Functionality | 20% | ≥ 8/10 | **硬门槛**：所有合同标准必须通过 |

**判定**:
- 任一维度未达阈值 → `SPRINT FAIL` → 返回 Generator 修复
- 全部通过 → `SPRINT PASS` → 标记任务 ✅，进入下一任务

---

## 子代理详解

### Orchestrator（入口）
- **工具**: Read, Write, Bash, Agent
- **模型**: claude-opus-4-6
- **职责**: 用户只和 Orchestrator 对话。它读取项目状态，决定下一步进入哪个 Phase，然后调用对应子代理。
- **决策逻辑**（伪代码）:
  ```
  if no spec.md → planner(PLAN phase)
  else if spec.md exists but no tasks.md → planner(TASK phase)
  else if tasks.md has uncompleted tasks and no sprint-contract → generator(propose contract)
  else if sprint-contract exists and eval-trigger exists → evaluator(CHECK)
  else if all tasks done → evaluator(/speckit.analyze)
  ```

### Planner
- **工具**: Read, Write, Bash, WebFetch
- **模型**: claude-opus-4-6
- **职责**: 只写规范文档，不写任何实现代码。运行一次（PLAN）+ 一次（TASK）。

### Generator
- **工具**: Read, Write, Edit, Bash, Agent
- **模型**: claude-sonnet-4-6
- **职责**: 每个 Sprint 实现一个任务。必须经过 contract 协商，必须经过 Evaluator 验证。

### Evaluator
- **工具**: Read, Write, Bash, Playwright MCP（navigate/screenshot/click/fill/evaluate）
- **模型**: claude-opus-4-6
- **职责**: 两种模式：
  1. **Contract Review**: 审查 sprint-contract.md 是否可测试
  2. **CHECK**: 用 Playwright 实际操作应用，逐项评分

---

## CLAUDE.md 与 AGENTS.md 的核心区别

| 维度 | AGENTS.md | CLAUDE.md |
|------|-----------|-----------|
| 受众 | 通用（Cursor/Copilot/Codex 等） | Claude Code 专用 |
| 核心内容 | 四阶段流程、产物、约束 | 子代理 prompt 模板 + 工具 + 模型 |
| 产物定义 | `spec.md`, `plan.md`, `tasks.md` | `planner-spec.json`（旧版遗留） |
| 编排器 | 无独立定义 | `orchestrator.md` 独立子代理 |
| Hooks | 未定义 | PostToolUse（git commit 提醒）、Stop（clean state 验证） |
| Slash Commands | 未定义 | `new-sprint`, `eval`, `status` |

---

## 关键约束（Hard Rules）

1. **Phase Gate**: 必须等上一个阶段的产物存在，才能进入下一阶段
2. **Contract First**: 未经 Evaluator contract 批准，禁止写代码
3. **Browser Verification**: 任务完成必须通过 Playwright 验证，不允许自我宣告完成
4. **Clean State**: 每次 commit 后必须测试通过、无未提交变更
5. **Never Praise Own Work**: Generator 不评价自己的输出，Evaluator 才是裁判

---

## 上下文管理

- **状态存储**: 文件（`planner-spec.json`、`claude-progress.txt`、`eval-result-{N}.md`、`sprint-contract.md`）
- **Compaction**: Opus 4.6+ 使用 Compaction（非 Context Reset）
- **Sonnet 4.5**: 需要每 Sprint 启动新 agent（读取 artifacts，不依赖对话历史）

---

## Harness 演进规则

> 每次模型升级后重新审视：
> - [ ] Context Reset 是否仍然必要？
> - [ ] Sprint Contract 协商是否仍有价值？
> - [ ] 四个评审维度是否仍有区分度？
> - [ ] Planner 和 Generator 能否合并？
>
> **原则**: 更简单的 harness + 更强的模型 > 复杂的 harness + 更弱的模型

---

## 核心设计理念（来自 Anthropic 文章）

### 问题：长程任务的两个失败模式

1. **One-shot 失败**：Agent 试图一次性完成整个应用，导致半成品
2. **过早完成**：Agent 在取得初步进展后过早宣告项目完成

### 关键洞察

1. **Generator-Evaluator 分离解决自评偏差**
   Agent 倾向于自我赞美输出。调优一个独立且持怀疑态度的 Evaluator，比让 Generator 自我批评更可 tractable。

2. **Grading Criteria 防止虚假通过**
   - Design quality（30%）：统一的视觉身份
   - Originality（30%）：拒绝 AI 垃圾（紫渐变、白卡片）
   - Craft（20%）：技术性正确（排版、间距、对比度）
   - Functionality（20%）：**硬门槛**，≥8/10

3. **结果对比**
   - Solo agent：20 分钟，$9 → 产出损坏的应用
   - 完整 harness：6 小时，$200 → 功能完整、有设计的软件

4. **演进方向**（Opus 4.6+）
   随着模型能力提升，可简化 harness 移除 Sprint 分解，保留 Planner 和 Evaluator 组件。

---

## 参考资料

- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)（Justin Young, Anthropic, 2025-11-26）
- [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)（Anthropic Engineering）
- [spec-kit](https://github.com/github/spec-kit)——规范驱动开发工具
