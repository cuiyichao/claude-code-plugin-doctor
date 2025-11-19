# Project Doctor 插件

**您的私人项目健康顾问**。

Project Doctor 不仅仅是一个 Linter。它采用与 `project-init` 相同的深度分析方法，结合架构感知能力，能够深入理解你的业务逻辑代码，发现那些编译器和普通静态分析工具发现不了的**功能性 Bug**。

## ✨ 核心功能

1.  **📊 智能架构分析**（v2.0 增强）:
    - **Monorepo 支持**: 自动检测 Monorepo 结构（workspaces、Lerna、前后端分离等）
    - **多语言检测**: 完整支持 Node.js、Go、Python、Java 等主流语言
    - **深度技术栈分析**: 精确识别框架、数据库、缓存、消息队列等
    - **架构模式推断**: 自动识别分层架构、微服务、Serverless 等模式
    - **模块清单生成**: 详细列出所有模块、文件数、依赖关系
    - 生成标准化的项目全景视图和架构图

2.  **🔍 按模块深度诊断**（v2.0 重构）:
    - **模块级系统化分析**: 对每个模块（Controller、Service、Model、Utils）独立深度分析
    - **职责合规性检查**: 验证每个模块是否遵守架构分层原则
    - **代码质量全面检查**: 错误处理、数据验证、资源管理、性能、安全
    - **依赖关系分析**: 检测循环依赖、跨层调用等架构问题
    - **场景感知**: 自动区分前端/后端项目，应用不同的诊断策略
    - **全链路功能分析**: 以业务功能为维度，跨文件追踪逻辑漏洞
    - **非语法层面**: 专注于事务一致性、并发问题、资源泄露和错误传播

3.  **📝 详细诊断报告生成**（v2.0 增强）:
    - **执行摘要**: 诊断统计、问题分类、健康度评分
    - **项目全景视图**: 完整技术栈、架构图、模块清单与职责定义
    - **按模块诊断结果**: 每个模块的健康度、发现的问题、代码示例、修复建议
    - **按功能深度分析**: 完整调用链路、风险评估、修复优先级
    - **代码质量评估**: 质量指标、架构评分、综合健康度
    - **修复建议与行动计划**: 按优先级排序，包含预计耗时和测试方案
    - 一键生成 `PROJECT_DIAGNOSIS.md`，专业级诊断报告

## 🚀 使用方法

### 安装
将本插件复制到你的 Claude Code 插件目录：

```bash
# 1. 确保插件目录存在
mkdir -p ~/.claude/plugins

# 2. 安装插件（假设你在 claude-code-third-party-plugins 根目录）
# 注意使用绝对路径以避免错误
cp -r "$(pwd)/plugins/project-doctor" ~/.claude/plugins/
```

### 运行诊断
1. **重启** Claude Code 会话（如果已打开）。
2. 在任何项目中运行：

```bash
/project-doctor
```

Claude 将会：
1. 自动分析项目架构。
2. 让你确认业务领域和核心功能。
3. 执行全链路扫描。
4. 生成诊断报告。

## 🎯 适用场景

- **接手新项目时**: 快速了解架构并排查“地雷”。
- **代码审查 (Code Review) 前**: 让 AI 先过一遍，减少低级逻辑错误。
- **测试用例设计**: 基于识别出的核心功能和潜在 Bug 设计测试点。

## ✨ v2.0 新特性

### 🎯 更完整的项目分析
- 采用 `project-init` 插件的成熟分析引擎
- 支持 Monorepo 项目深度分析
- 多语言、多框架智能识别
- 模块清单自动生成

### 🔬 更深度的模块诊断
- **模块级分析**: 不再是简单的文件级检查，而是对整个模块进行系统化诊断
- **职责合规性**: 检查每个模块是否遵守分层架构原则
- **代码质量全面**: 5大维度检查（错误处理、数据验证、资源管理、性能、安全）
- **依赖关系可视化**: 清晰展示模块间依赖和问题

### 📊 更专业的诊断报告
- **6大章节**: 执行摘要、项目全景、模块诊断、功能分析、质量评估、行动计划
- **问题分级**: Critical、High、Medium、Low 四级分类
- **问题分类**: 安全、数据一致性、逻辑错误、性能、架构、代码质量
- **可执行建议**: 每个问题都包含详细的代码示例和修复方案
- **工作量评估**: 提供预计修复时长和优先级建议

---

## 📋 示例报告片段

### 1. 模块健康度总览

```markdown
## 🔍 2. 按模块诊断结果

### 2.1 Controller 层 (`src/controllers/`)

#### 📊 模块健康度: 85% ⭐⭐⭐⭐

**扫描文件**: 8 个  
**发现问题**: 5 个 (🔴 1 | 🟠 2 | 🟡 2)
```

### 2. 详细问题分析

```markdown
##### ⚠️ **OrderController.ts**

**🔴 [Critical] 参数校验缺失**
- **位置**: `OrderController.ts:45-52`
- **问题描述**: 
  ```typescript
  // 当前代码
  async createOrder(req: Request) {
    const { userId, items } = req.body;
    return await orderService.create(userId, items);
  }
  ```
  未验证 `userId` 是否存在，`items` 是否为空数组。
  
- **影响**: 可能导致创建无效订单，影响数据一致性
- **建议修复**:
  ```typescript
  async createOrder(req: Request) {
    const { userId, items } = req.body;
    
    // 添加参数验证
    if (!userId || !items || items.length === 0) {
      throw new ValidationError('Invalid request parameters');
    }
    
    return await orderService.create(userId, items);
  }
  ```
```

### 3. 全链路分析

```markdown
### 3.1 功能：创建订单 (POST /api/orders)

#### 调用链路

```
客户端请求
    ↓
OrderController.createOrder() [src/controllers/OrderController.ts:45]
    ↓
OrderService.create() [src/services/OrderService.ts:125]
    ├─→ InventoryService.deduct()
    ├─→ OrderRepository.create()
    └─→ NotificationService.send()
```

#### 风险评估

| 风险类型 | 严重程度 | 发生概率 | 影响范围 |
|---------|---------|---------|---------|
| 数据不一致 | 🔴 高 | 中 | 订单、库存 |
| 超卖问题 | 🔴 高 | 低 | 库存 |
| 无效订单 | 🟠 中 | 中 | 订单 |
```

### 4. 行动计划

```markdown
## 🛠️ 5. 修复建议与行动计划

### 5.1 立即修复（1-2天内）

**优先级 P0 - Critical 问题**

1. **修复 OrderService 事务一致性问题**
   - 文件: `src/services/OrderService.ts`
   - 预计耗时: 4 小时
   - 需要: 添加数据库事务支持
   - 测试: 模拟失败场景验证回滚
```
