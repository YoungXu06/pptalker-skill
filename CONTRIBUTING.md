# 贡献指南 · Contributing

感谢你为 **PPTalker Skill** 做贡献！本 skill 的本质是一套结构化的 Agent 指令 + 知识库，它编排 PPTalker 的 MCP 工具把幻灯片转成配音讲解视频。贡献时请保持「Agent 可读、流程稳健、不浪费用户额度」三个特性。

---

## 项目结构与职责

| 路径 | 职责 | 改动时注意 |
|------|------|-----------|
| `SKILL.md` | 主入口：触发词、工具图、8 步工作流、硬规则 | 改流程必须同步更新对应步骤编号与「Retry Policy / Notes Display Format」 |
| `references/note-generation-prompt.md` | 讲稿生成 prompt 模板 + TTS 安全规则 + 语气/禁用词块 | 改讲稿规则必须同步 SKILL.md Step 5 与自检清单 C1–C8 |
| `references/error-handling.md` | 硬规则、错误码、超时策略、套餐限制 | 新增错误码要给出明确处理动作；不改动既有硬规则编号含义 |
| `references/examples.md` | 端到端范例 + 各格式取文 recipe | 新增格式需补一条完整的取文策略 |

---

## 三条不可削弱的硬规则

任何改动都不得削弱以下规则；如确需调整，请在 PR 描述里说明理由与影响：

1. **先审后渲** —— 未经用户明确确认 `final_notes`，绝不调用 `pptalker_create_video`。
2. **渲染超时不重试** —— `create_video` 超时即视为任务已提交，改去轮询 `pptalker_get_video_status`，绝不重复调用（避免重复扣费）。
3. **重试至多 3 次** —— 任一 MCP 工具失败最多重试 3 次，仍失败即停下报错并询问用户。

---

## 提交前自检

### 1. 流程 / 工具改动
- 改动任一步骤，需同步 `SKILL.md` 对应的步骤编号、`Tool Map` 与 `Workflow` 描述。
- 不要让 `SKILL.md` 引用 `references/` 中不存在的小节或占位符。

### 2. 讲稿 prompt 改动
- 修改语气块 / 禁用词块 / TTS 规则，需同步更新 `note-generation-prompt.md` 中的占位符表与自检清单 C1–C8。
- 新增语言的禁用词块，按现有 ZH / EN / generic 三类的结构补齐。

### 3. 错误处理一致性
- 新增/修改错误码时，`error-handling.md` 与 `SKILL.md` 的描述要一致。
- 超时与重试策略改动，三处（SKILL.md、error-handling.md、本指南）需保持一致。

---

## 提交流程

1. Fork 并新建分支：`git checkout -b feat/your-feature`
2. 若条件允许，用一份真实的 PPT / PDF / HTML 跑一遍完整工作流（上传 → 解析 → 确认讲稿 → 渲染 → 出片）验证。
3. 提交信息用约定式格式：`feat: ...` / `fix: ...` / `docs: ...` / `refactor: ...`
4. 发起 PR，描述：改了什么、为什么、如何验证。

---

## 报告问题

提 Issue 时请附上：输入文件类型与页数、期望产出、实际产出（含 `request_id` 与错误码，如有）、复现步骤。**请勿在 Issue 中粘贴你的 `PPTALKER_API_KEY`。**

---

## 相关项目

本 skill 与 [**content-to-slides**](https://github.com/YoungXu06/content-to-slides) 组成「内容 → 幻灯片 → 讲解视频」流水线。涉及两者衔接的改进（如讲稿格式兼容）也欢迎提来。

---

Thanks for contributing! 让做完的幻灯片更轻松地开口讲。
