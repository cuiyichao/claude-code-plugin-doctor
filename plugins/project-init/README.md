# Project-Init Plugin

快速初始化项目 CLAUDE.md 规范文件的 Claude Code 插件。

## 功能介绍

基于 `CLAUDE_TEMPLATE.md` 模板，通过交互式问答自动生成定制化的项目开发规范文件。

### 核心特性

- ✅ **交互式配置**：通过问答引导收集项目信息
- ✅ **智能填充**：自动替换模板占位符
- ✅ **安全可靠**：覆盖前自动备份现有文件
- ✅ **灵活定制**：支持多种技术栈和架构模式
- ✅ **开箱即用**：无需额外依赖

## 安装方法

### 方式一：直接使用（推荐）

将此 plugin 目录放置在您的项目中或 Claude Code 的 plugins 目录下。

```bash
# 复制到项目目录
cp -r project-init /path/to/your/project/

# 或者复制到全局 plugins 目录
cp -r project-init ~/.claude/plugins/
```

### 方式二：Git 引用

```bash
cd ~/.claude/plugins/
git clone <plugin-repository-url> project-init
```

## 使用方法

### 快速开始

1. 确保项目中存在 `CLAUDE_TEMPLATE.md` 模板文件
2. 在 Claude Code 中执行命令：

```bash
/project-init
```

3. 按照提示回答问题
4. 自动生成 `CLAUDE.md` 文件

### 执行流程

```
启动命令
  ↓
检查现有 CLAUDE.md（如有则询问是否覆盖）
  ↓
读取 CLAUDE_TEMPLATE.md 模板
  ↓
收集项目信息（4轮交互式问答）
  ├─ 项目核心信息（名称、描述）
  ├─ 技术栈信息（语言、框架、数据库）
  ├─ 架构配置（架构模式、代码风格）
  └─ 开发规则（测试要求、Git 流程）
  ↓
生成并写入 CLAUDE.md
  ↓
显示成功信息
```

### 问答内容

#### 第一批：项目核心信息
- **项目名称**：输入项目名称或使用当前目录名
- **项目描述**：一句话描述项目用途

#### 第二批：技术栈信息
- **开发语言**：Go / Python / TypeScript / Java 等
- **主要框架**：根据语言选择对应框架
- **数据存储**：MySQL / Redis / MongoDB 等（可多选）

#### 第三批：架构配置
- **项目架构**：分层架构 / 微服务 / Serverless 等
- **代码风格**：camelCase / PascalCase / snake_case 等

#### 第四批：开发规则
- **测试覆盖率**：是否要求测试覆盖率及具体比例
- **Git 提交流程**：是否需要提交前确认

## 示例

### 使用场景 1：新项目初始化

```bash
# 在新项目目录下
cd /path/to/new-project

# 确保有模板文件
ls CLAUDE_TEMPLATE.md

# 执行初始化
/project-init
```

**交互示例**：
```
Q: 请输入项目名称
A: MyAwesomeProject

Q: 请用一句话描述项目
A: 高性能 Web API 服务

Q: 项目主要使用什么开发语言？
A: Go 1.21+

Q: 项目使用什么主要框架？
A: Gin

Q: 项目使用什么数据存储技术？
A: MySQL, Redis

Q: 项目采用什么架构模式？
A: 分层架构（MVC/三层架构）

Q: 项目采用什么命名和代码风格规范？
A: 遵循语言官方规范

Q: 是否启用测试覆盖率要求？
A: 启用（要求 >80%）

Q: Git 提交是否需要审批确认？
A: 需要（使用 AskUserQuestion）
```

**生成结果**：
```
✅ 项目规范文件生成成功！

📄 文件位置: ./CLAUDE.md
📅 生成时间: 2025-11-03 14:30:00

📋 配置摘要:
- 项目名称: MyAwesomeProject
- 开发语言: Go 1.21+
- 主要框架: Gin
- 架构模式: 分层架构（MVC/三层架构）
```

