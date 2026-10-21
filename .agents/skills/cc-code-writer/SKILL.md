---
name: cc-code-writer
description: 实现数禾后端功能、修复和重构，按任务组织验证与代码审查。
---

## Codex 运行约定

先读取 `../_runtime/codex-runtime.md`，再使用下述工作流程。Claude 工具名是历史记法，按运行约定映射到当前可用工具。没有实际运行或独立验证时，不宣称已通过。


# 数禾代码编写 Agent

主会话按需求定位、实现、验证并审查。复杂任务可按本技能的委托流程拆给 Implementer、Verifier 和 Reviewer；角色约束显式传入，不是 Codex 注册角色。提交只在用户已授权时执行。

严重度代号（P0-P5 / G6 / G7）的定义见 `shared-rules/glossary.md`；对用户的报告用「必修 / 建议级」桶名，不用编号。

## 何时使用

新功能开发、Bug 修复、重构、TDD；单任务或已拆好的任务清单都接。分析型批量派发（如并发诊断 30 个 skill）也用本 skill 的并发章分组与收口，只是派的是分析子 agent 而不是 Implementer。

| 不该用的场景 | 改用 | 原因 |
|------|--------|------|
| 需求还没拆成任务、要先规划 | cc-planner | 本 skill 只做轻量拆解，不做完整五阶段 |
| 只要审查，不写代码 | cc-code-reviewer | 本 skill 的内置审查只审自己派出去的改动 |

## 只在这几处停下来问用户

- 实际执行超出已有授权的数据写入或不可逆操作（删数据、改表结构、force push）。涉及金额的本地代码修复按风险验证，不因此重复询问。
- 缺少会影响交付的目标分支信息时先查上下文和工作区；普通本地修改不要求用户先新建分支。
- 需求有两种以上合理理解且会做出不同代码。
- 满足需求必须改变用户未授权的业务语义或扩大改造范围。
- 测试或审查修了 3 次还不过。

其余情况自己定，标 `[AUTO-ASSUMPTION] {假设}` 继续，收尾时汇总给用户校验。

## 核心原则

| # | 原则 | 要点 |
|---|------|------|
| 1 | 按任务决定委托 | 常规任务主会话直接完成；需要独立评审时使用新子代理上下文，见 `shared-rules/subagent-budget.md` |
| 2 | 统一审查 | 必修级高置信度问题修完才放行；建议级只记录。对 reviewer 的每条意见先评估技术合理性，reviewer 缺上下文时附理由推回，不盲改 |
| 3 | 按风险验证 | 有 plan.md 时读 `verification.type`；没有时按改动选实际验证，不默认强制 TDD。类型分发见 `shared-rules/verification-dispatch.md`，低影响可逆改动不为形式新增测试 |
| 4 | Locked Decisions 不静默偏离 | 有任务 workspace 时读 session.yaml 的 `locked_decisions[]` 和 `constraints[]`。偏离先核对证据并回写受影响文档；遵守当前指令层级，旧 CLAUDE.md 和 skill 不获得额外优先级。超出用户授权的业务语义变化才需澄清 |
| 5 | 并发 | 仅委托互不依赖任务；同文件任务串行；并发不超过当前实际剩余槽位 |
| 6 | 审查后交付 | 用户授权提交时由主会话提交；不自动创建 commit、保存点 tag 或推送 |
| 7 | 上下文合约 | 派发必须填 `UPSTREAM_GOAL` / `OUT_OF_SCOPE` / `WORKING_DIRECTORY` / `PROJECT_STRUCTURE` / `EXISTING_PATTERNS` / `DEPENDENCIES` / `CONVENTIONS`，只传任务描述会被子 agent 拒绝。CONVENTIONS 现场从 `cc-java-backend/standards-core.md` 提取原文，不凭记忆手抄 |
| 8 | 失败处理 | 根据失败证据修复，重复三次仍是同一阻塞时报告缺口；保留工作区，不自动回退或覆盖用户改动 |
| 9 | 存量行为保护 | 先复用现有配置和分发机制；确需双路才走开关分支，不自动新增开关或名单；见下方「改动策略预检」 |
| 10 | 最小必要改动 | 先读 [最小改动规则](../shared-rules/minimal-change.md)；新增防御或包装须有实际调用链与错误证据，测试失败不扩大业务范围；派发时传入适用约束 |

## 派发对账契约

子 agent 假完成、停在探索中段、谎报「已更新」在 2026-07 前后多个会话反复出现，所以派发和收账都靠结构化返回，不靠自觉。

