# 设计说明与扩展指南

## 设计意图

本 skill 支持两种证据驱动输出：业务视角将真实代码转化为可复述、可调试、可维护的业务故事；架构视角将源码/框架的入口、主干执行、核心抽象与扩展边界转化为可执行的阅读路径。两者都不是代码逐行转述。

## 维护铁律

维护本 skill 时，任何用户反馈必须先抽象为适用于任意语言、架构、仓库、业务或框架库的通用规则。禁止将某个案例的模块、字段、状态、目录、服务名、调用链或颜色组合硬编码进通用指令；具体报告只能作为回归样本，不能成为唯一适配目标。

实现上刻意分成三层：

1. `SKILL.md` 负责触发边界、仓库配置、代码来源/版本快照、自动检索和准确性闸门。
2. `references/` 负责会频繁扩充的报告详细度与视觉规范。
3. `templates/` 负责稳定的离线 HTML 骨架和交互行为。

这种拆分使主指令保持在可加载范围内，同时让报告章节和模板可独立演进。

## 工作流程

1. 仅响应显式 `$codebase-storyteller`。
2. 读取根目录的 `config.local.yaml`，按精确映射、多个仓库根目录、旧版单根目录的优先级解析仓库路径；不要把本机配置写回公开版 `SKILL.md`。
3. 先解析每个仓库的目标 ref 与当前 HEAD。调用未显式来源且两者 commit 不一致时，先询问读取工作区或 ref；`ref` 模式使用 `git show`、`git ls-tree`、`git grep` 等只读 Git 操作，`worktree` 模式直接读取当前磁盘文件。两种模式均不得切换或修改工作区。
4. 工作区模式记录每个仓库当前 checkout 标识；报告页首和跨仓库说明使用该版本标识，并声明行号与当前工作区一致。
5. 未指定视角时先询问选择业务或架构；业务视角根据业务名聚焦检索，或在未指定业务时从 API/MQ/task 等入口列出重要业务等待选择。
6. 架构视角根据目标聚焦公开入口、调用主干、类型实现、装配/注册和扩展点；未指定目标时列出真实核心子系统、主干流程或抽象等待选择。
7. 建立证据账本，包含仓库、来源、版本标识、文件、行号、符号、结论类别；多仓库时额外记录可验证的 import、依赖、替换、注册、实现、调用或改造关系。
8. 读取完整纳入范围后，按视角抽取业务状态/流程/字段，或抽取调用链/类型关系/数据结构/扩展机制；按仓库归属组织同名机制，不因名称相同合并结论。
9. 解析 `要求`，先完成不受其影响的视角骨架与质量计划，再只在允许范围内调节表达、重点或补充内容；记录采纳、受限调整与冲突拒绝结果。
10. 按 `references/report-spec.md` 的当前视角章节与门槛组织文字和图表。
11. HTML 模式用 `templates/report.html` 生成内联 SVG 报告；Markdown 模式生成 Mermaid。
12. 按 `SKILL.md` 的完成门槛检查，再覆盖写入 skill 内的 `reports/` 目录；不要求用户提供输出路径。

## 调用分流示例

业务视角：

```text
$codebase-storyteller
仓库：order-service
视角：业务
业务：库存出库
```

架构视角：

```text
$codebase-storyteller
仓库：orm-library
视角：架构
目标：查询执行主干
```

未指定 `视角` 时，只询问用户选择业务视角或架构视角，不先扫描代码。已选架构视角但未指定 `目标` 时，扫描真实公开入口、类型关系和注册/装配点，列出候选目标后等待选择；业务视角未指定 `业务` 时，沿既有业务候选流程等待选择。

来源示例：

```text
来源：工作区
```

显式工作区来源直接读取每个仓库当前 checkout 的文件。未指定来源且 HEAD 与目标 ref 不一致时，必须询问用户读取工作区还是 ref；`分支：<name>` 是显式 ref 模式的兼容写法。

要求示例：

```text
要求：面向新维护者，补充两个实现的差异对比，并优先解释错误处理
```

要求只控制表达侧重点和补充内容。真实性、当前视角骨架、方法五要素、阻断式质量门槛、多仓库归属、静态离线 HTML 与 deep link 规则始终优先；冲突内容必须忽略并在完成提示中透明说明。

## 编辑器导航

- 证据脚注只呈现 `文件:行号`，不承担跳转功能。
- 方法名或明确对象名本身以 `.code-open` deep link 承担代码定位，并由 `editor.uri_template` 生成。
- VS Code 与 Cursor 使用自定义 URI 和 `target="_blank"`；HTML 不显示 fallback 命令。
- JetBrains 不假定存在统一可靠的 URI；未提供用户验证过的 URI 模板时，不生成伪造链接。

## 分层检索扩展

新增语言时，优先补充“入口、核心、异步、外部、实体”的等价目录和符号线索，而不是只按文件后缀搜索。