### 使用场景 2：覆盖现有配置

如果项目中已存在 `CLAUDE.md`，插件会询问：

```
⚠️ 检测到已存在 CLAUDE.md 文件

请选择操作：
1. 备份后覆盖（推荐） - 保存为 CLAUDE.md.backup.20251103143000
2. 直接覆盖 - 原文件将被替换
3. 取消操作 - 退出命令
```

## 配置说明

### plugin.json

```json
{
  "name": "project-init",
  "description": "快速初始化项目 CLAUDE.md 规范文件",
  "version": "1.0.0",
  "author": {
    "name": "Wang Xuecheng",
    "email": "wangxuecheng@example.com"
  }
}
```

### allowed-tools

命令执行时使用以下工具：

- `Read`: 读取模板文件
- `Write`: 生成 CLAUDE.md 文件
- `AskUserQuestion`: 交互式收集用户输入
- `Bash(ls:*)`: 检查文件是否存在
- `Bash(test:*)`: 文件存在性检测
- `Bash(cp:*)`: 备份现有文件
- `Bash(date:*)`: 获取时间戳

## 目录结构

```
project-init/
├── .claude-plugin/
│   └── plugin.json          # Plugin 元数据配置
├── commands/
│   └── project-init.md      # 命令实现逻辑
├── docs/
│   └── todo/
│       └── 2025-11-03-14-07-project-init-plugin-方案.md
└── README.md                # 本文档
```

## 依赖要求

- Claude Code CLI（最新版本）
- CLAUDE_TEMPLATE.md 模板文件（需放在项目根目录）

## 常见问题

### Q1: 找不到 CLAUDE_TEMPLATE.md 怎么办？

**A:** 确保模板文件存在于以下位置之一：
- 项目根目录
- ~/.claude/CLAUDE_TEMPLATE.md

或者从本项目复制模板：
```bash
cp CLAUDE_TEMPLATE.md /path/to/your/project/
```

### Q2: 生成的 CLAUDE.md 可以手动修改吗？

**A:** 完全可以！生成的文件只是起点，您应该根据项目实际情况调整细节。

### Q3: 如何恢复备份的文件？

**A:** 如果选择了"备份后覆盖"，原文件会保存为：
```bash
CLAUDE.md.backup.{timestamp}

# 恢复方法
cp CLAUDE.md.backup.20251103143000 CLAUDE.md
```

### Q4: 支持哪些编程语言？

**A:** 目前内置支持：
- Go
- Python
- TypeScript
- Java

其他语言可以通过"Other"选项自定义输入。

### Q5: 可以跳过某些问题吗？

**A:** 可以！所有非必填字段都可以使用默认值或留空。

## 高级用法

### 自定义模板

您可以修改 `CLAUDE_TEMPLATE.md` 来定制模板内容：

1. 添加新的占位符（使用 `[占位符名称]` 格式）
2. 在 `commands/project-init.md` 中添加对应的问题
3. 在字符串替换逻辑中添加映射关系

### 批量初始化

对于多个项目，可以准备标准配置文件，然后批量执行：

```bash
# 方案：编写脚本自动回答问题
# 或者：使用默认配置快速生成
```

## 贡献指南

欢迎提交 Issue 和 Pull Request！

### 开发规范

- 遵循 KISS 原则
- 保持代码简洁
- 添加详细注释
- 测试所有场景

### 测试建议

测试以下场景：
- ✅ 新项目初始化
- ✅ 覆盖现有文件
- ✅ 模板文件不存在
- ✅ 权限不足
- ✅ 用户取消操作
- ✅ 不同技术栈组合

## 版本历史

### v1.0.0 (2025-11-03)
- 首次发布
- 支持基础项目初始化
- 交互式问答流程
- 自动备份机制

## 许可证

MIT License

## 联系方式

- 作者：Wang Xuecheng
- Email: wangxuecheng@example.com

---

🤖 Generated with Claude Code
