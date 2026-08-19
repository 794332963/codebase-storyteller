# codebase-storyteller

一个面向后端开发者的 Codex Agent Skill。它从指定 Git ref 或当前工作区读取真实代码，生成可离线打开的中文报告，支持业务视角与架构（源码/框架）视角。

业务视角以业务为标题，讲解实体、状态、主链路、补偿与排障。架构视角以核心子系统、主干流程或抽象为标题，讲解入口、类型关系、调用主干、数据结构、设计模式、扩展点与阅读路线。

## 适用场景

- 新接手后端模块，需要理解业务主干和状态流转。
- 需要把 Application、Domain、MQ、DB、外部服务串成完整故事。
- 需要在报告中保留真实代码定位，继续排障或修改。
- 一个业务涉及多个仓库，需要按业务而不是按仓库组织说明。
- 阅读 ORM、SDK、框架库或基础组件，需要理解核心抽象、注册机制和扩展方式。

## 安装

```bash
cp -R codebase-storyteller ~/.codex/skills/
```

目录结构：

```text
codebase-storyteller/
├── SKILL.md
├── README.md
├── AGENTS.md
├── references/
│   ├── report-spec.md
│   └── visual-style.md
├── reports/
│   └── .gitkeep
└── templates/
    └── report.html
```

## 首次配置

首次运行时，skill 会在根目录创建 `config.local.yaml`。这个文件保存本机仓库路径和编辑器配置，已被 `.gitignore` 忽略，不会随公开仓库发布。内容如下：

```yaml
repository_root: "/workspace"
repository_roots:
  - "/workspace"
repository_paths: {}
target_branch: auto
code_source: ref # 可选：ref | worktree；未配置时默认 ref
editor:
  kind: vscode
  uri_template: "vscode://file/{path}:{line}:1"
  fallback_command: "code --goto '{path}:{line}:1'"
```

仓库分布在多个根目录时，直接增加 `repository_roots`：

```yaml
repository_roots:
  - "/Users/name/Projects"
  - "/Volumes/work/repos"
repository_paths: {}
```

仓库不在常规根目录、或不同根目录存在同名仓库时，添加精确映射：

```yaml
repository_roots:
  - "/workspace"
  - "/Volumes/work/repos"
repository_root: "" # 旧版兼容字段
repository_paths:
  order-service: "/Users/name/Projects/order-service"
  warehouse-service: "/Users/name/Projects/warehouse-service"
target_branch: release
editor:
  kind: cursor
  uri_template: "cursor://file/{path}:{line}:1"
  fallback_command: "cursor --goto '{path}:{line}:1'"
```

JetBrains IDE 示例：

```yaml
editor:
  kind: jetbrains
  uri_template: ""
  fallback_command: "idea --line '{line}' --column 1 '{path}'"
```

首次调用时未配置仓库根目录或编辑器，skill 会停下来询问并将答案写回 `config.local.yaml`。后续调用直接复用，不会重复询问。

## 代码来源

- `ref`（默认）：按 `target_branch` 或调用中的 `分支`，只读 Git ref 内容，不读取当前工作区未提交修改。
- `worktree`：直接读取当前 checkout 的磁盘文件，不切换或修改工作区；报告的行号与当前工作区一致。
- 未显式指定来源且仓库当前 HEAD 与计划读取的 ref 不一致时，skill 会先询问选择“当前工作区”或“配置的 ref”。多仓库时仍按各自实际版本标注。

## 调用方式

### 单仓库 HTML 报告

```text
$codebase-storyteller
仓库：order-service
视角：业务
业务：库存出库
```

默认读取每个目标仓库的默认分支（例如 `main`、`master` 或 `release`），通过 Git branch tree 获取内容，不切换或改动当前工作区。需要固定分支时，在调用中填写 `分支：release` 或在 `config.local.yaml` 中设置 `target_branch`。报告写入：

```text
~/.codex/skills/codebase-storyteller/reports/库存出库--order-service.html
```

### 显式读取当前工作区

```text
$codebase-storyteller
仓库：order-service, warehouse-service
视角：业务
业务：跨仓出库
来源：工作区
```

此模式直接读取每个仓库各自当前 checkout 的文件；页首会分别展示实际 checkout 标识并说明行号与当前工作区一致。

### 显式读取指定 ref

