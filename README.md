# Claude Code 插件市场

<div align="center">

**扩展 Claude Code 功能的社区插件集合**

[![Plugins](https://img.shields.io/badge/plugins-1-blue.svg)](./plugins)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-purple.svg)](https://claude.ai/code)

[插件列表](#插件列表) • [安装方法](#安装方法) • [开发指南](#开发插件) • [贡献](#贡献)

</div>

---

## 📦 什么是 Claude Code 插件？

Claude Code 插件是通过自定义斜杠命令（Slash Commands）、专用代理（Agents）、钩子（Hooks）和 MCP 服务器来扩展 Claude Code 功能的模块。插件可以在项目和团队之间共享，提供一致的工具和工作流程。

### 插件能做什么？

- ✅ **自动化工作流**：简化重复性任务
- ✅ **代码生成**：快速生成标准化代码和配置
- ✅ **质量保证**：自动化代码审查和测试
- ✅ **项目管理**：辅助项目初始化和规范管理
- ✅ **Git 集成**：优化版本控制流程

## 🎯 插件列表

### [project-init](./plugins/project-init/)

**项目规范初始化插件**

基于 CLAUDE_TEMPLATE.md 模板，通过交互式问答快速生成定制化的项目开发规范文件。

- **命令**: `/project-init` - 交互式初始化项目 CLAUDE.md 规范
- **特性**:
  - 9轮渐进式问答收集项目信息
  - 支持多种技术栈（Go/Python/TypeScript/Java）
  - 自动备份现有配置
  - 智能占位符替换
- **适用场景**: 新项目启动、团队规范标准化、开发流程规范化

## 📥 安装方法

### 方式一：全局安装（推荐）

```bash
# 克隆插件市场仓库
git clone <your-plugin-marketplace-repo>
cd claude-code-plugins

# 复制插件到 Claude Code 全局目录
cp -r plugins/* ~/.claude/plugins/
```

### 方式二：项目级安装

```bash
# 在项目目录下
cd /path/to/your/project

# 克隆或下载插件
git clone <your-plugin-marketplace-repo> .claude-plugins

# 在 .claude/settings.json 中配置插件路径
```

### 方式三：单个插件安装

```bash
# 仅安装特定插件
cp -r plugins/project-init ~/.claude/plugins/
```

## 🚀 使用插件

安装后，在 Claude Code 中即可使用插件命令：

```bash
# 启动 Claude Code
claude

# 使用插件命令（以 project-init 为例）
/project-init
```

## 📖 插件文档

每个插件都包含详细的 README.md 文档，包括：

- 功能介绍和特性说明
- 详细的使用教程
- 配置选项说明
- 常见问题解答
- 使用场景示例

查看具体插件文档：[plugins/](./plugins/)

## 🛠️ 开发插件

### 插件标准结构

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # 插件元数据（必需）
├── commands/                 # 斜杠命令（可选）
│   └── command-name.md
├── agents/                   # 专用代理（可选）
├── hooks/                    # 钩子脚本（可选）
└── README.md                # 插件文档（必需）
```

### plugin.json 配置示例

```json
{
  "name": "my-plugin",
  "description": "插件功能描述",
  "version": "1.0.0",
  "author": {
    "name": "Your Name",
    "email": "your.email@example.com"
  }
}
```

### 命令文件格式

```markdown
---
allowed-tools: Read, Write, Bash
description: 命令描述
---

命令执行逻辑...
```

### 开发步骤

1. **创建插件目录结构**
   ```bash
   mkdir -p my-plugin/.claude-plugin
   mkdir -p my-plugin/commands
   ```

2. **编写 plugin.json**
   - 定义插件元数据
   - 指定版本和作者信息

3. **实现命令逻辑**
   - 在 `commands/` 目录创建 `.md` 文件
   - 定义 `allowed-tools` 和执行步骤

4. **编写文档**
   - 创建 README.md
   - 包含使用示例和配置说明

5. **测试插件**
   - 复制到 `~/.claude/plugins/`
   - 在 Claude Code 中测试命令

## 📚 参考资源

### 官方文档

- [Claude Code 官方文档](https://docs.claude.com/en/docs/claude-code/overview)
- [插件系统文档](https://docs.claude.com/en/docs/claude-code/plugins)
- [官方插件示例](https://github.com/anthropics/claude-code/tree/main/plugins)

### 最佳实践

- **遵循 KISS 原则**：保持插件简洁明了
- **提供详细文档**：让用户快速上手
- **异常处理完善**：优雅处理边界情况
- **用户体验优先**：提供清晰的交互和反馈
- **版本语义化**：使用 semver 管理版本

## 🤝 贡献

我们欢迎社区贡献新插件或改进现有插件！

### 贡献流程

1. **Fork 本仓库**

2. **创建插件分支**
   ```bash
   git checkout -b plugin/your-plugin-name
   ```

3. **开发插件**
   - 在 `plugins/` 目录下创建新插件
   - 遵循标准插件结构
   - 编写完整的 README.md

4. **测试验证**
   - 在本地测试插件功能
   - 确保所有场景正常工作

5. **提交 Pull Request**
   - 清晰描述插件功能
   - 提供使用示例截图
   - 说明测试情况

### 贡献指南

- ✅ 插件应解决实际问题
- ✅ 代码质量和文档完善
- ✅ 遵循现有插件的风格
- ✅ 提供充分的测试和示例
- ❌ 避免重复造轮子
- ❌ 不引入不必要的依赖

## 📋 插件清单

| 插件名称 | 版本 | 描述 | 作者 |
|---------|------|------|------|
| [project-init](./plugins/project-init/) | v1.0.0 | 项目规范初始化 | Wang Xuecheng |

_更多插件持续添加中..._

## 🎨 插件分类

### 🚀 项目管理
- [project-init](./plugins/project-init/) - 项目规范初始化

### 🔧 开发工具
_即将推出..._

### 📝 代码生成
_即将推出..._

### ✅ 质量保证
_即将推出..._

### 🔄 Git 工作流
_即将推出..._

## 💡 插件想法

欢迎提出新插件想法！以下是一些潜在方向：

- **测试生成器**：自动生成单元测试
- **文档生成器**：从代码生成 API 文档
- **代码审查助手**：自动化 PR 审查
- **数据库迁移**：数据库变更管理
- **性能分析**：代码性能优化建议
- **依赖更新**：自动更新依赖版本

[提交想法 Issue →](../../issues/new)

## 📜 许可证

本项目采用 MIT 许可证。详见 [LICENSE](./LICENSE) 文件。

## 🔗 相关链接

- [Claude Code 官网](https://claude.ai/code)
- [Claude Code GitHub](https://github.com/anthropics/claude-code)
- [Claude API 文档](https://docs.anthropic.com/)
- [社区论坛](https://community.anthropic.com/)

## 📞 联系我们

- **Issues**: [提交问题](../../issues)
- **Discussions**: [参与讨论](../../discussions)
- **Email**: wangxuecheng@example.com

---

<div align="center">

**🌟 如果这个项目对你有帮助，请给我们一个 Star！**

Made with ❤️ by Claude Code Community

</div>
