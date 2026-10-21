# Code Reviewer 审查规则

先读 [最小改动与防御证据](../shared-rules/minimal-change.md)：结合完整方法和真实调用链确认错误；假设未来调用方失去过滤、代理失效或发生未证实风险，不足以要求新增防御包装。审查建议不授权扩大实现范围。

> 只写团队特有的规则：严重度和打分机制、内部框架约定、和通识相反的团队选择、带日期或 case 的踩坑。通识部分（SOLID、OWASP 注入、缓存三问题、方法长度、Given-When-Then）按模型默认判断，不在这里展开。

---

## 目录

- [一、严重度与打分](#一严重度与打分)
- [二、改存量代码](#二改存量代码)
- [三、架构合规](#三架构合规)
- [四、数据完整性](#四数据完整性)
- [五、异常处理](#五异常处理)
- [六、性能：N+1 与其余](#六性能n1-与其余)
- [七、设计模式、安全、单元测试](#七设计模式安全单元测试)
- [八、可读性模式](#八可读性模式)
- [九、误报清单](#九误报清单)
- [十、反向验证与建议级自检](#十反向验证与建议级自检)

---

## 一、严重度与打分

**内外语言分离**：本文件和开发者写规则时用 P0-P5 做 ranking；对外报告不出现 P 编号，用桶名（`❌ 必修` / `⚠️ 待确认` / `✅ 通过`）加维度标签（`[安全]` / `[数据完整性]`，不带 `D1` / `(P0)` 后缀），建议级条目（仅 full 模式）前缀 `Nit:` / `Suggestion:` / `FYI:`。P 编号像 ticket 项目管理术语，制造决策疲劳（2026-05-22 用户反馈）。

| 优先级 | 类别 | 描述 | 阻断 |
|------|------|------|------|
| P0 | 安全 / 数据完整性 / 业务行为与流程顺序 | 安全漏洞、数据泄露；事务缺失、数据不一致；验收标准未实现、操作顺序错 | 是 |
| P1 | 架构合规 | 分层违规、依赖反转、注释规范 | 是 |
| P1 | N+1 | 本次变更新写的循环内单点 IO | 是 |
| P2 | 单元测试 | 断言缺失 / 质量差 | 否，默认丢弃 |
| P3 | 性能（其余） | 无分页、无缓存、未分批 | 否，默认丢弃 |
| P4 | 设计模式 | 可用更好的模式 | 否，默认丢弃 |
| P5 | 代码简化 | 可读性、重复消除 | 否，默认丢弃 |

**默认输出**：只输出 P0 + P1，P2-P5 在阶段 10 自检后丢弃，不出现在报告里（包括 `<details>` 折叠区），只留一行 `nit_dropped: N`。建议级问题全不修也不影响合并，强制输出只制造决策疲劳。用户说"全量审 / 详细看一下 / `full_review=true`"，或 planner REVIEW 的 QA 带 `with_scoring` 时才输出 P2-P5。

### 置信度 0-100 打分

每条 finding 三证据项累加，阈值 80。机制照搬官方 Claude Code code-review 的 verification step 加量化阈值。

| 证据项 | 分值 | 判定 |
|--------|------|------|
| 能指向具体代码行 | +40 | 有 `file:line` 锚点且原文可 `<quote>` 逐字引用 |
| 能解释为什么是错的 | +35 | 说得清触发条件和后果，不是"看起来怪" |
| 有规范 / 文档 / 测试佐证 | +25 | 有 CLAUDE.md / SKILL / 测试 / 官方文档背书 |

| score | 桶 |
|-------|-----|
| ≥80 | 必修维度进 ❌ 必修；建议级维度默认丢弃，full 模式输出 |
| 40-79 | ⚠️ 待确认：有代码锚点但缺佐证或有合理替代解释 |
| <40 | 丢弃，折叠区 `[REMOVED:score<40]` 留痕 |

**反证扣分**：打分时按第十节「反向验证」四项找"为什么这不是问题"的证据，命中即扣（佐证消失 -25 / 原因被推翻 -35），跌破 80 移出必修桶。扣分要引出反证所在的 file:line 和原文，"看着没问题"、"无法确认"不算反证，拿不出就不扣。

**三类主题受保护**（空指针与越界 / 并发 / 行为或兼容性变化）：先过第九节误报清单，没命中且有代码行和原文的，最低进 ⚠️ 待确认。

---

## 二、改存量代码

先核现有配置和分发是否已满足需求；本节双路检查只用于已确认需要双路的改造，不因未新增开关或 fork 判缺陷，不要求为灰度、预览另建配置或名单。

diff 里有非本次新建的文件时，先判作用域再定级别，不能跳，否则每个 diff 都会报成必修：

- ❌ 必修：老函数签名被改 / 单例被删 / 老路被抽进新公共基类 / 老路的 if-else、switch 分支条件被动过（含往老条件挂 `||`、往老 if-else 链插 else-if）/ 老函数体被改写成"顺便支持新场景"
- 🟢 一行提示不阻断：1-2 行的 null 守卫、日志、typo、注释
- ⬜ 不报：命中 `shared-rules/legacy-change-policy.md`「触发识别」列的豁免：修 bug / 安全补丁 / 依赖升级 / 补前置校验 / 加关键日志

另查三处判据管不到的面：老路依赖的共享模块、配置装配、表结构。全文见 [`shared-rules/legacy-change-policy.md`](../shared-rules/legacy-change-policy.md)。

---

## 三、架构合规

### 3.1 分层与包结构

```
Controller  ->  只注入 Service（不注入 Repository / Mapper）
Service     ->  只注入 Repository、其他 Service（不注入 Mapper）
Repository  ->  继承 BaseRepository，使用 baseMapper
```

Controller 只做参数接收、校验、委托 Service、包装返回。出现 if-else / switch 业务判断（参数校验除外）、循环处理数据、编排多个 Service、数据转换聚合，判职责越界（P1）。

| DTO 类型 | 用途 | 所在包 |
|------|------|--------|
| Req | Controller 入参 | `common/req/` |
| Resp | Controller 出参 | `common/resp/` |
| Bo / Dto | Service 之间传递 | `common/bo/` / `common/dto/` |

包名用 `cn.caijiajia.*`，不用 `com.caijiajia`。

依赖方向（P1）：Repository / Service 不用 Controller 层类型（Req、HttpServletRequest）。在 Service 入口把 Req 转成 Bo 下传，下层签名只出现 Bo / 实体 / 基本类型。DIP、循环依赖按通识判 P1；SRP、OCP 建议级，if-else / switch 超过 5 个分支才建议策略模式。

### 3.2 注释规范（P1）

public 方法（含接口方法）要有 Javadoc，缺少或写入下列内容即阻断。

**下限**：功能描述说业务意图（`/** 获取用户 */` 对 `getUser()` 不合格）+ `@param`（只写有增量信息的参数：字段映射 / 单位 / 约束 / 可空语义；参数名加类型自明且无约束时省略，`@param reqDTO 出网请求` 这种参数名翻译判违规）+ `@return`（非 void 且有陷阱时）+ `@throws`（按异常类型加 HTTP 状态码分行，不罗列 `RuntimeException` / `NPE`）。单位字段明写 `分` / `毫秒` / `秒`。

**上限（不写进注释）**：

- 设计文档章节号（`见《XX 设计》§4.5 Step 0-8`）、`vault://` KB 路径、`@see <a href="jira">`、`参考: XX 文档`、`见 README / wiki`：链接会腐烂（KB 迁移、JIRA 号重排），背景去看 commit / PR。唯一例外：`@deprecated` 带 `{@link #替代方法}`
- 实现步骤顺序（`Step 1 必须早于 Step 2`）放方法体行内注释；修复背景（`避免 T1 灰度期 XX 噪音`）、调用者信息（`用于 XX 流程`）放 PR 描述
- 评审过程元数据：`W*` / `P*` / `D*` / `REV-*` / `针对 Wn xx 关切` / 行内 `+ Wn 防xx`，这是评审会话的临时标签，不是代码语义
- 设计 / 测试文档编号（`N-1`、`Q7`、`U-12`、`P-01`、`IT-05`）和「详细设计 04 章」、「共 8 个接口」这类会过期的断言，翻成人话；作为数据写库的常量值、测试方法名和 `@DisplayName` 里的用例号例外。机检：`scripts/comment_lint.py`（2026-10-03 用户固化）
- 「设计要点 1/2/3」长段列表：Javadoc 只讲调用者会用的，设计权衡放设计文档或 PR
- emoji / 装饰符号（🔴 ✅ ❌ ⚠️ 🔑 →）一律不出现，强调用文字（2026-08-03 用户固化）
- 行内注释连续 ≥2 行 `//` 讲同一件事：压一行、拆方法或移 PR 描述
- 方法 Javadoc `<p>` 论证段 ≥2 个即超标；契约陷阱压成一句，并列要点用 `<ul><li>` 一行一条（2026-08-19 用户固化）
- 字段 Javadoc 硬上限 2 行，只写值语义和取值陷阱（`null` 等价什么、单位、默认值）；类型选型论证、灰度 / 下线 / 迁移计划、对拍方案、`<p>` 段全不写（对齐 `cc-java-backend/standards-core.md`「字段 Javadoc 规范」，2026-08-20 用户固化）

**豁免**：getter / setter、`toString()`、`equals()` / `hashCode()`、`@Override` 简单委托、MyBatis Mapper 接口方法（Javadoc 降为 P5 建议）。

**判定表**：

| 条件 | 置信度 | 优先级 | 处理 |
|------|--------|--------|------|
| public 方法完全无 Javadoc | high | P1 | 补 |
| 参数 / 返回值有约束（单位、可空、范围、字段映射、"外层 ok≠业务成功"类陷阱）却无对应 `@param` / `@return` / `@throws` | high | P1 | 补该行 |
| `@param` / `@return` 零增量（只把参数名翻译一遍） | high | P1 | 删该行，不是补内容 |
| 类 / 接口 Javadoc >3 行，或含 `<p>` 叙事段，或复述注解已表达的事实（`@FeignClient` 的 path、`@Service` 的分层） | high | P1 | 压到 3 行内：只留"存在理由 + 必踩的坑" |
| 同一条契约陷阱在类注释与方法 Javadoc 各写一遍 | medium | P1 | 只留离调用者最近的那处 |
| 为省行数折叠 Javadoc 块结构（多行压单行 `/** xxx */`、删 `/**` `*/` 独立行、`@` 标签并进描述行） | high | P1 | 恢复多行块；"≤N 行"数的是内容行 |
| 中文注释块内 `{@code}` + `<b>` + `<p>` 合计 >2 处 | medium | P2 | 裸词能表达就不包标签；`{@link}` 指真实类型除外 |
| 描述无业务含义（方法名重复） | medium | P1 | 补 |
| 含设计文档章节号 / 外部跳转引用 / 评审编号 / 设计测试文档编号 / 过期数量断言 | high | P1 | 删或翻成人话 |
| 「设计要点 1/2/3」列表、≥2 个 `<p>` 论证段、字段 Javadoc >2 行 | high | P1 | 移 PR 描述或设计文档 |
| 写入实现步骤 / 修复背景 / 调用者 | high | P1 | 移方法体或 PR 描述 |
| 含 emoji / 装饰符号；行内连续 ≥2 行讲同一件事 | high | P1 | 删符号 / 压一行 |
| `@throws` 罗列 RuntimeException / NPE | low | P3 | 建议删 |
| private / protected 方法无注释 | low | P5 | 建议 |

### 3.3 方法命名

通用命名问题（`And` / `Or` 连接、`get` 返回 Optional、`find` 抛异常、布尔双重否定、动词空洞）按通识判 P2-P3。团队额外一条（P2）：**相对修饰词代替业务意图**，`shouldUseNewLogic` / `processV2` / `handleLegacyFlow` / `applyTempFix` 里的 New / Old / V1 / V2 / Legacy / Temp 是开发者视角的相对坐标，今天的 New 是明天的 Old。改用触发该分支的业务条件命名（`shouldApplyTieredPricing` / `handlePreMigrationOrder`）。

### 3.4 抽象充分性（P1）

涉及模块、接口或辅助方法取舍时，按 [模块与封装的价值判断](../shared-rules/module-design.md) 核真实调用方的约定、职责归属和变更/验证成本，不以行数、调用次数或实现数直接裁决。

- 反重复管道：重复比例只作线索；核实两条实际业务路径是否共享同一规则、是否同频变化，以及抽提后是否减少调用方协调。没有真实问题不要求提取基类/钩子。有意保留的老路副本按 `legacy-change-policy.md`，不为消除重复扩大改造范围。
- 职责归属：不同调用者是检查线索；只有无关业务职责混杂、约束难以隔离等具体问题才提出拆分，不能仅因 public 方法的调用者不同就判缺陷。
- 新供应商扩展：本任务确有第三个实现时，比较新增职责和共享约定；假想未来实现或复制行数不构成当前抽象不足的证据。建议级取舍不升格成必修，确认实际契约/行为缺陷后才按影响定级。

---

### 3.5 业务配置来源与作用范围

按 [最小改动规则的业务配置约定](../shared-rules/minimal-change.md) 核对配置来源、三种职责和验证证据；重复来源的处置沿用该规则，不另立一套判据。确认违反需求或既有配置契约后按具体行为影响定级，证据不足不升格为必修；未运行的场景如实标明。

## 四、数据完整性

事务、锁、竞态、状态机按通识判 P0。团队约定和场景区分：

- 多表写入用 `@Transactional(rollbackFor = Exception.class)`，裸 `@Transactional` 只回滚 RuntimeException；本类调用事务方法走 `AopContext.currentProxy()`
- 状态变更先过 `canTransitionTo()`，非法转换抛 `CjjClientException`
- 事务内触发异步用 `TransactionSynchronizationManager.registerSynchronization` 的 `afterCommit()`
- **单数据源 + Spring REQUIRED 传播**：实查外层事务回滚已覆盖内层操作且满足当前原子性契约时，不判缺少事务，不为补本层注解或代理包装另报架构问题
- **跨数据源 / 跨服务 RPC**：外层事务回滚不了远程操作，判 P0 需补偿
- **幂等替代锁**：「检查 + 操作」的幂等场景用 SET NX EX 比分布式锁轻，不判"缺少并发控制"

### 4.1 业务行为与流程顺序

对照需求或设计文档的验收标准逐条核：缺失或只实现主路径没有异常路径判 P0；文档标注"二期实现"的不算缺失。错误信息面向最终用户时要含错误码、描述、修复建议（面向内部调用方可简略）。写操作按"验证 → 操作 → 通知"顺序，通知在事务提交前发出、状态跳跃转换判 P0。查询操作不查顺序；状态机由独立 Bean 统一管理时只查该 Bean 是否被调用。

三条和全局 CLAUDE.md「共享方法自守完整前置条件」同源的规则：

- **参数传播完整性**：调用下层方法或构造 DTO 时，对传 null 或未赋值的字段核完整使用上下文、可空契约和已有校验；只有实际可达的错误使用才按影响定级，不能仅因出现 getter 或条件判断就判缺陷、要求新增守卫
- **信任边界**：按实际输入、可空契约、安全边界及已有校验判断是否缺少保护，指出可达错误路径。public / 共享可见性不能单独证明缺守卫；已有可靠校验覆盖当前路径且契约未要求本层再校验时，不因假设调用方未来取消过滤判 bug
- **字段值的上下文唯一性**：子类型 / 子状态字段（`equity_type`、`sub_status`）取值只在父类型上下文里唯一时，守卫父子条件一起判（`productClass == X && equityType == Y`），单判子字段会跨父类型误命中

---

## 五、异常处理

### 5.1 `catch (Exception e)` 泛化优先

和通识相反的团队选择：**不把"无具体类型在前"当反模式**。审查只看吞异常 / 掩盖金额 / 转译丢 cause / 中断位丢失四类真问题。规则原文见 `cc-java-backend/standards-core.md`「`catch (Exception e)` 使用指南」。

三个必要条件（缺一即警告）：异常信号有下游消费（throw / 分支变量 / 结构化记录加告警）；`throw` 业务异常带 cause；catch 后不 return 默认值掩盖金额或核心状态。

三个技术例外要具体 catch：`InterruptedException` 要 `Thread.currentThread().interrupt()` 恢复中断位；`TimeoutException` 与业务异常要区分 504 / 500 时；`JsonParseException` / `IllegalArgumentException` 要区分 400 / 500 时。

| 条件 | 置信度 | 优先级 |
|------|--------|--------|
| 空 catch | high | P1 |
| catch 后返回默认值掩盖金额 / 核心状态（余额 return ZERO） | high | P1 |
| 只 log 不抛 + 下游无分支消费 + 流程直通 | high | P2 |
| 转译丢 cause（`throw new Xxx(...)` 无 e） | high | P2 |
| 硬包 RuntimeException（丢业务语义 / HTTP 状态码错） | high | P2 |
| 可中断操作未单独 catch `InterruptedException` | high | P2 |
| 只 log + 下游有分支消费（flag / null / 状态被读） | medium | P3 |

合法场景：外部调用兜底转译为 `CjjServerException` 带 cause；批量任务 `catch` 记 `failedList` 后 `alertService.notify(failedList)`；异常作分支信号（`needFallback = true` 后续 `if` 消费）；JSON 解析后 `throw new CjjClientException(400, ..., e)`；finally 清理不掩盖主异常。

### 5.2 业务异常降级（P1）

项目统一业务异常是 `CjjClientException`（客户端）/ `CjjServerException`（服务端），不替换或降级成 `IllegalArgumentException` / `IllegalStateException` / `RuntimeException`。JDK 异常无 `code`，前端按统一 `code` 分支的处理拿不到可操作信息；全局 `@ExceptionHandler` 按业务异常类型映射 400 / 500，JDK 异常落兜底分支统一 500。

尤其不以"绕过 import 缺失 / 编译错误"为动机降级：编译失败是缺 import，补 `import cn.caijiajia.xxx.CjjClientException`，用项目真实错误码抛业务异常。diff 同时删 import 加换异常类型即判 P1。

合法场景：工具类 / 底层库（无 web 上下文）；`Objects.requireNonNull()` / `Preconditions.checkArgument()` 防御性断言；第三方回调强制要求 JDK 异常类型（`Comparator` 约定）。

---

## 六、性能：N+1 与其余

### 6.1 N+1（唯一进阻断桶的性能问题）

判定：循环体内出现单点 IO，单点查库（`selectById` / 单条 Condition 查询）、单点调 RPC / Feign / HTTP、单点查 Redis 都算，`for` / `forEach` / `stream().map()` 不限。

先判作用域再定级别，否则会把整个存量文件报成必修：

| 情况 | 级别 |
|------|------|
| 本次新增的循环带单点 IO | ❌ 必修 |
| 已有循环，本次往循环体里加了单点 IO | ❌ 必修，在错的地方继续加错的更该拦 |
| 本次重写 / 搬迁了这段方法 | ❌ 必修，算本次写出来的 |
| 循环在 diff 之外，只是同文件别处被改 | 💬 一行提示 |
| 本次只改了循环内的无关行（日志、字段名） | 💬 一行提示 |

存量 N+1 报告末尾一行汇总，不逐条展开、不进桶：`FYI: 存量 N+1 3 处（UserService.java:88 / OrderService.java:142,209），本次未改动，建议单独排期改批量。` 存量是历史债，本次 PR 不买单，但读到了不说等于默许。用户明确要求"改存量"才逐条展开。

提到必修级的原因：耗时和下游压力随数据量线性放大，测试环境十几条看不出来，生产上量直接超时或打爆下游，上线后必然返工。

```java
// ❌ 必修：N 个 id 打 N 次接口
for (String userId : distinctIds) {
    CollectionUserProfile profile = profileFromRuoyi(userId);
    if (profile != null) {
        profiles.put(userId, profile);
    }
}

// ✅ 循环外一次批量取，收成 Map，循环内只查 Map
List<CollectionUserProfile> list = ruoyiClient.listByUserIds(distinctIds);
Map<String, CollectionUserProfile> profiles = list.stream()
        .collect(Collectors.toMap(CollectionUserProfile::getUserId, Function.identity(), (a, b) -> a));
```

三种不算违规：循环体本身是 `Lists.partition` 分批后每批一次批量查；循环次数有硬上限且 ≤3（固定枚举、两三个固定渠道）；循环内取的是已加载到内存的 Map / 本地缓存。

下游没有批量接口不是放行理由，按序给出路：推动下游加批量接口 → 数据可缓存走缓存 → 都不行才用配置好的线程池并发调（不 `new Thread()`）并在方法注释写明原因。改法模板见 `cc-java-backend/standards-core.md`「批量操作与 N+1 红线」。

### 6.2 其余性能（P3，仅 full 模式）

团队约定：超过 500 条用 `Lists.partition(list, 500)` 分批；列表查询 `PageHelper.startPage` 分页；不 `new Thread()` / `Executors.newCachedThreadPool()`，注入配置好的线程池；`@Async` 同类调用不生效要抽到单独 Service。缓存、SQL 索引、异步超时、TOCTOU、可并行未并行按通识判。

---

## 七、设计模式、安全、单元测试

- 设计模式（P4）：枚举要有 `fromCode()`，建议 `isTerminal()` / `canTransitionTo()`；if-else / switch 超过 5 个分支建议策略接口加自动注册工厂；参数超过 5 个用 Context 对象。
- 安全（P0）：按 OWASP 通识判。MyBatis 用 `#{}` 不用 `${}`；密钥从配置中心读；摘要 / 签名 SHA-256 以上，密码存储 BCrypt。
- 单元测试（P2）：断言核心业务字段，异常测试 `assertThrows` 验证状态码和消息。

---

## 八、可读性模式

下面的行数/条件数是候选线索，提取或内联按「3.4 抽象充分性」核真实职责；单次调用且职责明确的辅助方法不因调用次数少判多余。

> 模式定义和代码示例见 `cc-java-backend/standards-core.md`「可读性模式」。全部 P5，仅 full 模式输出。

| 模式 | 触发场景 | 判定 |
|------|---------|------|
| 提取查询方法 | Service 方法中内联 Example / Condition 构建 ≥3 行 | 封装到 Repository 语义方法 |
| Guard Clause | if-else 嵌套 ≥2 层且有可早返回的短分支（<5 行） | 翻转为早返回 |
| 语义辅助方法 | 枚举 `.name().equals(field)` 或 3+ 条件组合 | 封装为 `isXxx()` / `hasXxx()` |
| 翻转 if-else | if 短分支（<5 行）+ else 长分支（>10 行） | 翻转条件 |
| 正向条件优先 | `if (!x) doFallback() else { 主逻辑 }` / `!x ? a : b` / `while (!isDone)` | 翻转分支、用反义谓词、或换 `hasNext` / `isEmpty` 等正向 API |
| 参数最小化 | 方法同时接收 A 和 f(A)（`plan` + `buildUrl(plan.id, plan.code)`） | f(A) 下沉到方法内部 |
| 同质段抽提 | 方法体内连续 ≥8 行单一职责块（埋点 / 日志 / 审计 / 校验 / DTO 拼装），常伴 ≥3 次链式取值或 5+ 参数调用 | 抽成语义命名 private 方法（`logVpiAnomaly(bo)` / `buildRefundReq(bo)`） |

不标记：Condition 仅 1-2 行且无排序分页；两个分支都 >10 行翻转无收益；条件只出现一次且语义清晰；反向 `!` 用于 Guard Clause 早返回、没有现成反义谓词的单次使用（`!StringUtils.isEmpty(s)`）、测试 DSL 的 `assertFalse`；主流程本身就是连续 8+ 行 builder 链或 DSL、抽出后传参超过 4 个、一次性脚本。

---

## 九、误报清单

发现疑似问题先对这张表，匹配即跳过。

| 疑似问题 | 验证方法 |
|------|---------|
| 代码注释写着"SQL 注入"、"安全风险" | 追踪到实际 SQL 层：注释可能是历史遗留，代码已用 `#{}` / PreparedStatement。看被注释代码的实际执行路径，不凭注释定性 |
| 注释写"TODO: 需要加事务"但方法已在事务内 | 查调用链上层是否已有 `@Transactional`，注释可能没清理 |
| 注释写"不安全"但有运行时校验 | 查 AOP / Filter / Interceptor 层统一校验 |
| 缺少同步但无多线程调用 | grep 调用方有无 `@Async` / 线程池 / MQ 消费；Spring 单例 Bean 的成员字段、static 字段默认视为有多线程访问 |
| 外层方法缺 `@Transactional` | 查调用链，上层可能已有事务 |
| 字符串用 `==` 比较 | 两端是 static final 常量或枚举时合法（枚举 `==` 是推荐写法） |
| `double` / `float` 计算 | 非金融场景的近似计算（百分比展示、日志统计）可接受 |
| 空 catch、Optional.get()、可能的空指针 | 查调用方是否已校验、注释是否说明有意忽略 |

---

## 十、反向验证与建议级自检

### 10.1 反向验证

每个疑似问题先找"为什么这不是问题"的证据：外层保护（事务、锁、校验在调用方已处理？）、配置兜底（框架级默认行为？）、代码路径（其他路径保证正确性？）、设计意图（故意如此？）。

public / 共享方法的外层保护同样按第四节「信任边界」核对；只有实际可达的未校验路径或明确的本层校验契约未满足，才保留 finding，不能仅假设调用方取消过滤。

### 10.2 建议级自检（仅 full 模式）

每条 P2-P5 建议要有代码锚点和可行性判断，否则不输出：

| 建议类型 | 空洞建议（不输出） | 有锚点的建议 |
|---------|-----------------|-------------------|
| 提取重复代码 | "L50 和 L120 有重复逻辑，建议提取" | "L50-58 和 L120-128 逐行比对，除变量名外一致，可安全提取" |
| 使用设计模式 | "建议用策略模式替代 if-else" | "5 个分支共享 (Order, Context)->Result 签名且按类型分派" |
| 拆分长方法 | "方法 60 行过长，建议拆分" | "L30-45 为数据校验、L46-60 为业务处理，可拆为 validateX() 和 processX()" |
| 减少嵌套 | "嵌套 4 层，建议卫语句" | "外两层为 null check 可早返回；内层是业务分支，保留" |
| 参数封装 | "6 个参数建议封装为 Context" | "6 个参数中 4 个属于订单信息且在 3 个方法间传递，适合 OrderContext" |

通过自检且收益真实的标「建议优化」并附验证依据；有锚点但收益不确定的标「仅供参考」并附不确定原因；无锚点或不可行的丢弃。
