---
name: codebase-storyteller
description: 仅在用户显式输入 `$codebase-storyteller` 时使用。用户只需给出一个或多个本地代码仓库名称和可选业务名称；本 skill 会读取指定分支的真实代码，并自动将详细、中文、图文并茂的业务梳理报告保存到 skill 的 reports 目录，帮助后端开发者快速理解实体、状态、主链路、异步补偿、字段流转、上下游和排障入口。
metadata:
  version: "2.1.0"
  output: "offline self-contained HTML by default, Markdown on request"
---

# Codebase Storyteller

仅在用户显式输入 `$codebase-storyteller` 时执行。一次只生成一个业务报告；不要因“业务梳理”“代码说明”等普通请求自动激活。

## 1. 仓库配置

本节是持久化配置。首次使用前填写 `repository_root`，或在 `repository_paths` 中为仓库设置绝对路径。用户可以直接修改这一段。

```yaml
repository_root: "" # 例如 /workspace；定位规则为 {repository_root}/{repository_name}
repository_paths: {} # 例如 { order-service: /Users/me/Projects/order-service }
target_branch: release
editor:
  kind: vscode # vscode | cursor | jetbrains
  uri_template: "vscode://file/{path}:{line}:1"
  fallback_command: "code --goto '{path}:{line}:1'"
```

配置规则：

- 每次运行先读取并检查此配置。
- `repository_paths[仓库名]` 优先于 `repository_root/仓库名`。
- `repository_root` 为空且目标仓库没有映射时，停止并只询问：`你的代码仓库都放在哪个本地路径下？`
- 得到用户路径后，写回本节的 `repository_root`，再继续执行；配置缺失时不得扫描仓库或生成报告。
- 默认目标分支是 `release`。用户可修改 `target_branch`；单次调用明确指定分支时，以用户指定为准。
- `editor.kind` 是必配项。首次配置仓库后若未配置编辑器，停止并询问：`你默认使用哪个代码编辑器：VS Code、Cursor，还是 JetBrains IDE（IntelliJ IDEA、GoLand 等）？`
- 在 macOS 上优先用 `open -Ra` 检查用户选择的应用是否已安装，再写回配置；检测到多个编辑器时仍以用户选择为准。
- `vscode` 使用 `vscode://file/{path}:{line}:1`，`cursor` 使用 `cursor://file/{path}:{line}:1`。
- `jetbrains` 默认使用命令行启动器作为回退：`idea --line {line} --column 1 {path}`；只有用户提供已验证的 JetBrains URL 模板时，才生成可点击 URI。

## 2. 资源与加载顺序

主流程只保留执行控制。开始生成前按需读取以下资源：

| 资源 | 何时读取 | 用途 |
|---|---|---|
| [references/report-spec.md](references/report-spec.md) | 已完成代码定位后 | 报告章节、文字详细度、证据与完整性标准 |
| [references/visual-style.md](references/visual-style.md) | HTML 输出时 | 离线内嵌 SVG、布局、交互、无障碍和视觉规则 |
| [templates/report.html](templates/report.html) | HTML 输出时 | 直接替换占位符并填充报告内容 |
| [AGENTS.md](AGENTS.md) | 需扩展语言或检索规则时 | 设计边界与扩展方式 |

不要把这些资源的全文复制回 `SKILL.md`，也不要跳过它们。

## 3. 输入约定

用户以如下形式调用：

```text
$codebase-storyteller
仓库：repo-a, repo-b
业务：可选
格式：html 或 md，可选，默认为 html
分支：可选，默认使用配置的 target_branch
```

输入约束：

- `仓库` 必填，可多个。
- `业务` 可选。未提供时，必须先列出核心/重要业务并等待用户选择；不得擅自挑一个业务生成。
- `格式` 只有 `html` 和 `md`；默认 `html`。
- 不向用户询问输出路径。始终写入本 skill 的 `reports/` 目录。
- 文件名使用 `{业务名}--{仓库名}.{html|md}`；多仓库按输入顺序用 `__` 连接仓库名。将空格和路径分隔符替换为 `-`，移除其他不安全文件名字符。
- 同名文件直接覆盖，不加 `v1`、日期或其他后缀。

## 4. 执行流程

### Step 0: 配置检查

1. 读取第 1 节配置。
2. 根据映射或根目录解析每个仓库的绝对路径。
3. 验证目录存在且是 Git 仓库。路径无效时列出无效仓库和解析路径，停止等待用户修正。
4. 仓库或编辑器配置缺失时按第 1 节提问并持久化配置，不进行后续动作。

### Step 1: 目标分支快照

只通过 Git 读取目标分支内容，绝不执行 `git checkout`、`git switch`、`git reset` 或改动用户工作区。

1. 依次确认 `refs/heads/<target_branch>`、`refs/remotes/*/<target_branch>` 是否存在。
2. 若存在，记录完整 ref 与 commit SHA，并使用该 ref 的 tree 和 blob 进行后续搜索、读取和行号计算。
3. 若不存在，识别默认分支（优先 `refs/remotes/origin/HEAD`，其次本地 `HEAD` 的分支）；记录回退原因。
4. 报告页首只展示业务标题、涉及仓库、分析分支和生成日期；不要展示 commit SHA 或纳入文件数。
5. 报告页首必须写明：`本次基于 <branch> 分支`。
6. 报告页首必须写明：`代码定位行号基于 <branch> 分支；阅读时建议将工作区切到该分支。`

