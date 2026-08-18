# codebase-storyteller

一个面向后端开发者的 Codex Agent Skill。给出本地仓库名和可选业务名后，它从指定 Git 分支读取真实代码，生成可离线打开的中文业务梳理报告。

报告以业务为标题，仓库只作为分析范围。内容包括系统总览、实体关系、状态机、主链路/取消/补偿时序图、字段生命周期、模块深挖、方法速查、上下游、设计意图和排障清单。

## 适用场景

- 新接手后端模块，需要理解业务主干和状态流转。
- 需要把 Application、Domain、MQ、DB、外部服务串成完整故事。
- 需要在报告中保留真实代码定位，继续排障或修改。
- 一个业务涉及多个仓库，需要按业务而不是按仓库组织说明。

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

打开 `SKILL.md` 的第 1 段“仓库配置”，填写本地仓库根目录和默认编辑器：

```yaml
repository_root: "/workspace"
repository_paths: {}
target_branch: release
editor:
  kind: vscode
  uri_template: "vscode://file/{path}:{line}:1"
  fallback_command: "code --goto '{path}:{line}:1'"
```

仓库不在统一根目录时，可以添加映射：

```yaml
repository_root: "/workspace"
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

首次调用时未配置仓库根目录或编辑器，skill 会停下来询问并将答案写回配置。

## 调用方式

### 单仓库 HTML 报告

```text
$codebase-storyteller
仓库：order-service
业务：库存出库
```

默认读取配置中的 `release` 分支，通过 Git branch tree 获取内容，不切换或改动当前工作区。报告写入：

```text
~/.codex/skills/codebase-storyteller/reports/库存出库--order-service.html
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
业务：退款
格式：md
分支：main
```

Markdown 保留同等详细度和所有章节；图表使用 Mermaid 代码块，适合 GitHub、Obsidian 和编辑器预览。

### 不提供业务名

```text
$codebase-storyteller
仓库：order-service
```

skill 会先从 API、MQ consumer、定时任务和模块边界识别核心业务，列出候选项并等待选择，不会擅自生成某个业务报告。

## 输出与代码导航

- HTML 完全自包含，不引用 CDN、远程图表服务、远程字体或图片。
- 图表以预渲染内联 SVG 保存，支持侧边栏目录和点击放大。
- 代码定位只通过独立“打开代码 ↗”按钮提供；普通“证据”脚注只是灰色来源说明。
- VS Code/Cursor 使用 URI 按钮。部分内嵌浏览器可能拦截自定义 URI，此时在系统浏览器打开报告，或使用报告提供的回退命令。
- JetBrains 未配置已验证 URI 模板时，skill 输出可复制的命令行定位命令，不伪造不可用链接。

## 隐私与发布

`reports/` 是本地生成物目录，默认被 `.gitignore` 忽略。生成的报告通常包含绝对路径、方法名、源码片段和内部业务信息，不应提交到公开仓库。

发布前检查：

1. 保持 `repository_root` 为空，不提交本机路径或仓库映射。
2. 不提交 `reports/` 内的真实业务报告。
3. 选择并添加适合你的开源许可证。
4. 检查 README、示例、截图和 Git 历史中没有内部仓库名、域名、客户信息或代码。

## 设计原则

- 只报告目标分支真实代码可证明的内容。
- 结论区分事实、代码推断和证据不足。
- 图表展示结构，文字解释顺序、字段、失败处理和设计意图。
- 报告以业务为中心，支持多仓库参与方标注。