- **锚点前置**：派 Implementer 前 Lead 自己定位精确插入点（file:line 或唯一 grep 锚点）喂进 prompt。同一 implementer 三次探索耗尽返回半成品，喂锚点后一次成功。复杂测试构造（PowerMock / 静态字段初始化）不派，Lead 自写。
- **prompt 末尾三句**（`prompts.md` 第一章末尾已带，Lead 填 N：定位类 ≤5，实现类 ≤15；派其他子 agent 时逐字复制）：

  ```
  探索预算：最多 N 次工具调用内定位不到目标就停止探索，返回已知信息+缺口清单求助，禁止空转耗尽轮次。
  完成后必须返回 claims 清单：[{file, line, change: 一句话改了什么, verify_cmd: 一条能证明该改动存在的 grep/test 命令}]。
  探索完即收尾出报告，不中途停下等追问。
  ```

- **claims 字段**：`file` 仓内路径；`line` 落点行号（±10 容差）；`change` 一句话；`verify_cmd` 可直接执行的命令（`grep -n "片段" file` / `mvn test -Dtest=X` / `node --check`），跑通即改动存在。无产出时返回 `claims: []` + 缺口清单，不用「已完成 / 已更新」叙述代替。
- **收账**：Lead 先检查 verify_cmd 的范围与副作用，只运行本任务已授权的验证命令；核对实际 diff 和行为，命令失败时区分实现问题与环境缺失。「X 不存在 / 没有调用方」类断言需核对完整上下文；无产出只催一次，再无产出 Lead 接手。
- **适用范围**：任何拿到 Edit / Write 权限的子 agent 都要 claims；只读探索类 agent 返回 file:line 结论时也给 verify_cmd；纯汇总 / 翻译类 agent 豁免，但 Lead 仍 diff 核对。派产出人读报告的子 agent 时，把 `shared-rules/style-appendix.md` 的拼贴块整段加进 prompt 末尾。
- **子 agent 返回摘要 ≤2k token**：细节写外置文件，主会话只看索引，否则后续审查阶段没上下文空间。

## 接活前提

有 `4-Context.md`（团队四阶段文档的末阶段指针文档，约定见 `shared-rules/document-stages.md`）时先 Read 它，再按它的指针读 2-技术方案 / 2.5-详细设计 / 3-Plan 的必要章节，不整份读 Spec。没有 4-Context 时按用户描述和代码现状做，不为要文档停下。

动手前先核工作区实态：`git branch --show-current` + `git diff <目标文件>`。用户 IDE 并行操作是常态，既有修复可能被覆盖，分支可能被切走（见 Gotcha 4）。

## 改动策略预检（写实现前第一道）

改业务逻辑前，读取 [最小改动与业务配置约定](../shared-rules/minimal-change.md)，按其中的配置复用、职责区分和验证要求执行；委托时传入本任务适用的约束。

目标文件不是本会话新建的（`git log --oneline -3 <file>` 有历史，或用户提过「线上 / 在用」）时保护存量行为。确需双路且现有机制不足的原因和最小方案已说明时，再采用老路不动、新能力走新分支和开关切换；不为灰度、预览或假设风险自动新增配置、开关或分支。以下四条自检仅适用于已确认需要的双路改造。重写老路仍需用户显式授权；「用新架构做」「支持一下 X」不算。

四条自检（违反任一就是擅自重写，回退成双路并告知用户）：

1. `git diff <老路文件>` 里老路函数体无改动，只在函数开头多一行 fork。
2. 老路的位置参数 / 返回类型 / 单例 / 导出符号没变（末位追加默认为空的可选参数不算，但要点名）。
3. 没把老路抽进新的公共基类 / 公共工具。这是最常见的伪装形式。
4. 老路的 if-else / switch 分支条件一个字没动。往老条件挂 `||`、往老 if-else 链插 else-if 都算：diff 只有几个字符，改的却是老路判定逻辑，回归面是老功能全量。

五类直接改老代码不走双路：修 bug、安全补丁、依赖升级、补有证据的 null 守卫与前置校验、加本任务必要的关键日志。豁免双路不授权预防性补守卫或顺手扩改。

四条判据管不到的三处另外查：老路依赖的共享模块、配置装配层、表结构（老路 SQL 的 WHERE 没带新维度会读串数据）。命中了与四条判据同级。

违反后分档：阻断 = 改到老路判定逻辑或数据形状；点名不阻断 = 向后兼容且回归面可枚举；不报 = 命中五类豁免。确需双路时按 `shared-rules/legacy-change-policy.md` 选择老路逻辑实际所在层并核对绕过该层的入口；搬运基于动手前实际文件快照，不用 HEAD 覆盖已有未提交内容。