## 架构视角检索扩展

架构视角优先沿可达性和抽象边界检索，而不是按业务目录命名猜测。对任意语言或框架，依次寻找：

1. 公开入口：导出 API、CLI、服务启动、路由/RPC 注册、事件订阅或库的构造入口。
2. 主干执行：入口调用的编排器、执行器、解析器、计划器、运行时或 I/O 边界。
3. 核心抽象：interface/trait/abstract class、泛型约束、基类、组合对象及其实现集合。
4. 装配与选择：DI/container、factory、builder、registry、configuration、feature flag 或默认实现选择。
5. 扩展机制：plugin、driver、hook、callback、middleware、adapter、visitor、interceptor 或等价机制。
6. 数据结构流转：request/options/context、AST/schema/plan、缓存条目、结果对象、错误对象的创建与传递。

只有代码存在这些概念时才写入报告；架构视角不得假设项目一定存在插件、ORM、缓存、schema、迁移或状态机。

### Go

- 入口：Gin/Echo/Fiber 路由、gRPC 注册、`cmd/`、handler。
- 核心：service/usecase/domain 方法、interface 实现。
- 异步：consumer、worker、cron、`go` routine 启动点。
- 实体：struct、gorm tag、repository SQL。

### Java/Kotlin

- 入口：`@RestController`、`@RequestMapping`、`@KafkaListener`、`@Scheduled`。
- 核心：`@Service`、domain/application 包、transaction boundary。
- 实体：`@Entity`、JPA repository、enum。

### TypeScript/JavaScript

- 入口：Express/Nest router/controller、queue processor、cron。
- 核心：service/use-case/domain modules。
- 实体：ORM schema/model、DTO、Zod/class-validator schema。

### Python

- 入口：FastAPI/Flask/Django views、Celery/RQ task、management command。
- 核心：service/usecase/domain package。
- 实体：Django/SQLAlchemy model、Pydantic model、Enum。

### Rust

- 入口：Axum/Actix routes、consumer/task runner。
- 核心：service/application/domain module。
- 实体：struct、enum、Diesel/SQLx query model。

### SQL

- 用迁移、DDL、view、procedure、trigger 作为实体关系、约束和字段默认值的辅助证据。
- SQL 单独不能证明完整应用流程，需回溯到调用 SQL 的应用代码。

## 图表扩展

HTML 最终图表统一内联 SVG，以保障离线可用。新增图类型时：

1. 先定义其必须包含的证据字段。
2. 在生成阶段把图预渲染为可缩放 SVG。
3. 使用已有 `.diagram-zoom`、lightbox、语义色和可访问性规则。
4. 不新增远程脚本、CDN 或运行时渲染依赖。

Markdown 可新增 Mermaid 图类型，但必须确认目标渲染器支持并避免把未知关系画成事实。

## UML 关系与设计亮点

- 类型关系图遵循 `references/visual-style.md` 的 UML 记法：泛化/继承和实现使用空心三角，组合使用实心菱形，聚合使用空心菱形，依赖使用虚线开箭头，关联使用实线箭头。每条边必须由代码证明，不从命名或目录推断。
- 只有明确的生命周期/所有权语义才可画组合或聚合；构造注入、调用或注册通常只证明依赖或关联，除非代码提供更强证据。
- `.highlight` 是可选设计亮点 callout，只在可回溯代码证据能解释设计取舍时使用。它不替代方法深挖或图表，不是品质或数量指标。

## 多仓库归属与关系

- 多仓库报告按 `仓库名 @ 实际版本/ref` 保留归属。总览、关系图和时序图用仓库边界、参与方标签和跨边箭头显式表达，而不是把组件混在同一层。
- 只有 import/依赖声明、构建替换、接口实现、注册、调用、生成契约或可验证改造证据存在时，才能描述仓库间的依赖、扩展、覆盖、fork 或上层关系。
- 底座、扩展、fork、改造版与上层仓库分别深挖；同名机制默认分开讲，只有共享实现或调用证据存在时才合并陈述。
- 单仓库不引入多仓库标签、边界图例或对比章节。

## 不可突破的边界

- 不编造业务规则、状态、调用、补偿或依赖。
- 不读取或写入目标仓库之外不必要的敏感文件。
- 不切换、重置、提交或修改用户的 Git 工作区。
- 不因图表排版方便而牺牲证据、可读性或中文文字详细度。
- 不因架构视角而缩减业务视角的 14 章节、方法五要素、时序图细节、ER 图或状态机真实性门槛。
- 不因来源模式、UML 关系或设计亮点而猜测代码关系、忽略版本差异，或给简单业务额外注水。
- 不因多仓库数量或要求而混淆仓库归属、合并不同版本机制、虚构关系，或削弱任何真实性与阻断门槛。
- 发布前检查公开仓库不含 `reports/` 下任何真实业务 `.html`/`.md`，也不含 `config.local.yaml`。
