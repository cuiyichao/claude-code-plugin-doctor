# Claude Code 插件

本目录包含可扩展 Claude Code 功能的插件集合。这些插件通过自定义命令、专用代理和工作流程来增强 Claude Code 的能力。

## 什么是 Claude Code 插件？

Claude Code 插件是通过自定义斜杠命令、专用代理、钩子和 MCP 服务器来增强 Claude Code 的扩展模块。插件可以在项目和团队之间共享，提供一致的工具和工作流程。

了解更多：[官方插件文档](https://docs.claude.com/en/docs/claude-code/plugins)

## 本目录中的插件

### [project-init](./project-init/)

**项目规范初始化插件**

基于 CLAUDE_TEMPLATE.md 模板，通过交互式问答快速生成定制化的项目开发规范文件。

- **命令**: `/project-init` - 交互式初始化项目 CLAUDE.md 规范文件
- **特性**:
  - 9轮渐进式问答收集项目信息
  - 支持多种技术栈（Go、Python、TypeScript、Java）
  - 自动备份现有配置，避免意外覆盖
  - 灵活的占位符替换机制
  - 详细的选项说明和引导
- **适用场景**: 新项目启动、团队开发规范标准化、项目文档规范化

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
