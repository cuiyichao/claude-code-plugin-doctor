---
allowed-tools: Read, Write, AskUserQuestion, Bash(ls:*), Bash(test:*), Bash(cp:*), Bash(date:*)
description: 基于 CLAUDE_TEMPLATE.md 快速初始化项目 CLAUDE.md 规范文件
---

基于 CLAUDE_TEMPLATE.md 模板，通过交互式问答生成项目的 CLAUDE.md 开发规范文件。

执行步骤：

## 1. 前置检查

首先执行以下检查：

a. 检查当前目录是否已存在 CLAUDE.md 文件
   - 如果存在，使用 AskUserQuestion 询问用户是否要覆盖：
     - 选项 1: "备份后覆盖"（推荐）- 将现有文件备份为 CLAUDE.md.backup.{timestamp}
     - 选项 2: "直接覆盖" - 直接覆盖现有文件
     - 选项 3: "取消操作" - 退出命令
   - 如果用户选择取消，立即终止执行

b. 检查 CLAUDE_TEMPLATE.md 是否存在
   - 使用 Bash 的 test 命令检查文件是否存在
   - 如果不存在，输出错误信息：
     ```
     ❌ 错误：未找到 CLAUDE_TEMPLATE.md 模板文件

     请确保在当前目录或父目录中存在 CLAUDE_TEMPLATE.md 文件。
     您可以从以下位置获取模板：
     - 项目根目录
     - ~/.claude/CLAUDE_TEMPLATE.md
     ```
   - 终止执行

## 2. 读取模板

使用 Read 工具读取 CLAUDE_TEMPLATE.md 的完整内容，保存到变量中供后续使用。

## 3. 信息收集

使用 AskUserQuestion 工具分批收集用户输入，每批问题相关联。注意：每个选项都要提供详细的 description 说明。

### 第一批：项目核心信息

询问以下信息（必填）：

**问题 1：项目名称**
- header: "项目名称"
- question: "请输入项目名称："
- options:
  - 使用当前目录名作为默认值
  - 自定义输入（Other）

**问题 2：项目描述**
- header: "项目描述"
- question: "请用一句话描述项目："
- options: 提供 3-4 个常见项目类型示例，用户可选择或自定义
  - "Web 应用后端服务"
  - "前端用户界面应用"
  - "数据处理和分析系统"
  - "微服务架构项目"
  - 自定义（Other）

### 第二批：技术栈信息

**问题 3：开发语言**
- header: "开发语言"
- question: "项目主要使用什么开发语言？"
- options:
  - "Go 1.21+"（description: "高性能并发处理，适合后端服务"）
  - "Python 3.10+"（description: "数据处理和 AI 应用首选"）
  - "TypeScript 5.0+"（description: "类型安全的前端开发"）
  - "Java 17+"（description: "企业级应用开发"）
  - 自定义（Other）

**问题 4：主要框架**
- header: "框架选择"
- question: "项目使用什么主要框架？"
- options: 根据上一步选择的语言，提供对应的框架选项
  - Go: "Gin / Echo / Fiber / 其他"
  - Python: "FastAPI / Django / Flask / 其他"
  - TypeScript: "Next.js / React / Vue / 其他"
  - Java: "Spring Boot / Micronaut / Quarkus / 其他"

**问题 5：数据存储**
- header: "数据存储"
- question: "项目使用什么数据存储技术？"
- multiSelect: true（允许多选）
- options:
  - "MySQL / PostgreSQL"（description: "关系型数据库"）
  - "MongoDB / DynamoDB"（description: "文档数据库"）
  - "Redis / Memcached"（description: "缓存系统"）
  - "Elasticsearch"（description: "搜索引擎"）
  - 自定义（Other）

### 第三批：架构配置

**问题 6：项目架构**
- header: "架构模式"
- question: "项目采用什么架构模式？"
- options:
  - "分层架构（MVC/三层架构）"（description: "Controller → Service → DAO，适合传统应用"）
  - "微服务架构"（description: "服务拆分，独立部署和扩展"）
  - "Serverless"（description: "事件驱动，按需计费"）
  - "单体应用"（description: "简单直接，适合小型项目"）
  - 自定义（Other）