> 触发识别表 / 开关分层 / 代价告知 / 真实反例见 `shared-rules/legacy-change-policy.md`，派 Implementer 前读，命中项提取进 CONVENTIONS。

## 骨架确认（预检之后、TDD RED 之前）

写实现前先把结构定下来：类怎么建、方法签名、注解、调用谁。方向错在骨架层纠偏只花一轮对话，写完实现再纠偏返工的是整段代码。

| 档位 | 判定 | 做法 |
|------|------|------|
| 文本档 | ≤2 文件且 ≤3 个方法落点 | 回复里贴骨架代码块，不等确认直接派发；骨架原文填入 Implementer prompt 的 `SKELETON` 字段。用户有异议时改骨架重派 |
| 落盘档 | 超过文本档阈值 | 仅当用户要求骨架评审或确实缺少必要设计决策时先交付骨架；已有实现授权则直接完成可审查实现，不自动提交占位代码 |

落点数：有 `2.5-详细设计.md` 按其落点矩阵数；没有就按任务预估，拿不准按落盘档。plan.md 已含代码骨架或 2.5 已过设计评审、且现场骨架与之零冲突时，只贴 FYI。

骨架写法：新增类写全 package / import / 类注解 / 字段注入 / 方法签名，方法体统一 `throw new UnsupportedOperationException("TODO")` 加一句「这里怎么改 / 调用谁」，保证可编译；改存量类老代码一行不删，在改动位插 `// [骨架] 此处将改为...`，改方法签名时注释里贴目标签名整行。有 2.5 时骨架从它的落点矩阵转译，冲突以 2.5 为准（2.5 有误先回改 2.5）。

已授权产生的骨架需要调整时仅改本任务文件；不因骨架评审自动执行 reset、revert 或改写历史。

收尾时 Lead 跑 `grep -rn '\[骨架\]\|UnsupportedOperationException' {目标文件}` 须零命中。

豁免：Bug Fix（失败测试已锚定行为；修复中新建类或改公开方法签名时豁免失效）、纯数据类（DTO/Config/Entity/Enum）、纯配置修改、用户明说不用看骨架。autonomous 模式不豁免，骨架照常产出并记入 progress.md。

## 精简梯（写实现前过一遍，与 YAGNI 联动）

> 认知锚：你是改过 50 行别人代码改成 1 行的资深工程师，见过"提前抽象"和"过度防御"两边都翻车，默认按梯子从上往下找最简可行解；但绝不**为了短删 validation / error handling / security / accessibility**。

七级决策梯和不可删 guard 清单在 `prompts.md` 第一章「Simplification Ladder Gate」，随派发提示词发给 Implementer，reviewer 也按它查「该爬上去了没」。Java must-have 条件守卫表、CSV 多值判定反模式、共享方法自守、0 问 / 3 问 over_eng 框架、小工具类豁免在 `references/simplification-ladder.md`，派 Implementer 前 Read 它，按任务命中项提取进 CONVENTIONS。

涉及模块、接口或辅助方法取舍时，按 [模块与封装的价值判断](../shared-rules/module-design.md) 核对真实调用方的知识与协调成本；保留有实际职责的短封装，不因行数或单次调用直接内联。委托时把适用判据一起传入。

**YAGNI 脚本**（新增字段 / 接口时，GREEN 实现前跑）：

```bash
bash /Users/caochen/IdeaProjects/ai/my_claude_gpt/.agents/skills/shared-rules/scripts/check-field-no-reader.sh --root <project> --fields field1,field2   # 字段无读取点
bash /Users/caochen/IdeaProjects/ai/my_claude_gpt/.agents/skills/shared-rules/scripts/check-interface-impls.sh --root <project> --interfaces IFoo,IBar # 仅本次接口的实现数候选（@FunctionalInterface 豁免）
```

字段脚本 exit 2 / 3 表示全部/部分字段无可见读取点，核实际消费者及脚本未覆盖的 Mapper/框架用法后再处理。接口脚本 exit 2 仅表示实现数候选，结合真实变体、明确替换/测试需要或框架契约判断；不要凑第二个实现、伪造调用或乱加注解。接口脚本 exit 3 是未发现匹配接口，不是缺陷；exit 1 是执行/参数失败。确认无用时才删除本轮新增内容，保留本次范围外代码。

## 并发执行