```text
$codebase-storyteller
仓库：order-service
视角：业务
业务：库存出库
来源：指定分支
分支：release
```

### 多仓库 HTML 报告

```text
$codebase-storyteller
仓库：order-service, warehouse-service
业务：跨仓出库
```

报告标题是“跨仓出库业务梳理”，输出文件名为：

```text
跨仓出库--order-service__warehouse-service.html
```

### Markdown 报告

```text
$codebase-storyteller
仓库：payment-service
视角：业务
业务：退款
格式：md
分支：main
```

Markdown 保留同等详细度和所有章节；图表使用 Mermaid 代码块，适合 GitHub、Obsidian 和编辑器预览。

### 业务视角不提供业务名

```text
$codebase-storyteller
仓库：order-service
视角：业务
```

skill 会先从 API、MQ consumer、定时任务和模块边界识别核心业务，列出候选项并等待选择，不会擅自生成某个业务报告。

### 架构视角

```text
$codebase-storyteller
仓库：orm-library
视角：架构
目标：查询执行主干
```

架构视角从真实入口 API、核心类型、实现关系、装配/注册点和调用图组织报告，不会套用业务实体或状态机章节。

### 未指定视角或架构目标

```text
$codebase-storyteller
仓库：orm-library
```

未指定 `视角` 时，skill 只询问选择“业务视角”还是“架构视角”。选择架构视角但未提供 `目标` 时，skill 会列出真实的核心子系统、主干流程或核心抽象并等待选择；不会猜测目标。

### 要求

```text
$codebase-storyteller
仓库：order-service, warehouse-service
视角：业务
业务：跨仓出库
要求：补充一节两个实现的差异对比，优先解释重试边界
```

`要求` 是受约束的补充入口，可调整语言、讲解侧重、章节内顺序、既有图类型取舍或补充小节。它不能关闭证据、删减固定章节/图、要求脱离代码作答，或覆盖离线模板、deep link 与多仓库归属规则；冲突部分会被忽略并在完成提示中说明。

## 输出与代码导航

- HTML 完全自包含，不引用 CDN、远程图表服务、远程字体或图片。
- 图表以预渲染内联 SVG 保存，支持侧边栏目录和点击放大。
- 类型关系图在代码可证明时使用 UML 泛化、实现、组合、聚合、依赖与关联记法；不会因类型名或目录猜测关系。
- 多仓库报告逐一标注仓库与实际版本/ref；跨仓库调用、实现、注册、依赖或改造关系仅在真实代码证明时绘制和解释。
- 方法名或明确对象名本身是 `.code-open` 编辑器 deep link；普通“证据”脚注默认不显示。
- “设计亮点”是可选且克制的 `.highlight` callout，仅用于有 `file:line` 证据的设计取舍说明，不是固定内容。
- VS Code/Cursor deep link 带 `target="_blank"`。部分内嵌浏览器可能拦截自定义 URI，此时在系统浏览器打开报告。
- JetBrains 仅在用户配置已验证 URI 模板时生成可点击链接，不伪造不可用链接。

## 隐私与发布

`reports/` 是本地生成物目录，默认被 `.gitignore` 忽略。生成的报告通常包含绝对路径、方法名、源码片段和内部业务信息，不应提交到公开仓库。

发布前检查：

1. 不提交 `config.local.yaml`，不要把本机路径或仓库映射写入公开版 `SKILL.md`。
2. 不提交 `reports/` 内的真实业务报告。
3. 选择并添加适合你的开源许可证。
4. 检查 README、示例、截图和 Git 历史中没有内部仓库名、域名、客户信息或代码。
5. 确认公开仓库不含任何真实 `reports/` 下的 `.html`/`.md`，也不含 `config.local.yaml`。

## 设计原则

- 只报告目标分支真实代码可证明的内容。
- 结论区分事实、代码推断和证据不足。
- 图表展示结构，文字解释顺序、字段、失败处理和设计意图。
- 两种视角共用真实代码证据、静态 SVG、离线 HTML 与代码定位规则；业务视角的 14 章节质量基线不会因架构视角而降低。
- 代码来源、UML 记法和设计亮点均为通用能力，不预设某种语言、架构、模式或项目必须具备。
- 多仓库归属与 `要求` 同样是通用默认能力：前者不混淆底座/扩展/改造版本，后者不能成为绕过真实性或质量门槛的后门。
