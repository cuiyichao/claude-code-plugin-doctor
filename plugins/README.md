# Claude Code 插件

本目录包含可扩展 Claude Code 功能的插件集合。这些插件通过自定义命令、专用代理和工作流程来增强 Claude Code 的能力。

## 什么是 Claude Code 插件？

Claude Code 插件是通过自定义斜杠命令、专用代理、钩子和 MCP 服务器来增强 Claude Code 的扩展模块。插件可以在项目和团队之间共享，提供一致的工具和工作流程。

了解更多：[官方插件文档](https://docs.claude.com/en/docs/claude-code/plugins)

## 本目录中的插件

### [project-doctor](./project-doctor/) ⭐ **v2.0 升级**

**项目深度健康诊断插件**

您的私人项目健康顾问。采用架构感知能力，深度分析项目代码，发现编译器和普通静态分析工具无法发现的功能性 Bug。

- **命令**: `/project-doctor` - 对项目进行全面健康检查
- **核心功能**:
  - 📊 **智能架构分析**（v2.0 增强）
    - Monorepo 支持、多语言检测、深度技术栈分析
    - 架构模式推断、模块清单生成
  - 🔍 **按模块深度诊断**（v2.0 重构）
    - 模块级系统化分析（Controller、Service、Model、Utils）
    - 职责合规性检查、代码质量全面检查
    - 依赖关系分析、全链路功能分析
  - 📝 **详细诊断报告生成**（v2.0 增强）
    - 执行摘要、项目全景视图、按模块诊断结果
    - 按功能深度分析、代码质量评估、修复建议与行动计划
- **v2.0 新特性**:
  - 采用 `project-init` 插件的成熟分析引擎
  - 模块级分析取代简单文件级检查
  - 6大章节专业诊断报告
  - 问题分级（Critical/High/Medium/Low）和分类（安全/数据一致性/逻辑错误/性能/架构/代码质量）
- **适用场景**: 接手新项目、代码审查前、测试用例设计

---

### [project-init](./project-init/)

**项目规范初始化插件**

基于内置模板，通过智能分析和交互式问答快速生成定制化的项目开发规范文件。

- **命令**: `/project-init` - 交互式初始化项目 CLAUDE.md 规范文件
- **特性**:
  - ✅ **智能项目分析**：自动检测项目类型、技术栈、架构模式
  - ✅ **Monorepo 支持**：自动识别和处理多项目结构（前后端分离、微服务等）
  - ✅ **按需询问**：只对无法推断的信息进行询问，减少用户输入负担
  - ✅ **内置模板**：模板已嵌入命令中，无需外部文件（支持单项目和 Monorepo）
  - ✅ **交互式配置**：通过问答引导收集项目信息
  - ✅ **智能填充**：自动替换模板占位符
  - ✅ **安全可靠**：覆盖前自动备份现有文件
  - ✅ **跨平台支持**：完全兼容 Windows / macOS / Linux
- **适用场景**: 新项目启动、团队开发规范标准化、项目文档规范化

---

### [session-manager](./session-manager/)

**会话管理插件**

智能保存和恢复 Claude Code 会话，支持进度跟踪和工作连续性。

- **命令**: 
  - `/session-manager:save` - 保存当前会话
  - `/session-manager:continue` - 继续之前的会话
- **核心功能**:
  - 🔄 **会话保存**: 智能总结并保存当前会话的所有关键信息
  - 🔍 **智能搜索**: 支持按项目、时间、关键词搜索已保存的会话
  - 📊 **进度跟踪**: 自动提取 TodoWrite 记录，清晰展示工作进度
  - 💡 **决策记录**: 捕获所有 AskUserQuestion 交互，保留关键决策
  - 📂 **文件追踪**: 记录所有文件变更，便于回顾
  - 💾 **分层存储**: 摘要 + 详细日志分离，兼顾查询效率和数据完整性
  - 🤖 **智能数据访问**: AI 自主决定是否读取完整对话，用户只看摘要
- **适用场景**: 长期项目管理、多项目切换、Token 限制处理、紧急任务插入

## 安装

这些插件已包含在本仓库中。要在您的项目中使用它们：

### 方式一：全局安装

```bash
# 复制所有插件到 Claude Code 全局目录
cp -r plugins/* ~/.claude/plugins/
```

### 方式二：单个插件安装

```bash
# 仅安装特定插件
cp -r plugins/project-init ~/.claude/plugins/
```

### 方式三：项目级配置

在项目的 `.claude/settings.json` 中配置插件路径：

```json
{
  "plugins": {
    "directories": [
      "./plugins"
    ]
  }
}
```

## 使用插件

安装后，在 Claude Code 中使用插件命令：

```bash
# 启动 Claude Code
claude

# 使用插件（以 project-init 为例）
/project-init
```

详细使用方法请查看各插件的 README.md 文档。

## 插件结构

每个插件遵循标准的 Claude Code 插件结构：

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # 插件元数据
├── commands/                 # 斜杠命令（可选）
│   └── command-name.md
├── agents/                   # 专用代理（可选）
├── hooks/                    # 钩子脚本（可选）
└── README.md                # 插件文档
```

## 贡献

欢迎为本目录添加新插件！贡献时请：

1. 遵循标准插件结构
2. 包含完整的 README.md 文档
3. 在 `.claude-plugin/plugin.json` 中添加插件元数据
4. 记录所有命令和代理
5. 提供使用示例

### 插件开发清单

- [ ] 创建标准目录结构
- [ ] 编写 plugin.json 元数据
- [ ] 实现命令或代理逻辑
- [ ] 编写详细的 README.md
- [ ] 添加使用示例和截图
- [ ] 测试所有功能场景
- [ ] 更新本文档的插件列表

## 了解更多

- [Claude Code 文档](https://docs.claude.com/en/docs/claude-code/overview)
- [插件系统文档](https://docs.claude.com/en/docs/claude-code/plugins)
- [官方插件示例](https://github.com/anthropics/claude-code/tree/main/plugins)

---

💡 **提示**: 每个插件的详细功能和使用方法请查看其专属 README.md 文档。
