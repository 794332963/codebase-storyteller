# 设计说明与扩展指南

## 设计意图

本 skill 的输出不是架构摘要，也不是代码逐行转述。它的任务是用证据驱动的方式，将一组真实代码转化为后端开发者能够复述、调试和继续维护的业务故事。

实现上刻意分成三层：

1. `SKILL.md` 负责触发边界、仓库配置、分支快照、自动检索和准确性闸门。
2. `references/` 负责会频繁扩充的报告详细度与视觉规范。
3. `templates/` 负责稳定的离线 HTML 骨架和交互行为。

这种拆分使主指令保持在可加载范围内，同时让报告章节和模板可独立演进。

## 工作流程

1. 仅响应显式 `$codebase-storyteller`。
2. 读取 `SKILL.md` 第 1 段配置，确认仓库绝对路径和默认编辑器。
3. 通过 `git show`、`git ls-tree`、`git grep` 或等价只读 Git 操作读取目标分支；不得切换用户当前分支。
4. 根据业务名聚焦检索，或在未指定业务时从 API/MQ/task 等入口列出重要业务等待选择。
5. 建立证据账本，包含仓库、分支、文件、行号、符号、结论类别。
6. 读取完整纳入范围后，抽取实体、状态、链路、字段生命周期、上下游和失败处理。
7. 按 `references/report-spec.md` 组织详细文字和图表规格。
8. HTML 模式用 `templates/report.html` 生成内联 SVG 报告；Markdown 模式生成 Mermaid。
9. 按 `SKILL.md` 的完成门槛检查，再覆盖写入 skill 内的 `reports/` 目录；不要求用户提供输出路径。

## 编辑器导航

- 证据脚注只呈现 `文件:行号`，不承担跳转功能。
- 代码定位控件只出现在核心方法、方法速查表和排障入口，并由 `editor.uri_template` 生成。
- VS Code 与 Cursor 使用自定义 URI；HTML 在嵌入浏览器中无法保证系统协议放行，因此保留 `fallback_command`。
- JetBrains 不假定存在统一可靠的 URI；未提供用户验证过的 URI 模板时只展示可复制的命令行定位命令。

## 分层检索扩展

新增语言时，优先补充“入口、核心、异步、外部、实体”的等价目录和符号线索，而不是只按文件后缀搜索。

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

## 不可突破的边界

- 不编造业务规则、状态、调用、补偿或依赖。
- 不读取或写入目标仓库之外不必要的敏感文件。
- 不切换、重置、提交或修改用户的 Git 工作区。
- 不因图表排版方便而牺牲证据、可读性或中文文字详细度。