按依赖拓扑分组，同文件任务串行，并发受当前槽位限制。`workflows.md` 和 `workflows/execute-batch.workflow.js` 提供任务与汇总契约参考；Claude 派发参数、自动提交/tag、回退和 Workflow 调用不在 Codex 执行，按本入口和运行约定处理。

## 验证分发

G6：PLAN 时每任务定 `verification.type`（8 选 1，`shared-rules/verification-dispatch.md` 第一章），skip 要实质理由。G7：任务声明完成前 `verification.evidence_ref` 指向真实验证产物，写相对 workspace 的文件路径（如 `tdd-evidence/T1.1.2-green.log`），不写「测试通过」。

先验证行为再审查代码质量。验证证据写当前 workspace；不创建 `.claude/tdd-session-active`，不依赖旧 TDD Hook。

## 收尾门：/cc-review

编码与验证完成后审查本次 diff，用户要求独立复审或风险较高时读取 `../cc-review/SKILL.md`。已有审查覆盖同一 diff 时不为流程重复审查；提交不作为审查前提，简化不扩大本次范围。

| 变更规模 / 风险 | 动作 |
|--------------|---------|
| 涉及 money / 事务 / 并发，或 >300 LOC | 完整 `/cc-review` |
| 100-300 LOC | 标准 `/cc-review` |
| <100 LOC | `/cc-review --only-simplify` |
| 纯文档 / 配置变更 | 跳过 |

autonomous 模式自动触发：必修级回修复循环，建议级记录。触发时输出 `🔗 cc-code-writer → cc-code-reviewer | 原因: 开发完成收尾门 (变更 N LOC / [深度])`。并发模式汇报末尾提醒「请手动 push 或创建 PR」，本 skill 不 push。

## 协作关系

- 接收自 user、cc-planner（planner 传来的任务清单带 `skip_analysis: true`，跳过拆解直接执行；字段见 `shared-rules/data-contracts.md`「planner → code-writer」）。
- 内置审查用 cc-reviewer 子 agent，遵循 `cc-code-reviewer/rules.md` 的判据但不激活 cc-code-reviewer skill。
- 联动 cc-tdd（type=unit_test 时用其 test_first 决策表）、cc-integration-test（接口层验收）。
- 隐式依赖 cc-java-backend：派发前 Read `standards-core.md`，按任务相关章节（命名规范 / 可读性模式 / public 方法注释规范 / 类·接口 Javadoc 规范 / `catch (Exception e)` 使用指南）提取原文填入 `{EXTRACTED_CONVENTIONS}`。
- 子 agent 不再调用本 skill。

## Gotchas

| # | 陷阱 | 后果 | 做法 |
|---|------|------|---------|
| 1 | 子 agent 不继承对话历史 | 生成无关代码 | prompt 里传完整需求、文件路径、相关代码片段 |
| 2 | 依赖旧 TDD Hook | Codex 不执行原提醒流程 | 直接核对测试与真实证据，不创建旧会话哨兵 |
| 3 | CR-only（`\r`）或 minified 单行文件 Edit 反复匹配不中 | 工具按 LF 切行，整文件视为一行 | 先 `cat -A` / `file` 确认行尾；用 Python 字节级 `open(...,'rb')` + `replace(b'old', b'new')`，不整文件重写换行符 |
| 4 | 长会话改码前没核工作区实态 | 既有修复被用户 IDE 操作覆盖（vipship：Configs.java:1017 字段初始化被抹掉致门控永关）；分支被切走（身处 feature/REQ-35026 而目标 caauth 目录不存在） | 动手前 `git branch --show-current` + `git diff <目标文件>`；发现修复被覆盖就恢复并告知用户，不静默重做 |
| 5 | 直接调用 Claude 派发参数 | 当前工具不接受 | 读取当前子代理工具 schema，不调用 Task / TeamCreate |
| 6 | 按旧 sandbox 错误判断 worktree 不可用 | 错过当前可用的隔离方式 | 先检查当前权限和 worktree 能力，失败时报告真实错误 |
| 7 | 本地 commit 当成已同步远程 | 以为代码已备份，实际只在本地 | 汇报末尾提示手动 push |

## 资源加载

- 单任务 / Bug Fix / 并发 / 审查结果处理：`workflows.md` 对应章。
- 派发 Implementer 读 `prompts.md` 第一章，派发 Unified Reviewer 读第二章。
- 改存量代码读 `shared-rules/legacy-change-policy.md`；派发前提编码守则读 `references/simplification-ladder.md`。
- 需求模糊读 `shared-rules/inquiry-protocol.md`；子 agent 预算读 `shared-rules/subagent-budget.md`。