**问题 7：代码风格规范**
- header: "命名规范"
- question: "项目采用什么命名和代码风格规范？"
- options:
  - "驼峰命名（camelCase）"
  - "帕斯卡命名（PascalCase）"
  - "下划线命名（snake_case）"
  - "遵循语言官方规范"（推荐）
  - 自定义（Other）

### 第四批：开发规则配置

**问题 8：测试覆盖率要求**
- header: "测试要求"
- question: "是否启用测试覆盖率要求？"
- options:
  - "启用（要求 >80%）"（description: "严格测试要求，保证代码质量"）
  - "启用（要求 >60%）"（description: "中等测试要求"）
  - "不强制要求"（description: "允许灵活处理"）

**问题 9：Git 提交流程**
- header: "提交规范"
- question: "Git 提交是否需要审批确认？"
- options:
  - "需要（使用 AskUserQuestion）"（description: "提交前必须询问用户确认"）
  - "不需要"（description: "允许直接提交"）

## 4. 生成 CLAUDE.md

根据收集的信息，执行以下步骤：

a. 创建变量映射表：
   ```
   [项目名称] → 用户输入的项目名称
   [项目一句话描述] → 用户输入的项目描述
   [框架名称和版本] → 用户选择的框架
   [数据库/缓存技术] → 用户选择的数据存储（多选则用逗号连接）
   [语言和版本] → 用户选择的开发语言
   [命名规范] → 用户选择的代码风格
   ```

b. 对模板内容进行字符串替换：
   - 使用 Read 工具读取的模板内容
   - 逐个替换所有 [占位符] 为实际值
   - 对于架构图部分，根据用户选择的架构模式提供示例

c. 添加生成时间戳：
   - 使用 Bash date 命令获取当前日期
   - 替换模板最后的 [日期] 为实际日期

d. 根据用户的测试和 Git 配置，调整对应章节：
   - 如果不要求测试覆盖率，修改测试规范章节
   - 如果不需要 Git 审批，删除或注释相关规则

## 5. 文件写入

a. 如果用户在步骤 1 中选择了"备份后覆盖"：
   - 使用 Bash cp 命令将现有 CLAUDE.md 备份为 CLAUDE.md.backup.{timestamp}
   - 输出备份信息

b. 使用 Write 工具将生成的内容写入 CLAUDE.md 文件

## 6. 确认完成

输出成功信息，格式如下：

```
✅ 项目规范文件生成成功！

📄 文件位置: ./CLAUDE.md
📅 生成时间: {当前时间}

📋 配置摘要:
- 项目名称: {项目名称}
- 开发语言: {语言}
- 主要框架: {框架}
- 架构模式: {架构}

💡 下一步操作:
1. 查看生成的 CLAUDE.md 文件
2. 根据项目实际情况调整细节
3. 提交到版本控制系统

🤖 Generated with Claude Code
```

## 注意事项

1. **用户体验优先**：
   - 每个问题的 options 都必须提供详细的 description
   - 提供合理的默认值，允许用户快速完成
   - 问题顺序要符合逻辑（基础信息 → 技术栈 → 架构 → 规则）

2. **错误处理**：
   - 文件读写失败时，提供清晰的错误信息
   - 权限不足时，建议用户检查目录权限
   - 用户输入验证（避免空值或非法字符）

3. **灵活性**：
   - 所有占位符如果用户未提供，使用合理的默认值或留空
   - 允许用户跳过可选配置
   - 支持"Other"选项让用户自定义

4. **向后兼容**：
   - 备份机制确保不会丢失现有配置
   - 提供清晰的操作提示和确认

5. **遵循 KISS 原则**：
   - 不进行过度验证
   - 不添加不必要的功能
   - 专注于核心任务：读取模板 → 收集信息 → 生成文件