不要把当前工作区未提交改动当作证据。

### Step 2: 自动定位业务代码

用户不需要提供文件路径。对每个仓库在目标 branch tree 中分层检索：

- 入口层：`application`、`api`、`controller`、`handler`、`router`、`cmd`。
- 核心层：`domain`、`service`、`usecase`、`biz`、`core`。
- 异步层：`task`、`job`、`consumer`、`listener`、`worker`、`scheduler`。
- 外部依赖层：`client`、`gateway`、`adapter`、`infra`、`rpc`、`repository`。
- 实体层：`model`、`entity`、`aggregate`、`po`、`dao`、`schema`、`dto`。

自动识别主要语言和项目布局，至少覆盖 Go、Java/Kotlin、TypeScript/JavaScript、Python、Rust、SQL；语言特有检索规则见 `AGENTS.md`。

给了业务名称时：

1. 用业务名、模块名、路由、消费者/任务名、实体名和相邻调用图检索。
2. 沿调用方向扩展到入口、领域、存储、异步与外部依赖。
3. 只纳入能够直接支撑该业务主干、取消或修复链路的文件。

未给业务名称时：

1. 扫描对外路由、RPC/HTTP handler、MQ consumer、定时任务和模块包名。
2. 按入口数量、关联核心代码规模和模块边界识别并排序“核心/重要”业务；忽略纯工具、配置和琐碎 CRUD。
3. 每项给出业务名、所属仓库、主要入口和一句话说明。
4. 等待用户选择一个/多个业务或输入“全部生成”。收到选择后再继续。

完成后打印：

`已定位 X 个相关文件，开始阅读并分析...`

### Step 3: 结构化读取与证据建模

先读完全部相关文件，再形成结论或输出报告。维护一个内部证据账本，每个事实都至少记录 `仓库 / branch / file / line / symbol`。

提取以下内容，均以真实代码为准：

1. **实体**：核心 struct/class/model/entity、字段、类型、关联字段、状态字段、批次/追踪/幂等字段。
2. **状态机**：全部枚举或常量状态、每个状态写入点、`from -> to`、触发方法、前置校验、终态。
3. **流程**：创建、绑定、出库、取消、同步、重试、修复、补偿等实际存在的链路；记录调用顺序、关键参数、DB 写入、外部调用、MQ/任务及失败分支。
4. **上下游**：上游触发方、下游依赖、仓库归属、同步/异步边界。
5. **字段生命周期**：首次写入、每次更新、更新条件、复制/清空逻辑和终态值。

多仓库调用必须标注参与方所属仓库。代码没有证明的关系、状态或意图不得绘制或陈述为事实。

### Step 4: 生成报告

1. 读取 `references/report-spec.md`。
2. HTML 模式额外读取 `references/visual-style.md` 与 `templates/report.html`。
3. 先完成完整性自检，再一次性写出报告；不要边读边输出报告正文。
4. Markdown 模式使用 Mermaid 代码块、普通代码块和锚点目录，但必须保留同等详细度与全部章节。
5. HTML 模式基于模板生成完全自包含、可离线双击打开的单文件报告。所有图必须是报告内的内联 SVG；禁止 CDN、PlantUML URL、Mermaid URL、外部字体、外部图片和运行时网络请求。
6. 自动创建 `reports/` 目录（若不存在），并按第 3 节的固定命名规则覆盖写入。
7. 报告标题、侧边栏品牌和文件名以业务名称为主；仓库名只作为范围、分支和文件名后的辅助信息出现。不要把单个仓库名写成业务标题。
8. HTML 中的图下注释“证据”只显示灰色 `文件:行号` 来源文本，不作为链接。每个核心方法、排障入口和方法速查表使用单独的紧凑“打开代码”按钮生成 editor URI；按钮使用 `target="_blank"`。
9. 若编辑器只有 `fallback_command` 而没有可靠 URI，HTML 显示复制命令按钮，不把不能执行的命令伪装成跳转链接。

## 5. 准确性与完成门槛

在写出文件前逐项检查：

- 所有结论、节点、方法、状态、外部调用和字段变化均可回溯到目标 branch 的真实 `file:line`。
- 每个核心实体都有字段说明、关联关系；每个有状态实体都有独立状态机。
- 每条实际存在的主链路、取消链路、补偿/修复链路均已绘图并逐段拆解；缺失的链路要明确写“代码范围内未发现”。
- 流程图中出现的核心方法，都在模块深挖或方法速查表中解释。
- 每个核心方法都能回答职责、执行顺序、字段变化、失败处理、设计意图五件事；证据不足时如实标注而非补写。
- HTML 不存在外部 `http(s)` 资源、暗色主题或空白/损坏图占位符。
- 每个核心方法和排障入口都有按当前 `editor` 配置生成的独立代码定位按钮；行号来自目标 branch。无法可靠定位时，链接到该文件第 1 行并标注 `定位近似`。
- 不将普通“证据”文本做成蓝色跳转链接。报告内嵌浏览器若拦截自定义 URI，按钮仍保留目标 URI，用户可在系统浏览器打开报告或使用显示的回退命令。
- 输出文件存在、非空，且位于本 skill 的 `reports/` 目录并符合固定命名规则。

完成后只简要告诉用户：输出路径、分析分支、纳入文件数，以及配置位于 `SKILL.md` 第 1 段。证据不足只能在对应结论旁标注，不单独组成报告章节。
