---
allowed-tools: Read, Write, Bash, Glob, Grep, AskUserQuestion
description: 智能分析项目架构，生成规范文档，并深度扫描核心代码中的功能性 Bug
---

# Project Doctor - 项目深度审计

你是一位资深的技术专家（架构师级别）。你的任务是对当前项目进行“体检”。
体检分为两个阶段：**Phase 1: 结构与规范分析** 和 **Phase 2: 深度 Bug 挖掘**。

---

## Phase 1: 智能项目分析与架构建模

采用与 `project-init` 插件相同的深度分析方法，构建完整的项目架构视图。

---

## ⚠️ 关键原则：目录扫描限制

**绝对不允许向上级目录查找或扫描**

### 为什么？

考虑以下 Monorepo 结构：

```
myproject/
├── frontend/          ← 如果在这里运行 /project-doctor
│   ├── package.json   ← 只应分析这个
│   └── src/
├── backend/           ← 如果在这里运行 /project-doctor
│   ├── go.mod         ← 只应分析这个
│   └── src/
└── README.md
```

**正确行为**：
- ✅ 在 `myproject/frontend/` 运行 → 只分析前端项目
- ✅ 在 `myproject/backend/` 运行 → 只分析后端项目
- ✅ 在 `myproject/` 运行 → 检测到 Monorepo，分析所有子项目

**错误行为**：
- ❌ 在 `myproject/backend/` 运行却向上找到 `../frontend/package.json`
- ❌ 在 `myproject/frontend/` 运行却读取 `../backend/go.mod`
- ❌ 混淆前后端项目的配置和代码

### 实现规则

1. **配置文件读取**：
   - ✅ `Read ./package.json`
   - ✅ `Read ./go.mod`
   - ❌ `Read ../package.json`
   - ❌ `Read ../../config.yml`

2. **目录扫描**：
   - ✅ `Glob: ./src/**/*`
   - ✅ `Glob: */package.json`（向下查找子目录）
   - ❌ `Glob: ../**/*`
   - ❌ `Glob: ../../**/*`

3. **代码搜索**：
   - ✅ `Grep "import" --path ./src/`
   - ✅ `Grep "mysql"` （默认在当前目录）
   - ❌ 不指定路径时可能向上搜索

### 1.1 Monorepo 检测

**目标**: 智能检测项目是否为 Monorepo（包含多个子项目）

#### a. 检测显式 Monorepo 配置

```bash
# 1. 检测 Node.js workspaces
Read package.json → 检查 workspaces 字段

# 2. 检测 Lerna
Read lerna.json → 是否存在

# 3. 检测 Turborepo
Read turbo.json → 是否存在

# 4. 检测 pnpm workspaces
Read pnpm-workspace.yaml → 是否存在
```

#### b. 检测典型目录结构

```bash
# 检测前后端分离
Glob: frontend/, backend/
  - 检查是否有独立配置文件

# 检测 packages/apps 结构
Glob: packages/*/package.json
Glob: apps/*/package.json

# 检测 Go 多应用结构
Glob: cmd/* (多个子目录)

# 检测多服务结构
Glob: services/*/ (多个子目录)
```

#### c. 记录检测结果

```javascript
MonorepoDetection = {
  isMonorepo: true/false,
  confidence: "high" | "medium" | "low",
  type: "workspaces" | "lerna" | "frontend-backend" | "multi-packages" | ...,
  subProjectPaths: ["frontend/", "backend/", ...]
}
```

### 1.2 项目深度分析

对于单项目或 Monorepo 的每个子项目，执行以下分析：

#### a. 检测项目类型与技术栈

**⚠️ 重要原则：只在当前目录及子目录查找配置文件，绝不向上级目录查找**

原因：
- 避免在子项目中错误分析父项目
- 正确处理 Monorepo 结构（如 `myproject/backend/` 和 `myproject/frontend/` 是独立项目）
- 确保分析范围与用户运行命令的目录一致

**Node.js/TypeScript 项目**:
```bash
# 1. 先尝试读取当前目录的配置文件
Read ./package.json

# 2. 如果当前目录没有，使用 Glob 在子目录查找
Glob: */package.json
Glob: */*/package.json  # 最多向下两层

# 3. 绝不执行 Read ../package.json（向上查找）

提取信息:
  - name, description
  - dependencies 中的框架 (react, vue, next, express, nestjs)
  - devDependencies 中的 typescript
  - 测试框架 (jest, vitest, mocha)
  - 代码风格配置 (eslintConfig, prettier)
```

**Go 项目**:
```bash
# 1. 先尝试读取当前目录的配置文件
Read ./go.mod

# 2. 如果当前目录没有，使用 Glob 在子目录查找
Glob: */go.mod
Glob: */*/go.mod  # 最多向下两层

# 3. 绝不执行 Read ../go.mod（向上查找）

提取信息:
  - module 声明（项目名称）
  - go 版本
  - require 中的框架 (gin, echo, fiber)
```

**Python 项目**:
```bash
# 1. 先尝试读取当前目录的配置文件
Read ./pyproject.toml
Read ./requirements.txt
Read ./setup.py

# 2. 如果当前目录没有，使用 Glob 在子目录查找
Glob: */pyproject.toml
Glob: */requirements.txt

# 3. 绝不执行 Read ../pyproject.toml（向上查找）

提取信息:
  - 项目名称、版本
  - requires-python 版本
  - 依赖中的框架 (fastapi, django, flask)
  - 测试框架 (pytest, unittest)
```

**Java 项目**:
```bash
# 1. 先尝试读取当前目录的配置文件
Read ./pom.xml
Read ./build.gradle

# 2. 如果当前目录没有，使用 Glob 在子目录查找
Glob: */pom.xml
Glob: */build.gradle
Glob: */*/pom.xml  # 最多向下两层（Maven 多模块项目）

# 3. 绝不执行 Read ../pom.xml（向上查找）

提取信息:
  - artifactId / name
  - Java version
  - 依赖中的框架 (spring-boot, micronaut, quarkus)
```

#### b. 分析项目结构与架构模式

**⚠️ 重要原则：只扫描当前目录及子目录，不扫描上级目录**

```bash
# 使用 Glob 检测当前目录下的子目录结构（相对路径）
检测路径:
  - ./src/, ./lib/, ./pkg/ (源代码)
  - ./test/, ./tests/, ./__tests__/ (测试)
  - ./docs/ (文档)

# 使用相对路径 Glob，确保不会向上查找
Glob: ./src/**/*
Glob: ./lib/**/*
Glob: ./pkg/**/*

# 绝不使用 ../ 向上查找
# ❌ 错误示例: Glob: ../src/**/*
# ❌ 错误示例: Read ../src/config.ts

架构模式推断（基于当前目录）:
  - 存在 ./src/controllers, ./src/services, ./src/models → 分层架构（MVC）
  - 存在 ./src/api, ./src/service, ./src/logic, ./src/data → 四层架构
  - 存在 ./src/components, ./src/pages → 组件化架构（前端）
  - 存在 ./packages/, ./services/, ./apps/ → 微服务架构
  - 存在 ./functions/, ./lambda/ → Serverless
  - 存在 ./cmd/, ./internal/, ./pkg/ → Go 标准布局
```

#### c. 分析代码风格

```bash
# 读取 2-3 个代码文件样本
# 分析命名模式:
  - camelCase (userName, getUserData)
  - PascalCase (UserName, GetUserData)
  - snake_case (user_name, get_user_data)

# 检测 Lint 配置:
  - .eslintrc.* (JS/TS)
  - .golangci.yml (Go)
  - .flake8, pyproject.toml (Python)
```

#### d. 分析数据存储

**⚠️ 重要原则：只在当前目录及子目录搜索，不搜索上级目录**

```bash
# 使用 Grep 搜索数据库相关导入（限定在当前目录）
# 默认 Grep 只搜索当前目录及子目录，不会向上搜索

关键词:
  - "mysql", "pg", "postgres" → PostgreSQL/MySQL
  - "mongodb", "mongo" → MongoDB
  - "redis" → Redis
  - "elasticsearch" → Elasticsearch
  - "kafka", "rabbitmq" → 消息队列

# 检查配置文件（只在当前目录查找）
Read ./docker-compose.yml
Read ./.env.example
Read ./config/database.yml

# 或使用 Glob 在当前目录子目录查找
Glob: ./config/**/*.yml
Glob: ./.env*

# 绝不执行：
# ❌ Read ../docker-compose.yml
# ❌ Grep 时不指定路径限制
```

#### e. 检测模块与层级

**⚠️ 重要原则：只识别当前目录下的模块，不识别上级目录的模块**

**识别所有核心模块**:
```bash
# 使用 Glob 识别当前目录下的模块/包（相对路径）
Glob: ./src/controllers/*
Glob: ./src/services/*
Glob: ./src/models/*
Glob: ./src/utils/*

# 或使用更通用的模式
Glob: ./src/**/*.ts
Glob: ./src/**/*.go
Glob: ./src/**/*.py
Glob: ./src/**/*.java

# 为每个模块记录:
  - 模块路径（相对于当前目录）
  - 模块类型（Controller, Service, Logic, Model, Utils等）
  - 主要文件列表
  - 依赖关系

# 确保所有路径都是相对于当前目录的
# 示例：./src/controllers/UserController.ts
# 而不是：../src/controllers/UserController.ts
```

#### f. 记录完整分析结果

```javascript
ProjectAnalysis = {
  projectName: { value: "...", source: "...", confidence: "high" },
  projectDescription: { value: "...", source: "...", confidence: "medium" },
  language: { value: "TypeScript 5.0+", source: "...", confidence: "high" },
  framework: { value: "Next.js", source: "...", confidence: "high" },
  storage: { value: ["PostgreSQL", "Redis"], source: "...", confidence: "medium" },
  architecture: { value: "分层架构", source: "...", confidence: "medium" },
  namingStyle: { value: "camelCase", source: "...", confidence: "high" },
  
  // 模块清单
  modules: [
    { path: "src/controllers/", type: "Controller", files: [...] },
    { path: "src/services/", type: "Service", files: [...] },
    { path: "src/models/", type: "Model", files: [...] },
    { path: "src/utils/", type: "Utils", files: [...] }
  ],
  
  // 模块间依赖关系
  dependencies: [
    { from: "Controller", to: "Service", confidence: "high" },
    { from: "Service", to: "Model", confidence: "high" }
  ]
}
```

### 1.3 输出分析结果

```
🔍 项目深度分析完成！

📦 项目类型: [单项目 / Monorepo]
✅ 开发语言: TypeScript 5.0+
✅ 主要框架: Next.js 14
✅ 架构模式: 分层架构（MVC）
✅ 数据存储: PostgreSQL, Redis
✅ 命名规范: 驼峰命名 (camelCase)

📂 检测到的核心模块:
  - src/controllers/ (8 个文件) - Controller 层
  - src/services/ (12 个文件) - Service 层
  - src/models/ (15 个文件) - Model 层
  - src/utils/ (6 个文件) - 工具层

🔗 模块依赖关系:
  Controllers → Services → Models → Database
```

> **输出 1**: 在最终报告的开头，生成一个**标准化的项目架构视图**，包含 ASCII 架构图、详细的模块清单和层级职责说明。

**【交互步骤：确认业务领域】**
在进入 Phase 2 之前，使用 `AskUserQuestion` 确认你对项目业务领域的理解：

```yaml
question: "基于目录结构，我识别出本项目包含以下核心业务领域 (Business Domains)。这有助于我后续识别具体功能。请确认："
header: "业务领域识别"
multiSelect: true
options:
  - label: "📦 [领域1] (例如：Order Management)"
    description: "主要路径: src/modules/order"
  - label: "👤 [领域2] (例如：User System)"
    description: "主要路径: src/modules/user"
  - label: "🔧 [领域3] (例如：Inventory)"
    description: "主要路径: src/modules/inventory"
  - label: "✏️ 补充/修正"
    description: "识别不准，我要手动补充核心领域路径"
```

---

## Phase 2: 按模块深度诊断 (Module-Based Deep Analysis)

这是你的核心价值所在。基于 Phase 1 识别的模块和架构，**按模块进行系统化、深度的代码分析**。

### 2.1 诊断策略分流

根据 Phase 1 识别的 `Language` 和 `Framework`，选择诊断策略：

#### 🅰️ 策略 A：前端项目 (Frontend Focus)
*适用：React, Vue, Angular, Next.js, Flutter, Android, iOS 等*

**核心关注点 (User Interaction & State)**:
1.  **交互与状态管理**:
    - 状态更新是否会导致不必要的重渲染？(React.memo, useMemo)
    - 异步操作（API 调用）是否有 Loading/Error 状态处理？
    - 复杂表单的状态同步问题。
2.  **组件设计**:
    - Props 透传层级是否过深（Prop Drilling）？
    - Hooks 依赖项是否完整（useEffect deps）？
3.  **API 集成**:
    - 接口调用的异常捕获（try-catch / Promise.catch）。
    - 敏感数据是否暴露在前端？

#### 🅱️ 策略 B：后端项目 (Backend Focus)
*适用：Node.js API, Go (Gin/Echo), Python (FastAPI/Django), Java (Spring) 等*

**核心关注点 (API & Logic & Data)**:
1.  **接口与参数**:
    - 入参校验是否严谨？（防止 SQL 注入、越权）
    - 响应格式是否统一？
2.  **业务逻辑**:
    - 事务一致性（ACID）：多步数据库操作是否在同一事务中？
    - 并发控制：是否存在竞态条件（Race Condition）？
    - 异常处理：是否吞掉了底层异常？日志是否关键信息缺失？
3.  **性能隐患**:
    - 循环内的数据库查询（N+1 问题）。
    - 未关闭的资源（文件流、连接池）。

#### 🆎 策略 C：全栈/混合项目
*适用：Monorepo, Serverless 等*
- 分别对前端目录应用策略 A，对后端目录应用策略 B。

---

### 2.2 按模块系统化诊断

**核心原则**: 对 Phase 1 识别的每个模块进行**独立且深度**的分析。

#### 诊断流程

对于 ProjectAnalysis.modules 中的每个模块：

```javascript
// 遍历所有模块
for each module in ProjectAnalysis.modules:
  1. 读取模块所有文件
  2. 分析模块职责是否清晰
  3. 检查模块内代码质量
  4. 分析模块间依赖关系
  5. 记录发现的问题
```

#### Step 1: 模块级分析

**对于每个模块执行以下检查**：

**A. 职责分析** (根据模块类型):
```yaml
Controller 层:
  ✅ 应该做:
    - 接收和校验请求参数
    - 调用 Service 层处理业务
    - 返回统一格式的响应
  ❌ 不应该做:
    - 包含复杂业务逻辑
    - 直接操作数据库
    - 处理数据转换（应在 Service 层）

Service 层:
  ✅ 应该做:
    - 实现核心业务逻辑
    - 协调多个 Repository/Model
    - 处理事务边界
  ❌ 不应该做:
    - 处理 HTTP 请求细节
    - 直接构造 SQL 语句
    - 包含表现层逻辑

Model/Repository 层:
  ✅ 应该做:
    - 数据访问和持久化
    - 数据验证和转换
    - 封装数据库操作
  ❌ 不应该做:
    - 包含业务逻辑
    - 调用其他 Service
    - 处理 HTTP 相关逻辑

Utils 层:
  ✅ 应该做:
    - 提供通用工具函数
    - 无状态、可复用
  ❌ 不应该做:
    - 包含业务逻辑
    - 依赖特定模块
```

**B. 代码质量检查**:
```bash
# 对模块内每个文件执行:
1. 错误处理:
   - 是否有异常捕获？
   - 错误是否正确传播？
   - 日志是否完整？

2. 数据验证:
   - 入参是否验证？
   - 边界条件是否处理？
   - 空值是否检查？

3. 资源管理:
   - 数据库连接是否关闭？
   - 文件流是否关闭？
   - 是否有内存泄漏风险？

4. 性能问题:
   - 是否有 N+1 查询？
   - 循环内是否有重复操作？
   - 是否有不必要的数据复制？

5. 安全问题:
   - 是否有 SQL 注入风险？
   - 是否有 XSS 风险？
   - 敏感数据是否加密？
```

**C. 依赖关系分析**:
```bash
# 检查模块间调用是否合理
1. 分层合规性:
   - Controller 是否直接调用 Model？（违规）
   - Service 是否调用 Controller？（违规）
   - 是否存在循环依赖？（严重问题）

2. 耦合度分析:
   - 模块间依赖是否过多？
   - 是否可以解耦？
```

#### Step 2: 识别核心业务功能

基于模块分析结果，识别核心业务功能：

**后端项目** - 从 Controller 层识别：
```bash
# 分析 Controller 层的所有文件
# 提取所有 API 端点和对应的业务功能

示例:
  - POST /api/orders → "创建订单"
  - POST /api/payments/callback → "支付回调"
  - GET /api/reports → "生成报表"
```

**前端项目** - 从 Pages/Components 识别：
```bash
# 分析页面组件和关键交互

示例:
  - LoginForm.tsx → "用户登录"
  - CheckoutPage.tsx → "购物车结算"
  - Dashboard.tsx → "数据仪表盘"
```

#### Step 3: 功能确认交互

向用户展示识别出的**业务功能列表**：

```yaml
question: "基于模块分析，我识别出以下核心业务功能。请选择您希望重点诊断的功能（全链路分析）："
header: "核心功能诊断"
multiSelect: true
options:
  - label: "🛒 创建订单 (POST /api/orders)"
    description: "涉及: OrderController → OrderService → OrderRepository, InventoryService"
  - label: "💳 支付回调 (POST /api/payments/callback)"
    description: "涉及: PaymentController → PaymentService → OrderService"
  - label: "📊 生成报表 (GET /api/reports)"
    description: "涉及: ReportController → ReportService → Multiple Repositories"
  - label: "✏️ 手动指定功能"
    description: "由我输入要诊断的功能名称或入口文件"
```

---

### 2.3 全链路功能诊断

对于用户选中的每个功能，执行**完整调用链分析**：

#### Step 1: 构建调用链

```bash
1. 读取入口代码 (Controller/Page)
2. 分析函数调用和依赖导入
3. 递归追踪所有相关代码:
   - 入口 → Service → Repository/Model
   - 包含所有分支调用
4. 记录完整调用链路
```

#### Step 2: 跨层级逻辑分析

**在完整上下文中寻找跨文件的逻辑 Bug**：

**A. 数据流完整性**:
```bash
检查点:
  - 参数在层级传递中是否丢失？
  - 数据类型转换是否正确？
  - 必填字段是否被正确传递？

示例问题:
  ❌ Controller 传递了 userId，但 Service 没有使用
  ❌ Service 返回了 error，但 Controller 没有处理
```

**B. 信任边界问题**:
```bash
检查点:
  - 谁负责参数校验？
  - 谁负责权限检查？
  - 是否存在"都以为对方验证了"的问题？

示例问题:
  ❌ Controller 假设 Service 会验证参数
  ❌ Service 假设 Controller 已经验证参数
  → 结果：没有任何验证
```

**C. 事务一致性**:
```bash
检查点:
  - 多个数据库操作是否在同一事务中？
  - 失败时数据是否会回滚？
  - 是否有部分成功的风险？

示例问题:
  ❌ 扣减库存后，创建订单失败，库存未回滚
  ❌ 更新了订单状态，但发送通知失败，无补偿机制
```

**D. 错误传播路径**:
```bash
检查点:
  - 底层异常是否被正确捕获？
  - 错误信息是否被吞掉？
  - 日志是否记录关键信息？

示例问题:
  ❌ Repository 抛出异常，Service catch 后没有处理
  ❌ Service 返回 error，Controller 没有记录日志
```

**E. 并发安全**:
```bash
检查点:
  - 是否存在竞态条件？
  - 共享资源是否有锁保护？
  - 是否有幂等性保证？

示例问题:
  ❌ 多个请求同时扣减库存，可能超卖
  ❌ 重复支付回调，订单状态被重复更新
```

---

### 2.4 问题分类与优先级

**对发现的每个问题分类并评级**：

```yaml
严重等级:
  - [Critical] 严重: 可能导致数据丢失、安全漏洞、系统崩溃
    示例: SQL注入、事务不一致、内存泄漏
  
  - [High] 高: 影响核心功能，可能导致业务错误
    示例: 参数未验证、错误未处理、并发问题
  
  - [Medium] 中: 影响代码质量，可能影响性能或维护性
    示例: N+1查询、代码重复、耦合度高
  
  - [Low] 低: 代码风格或最佳实践问题
    示例: 命名不规范、注释缺失、日志不足

问题分类:
  - 🔒 安全问题 (Security)
  - 💾 数据一致性 (Data Consistency)
  - 🐛 逻辑错误 (Logic Bug)
  - ⚡ 性能问题 (Performance)
  - 🏗️ 架构问题 (Architecture)
  - 📝 代码质量 (Code Quality)
```

---

## Phase 3: 生成详细诊断报告

分析完成后，在根目录生成一份名为 `PROJECT_DIAGNOSIS.md` 的详细报告。

**报告模板：**

```markdown
# 🏥 Project Doctor 诊断报告

**生成时间**: YYYY-MM-DD HH:MM
**项目路径**: [项目路径]  
**项目类型**: [单项目 / Monorepo]  
**开发语言**: [语言和版本]  
**主要框架**: [框架名称和版本]  
**健康度评分**: [0-100] / 100

---

## 📊 执行摘要

### 诊断统计
- **扫描模块数**: [数量] 个
- **扫描文件数**: [数量] 个
- **代码行数**: 约 [数量] 行
- **发现问题数**: [数量] 个
  - 🔴 Critical: [数量]
  - 🟠 High: [数量]
  - 🟡 Medium: [数量]
  - 🟢 Low: [数量]

### 问题分类统计
- 🔒 安全问题: [数量]
- 💾 数据一致性: [数量]
- 🐛 逻辑错误: [数量]
- ⚡ 性能问题: [数量]
- 🏗️ 架构问题: [数量]
- 📝 代码质量: [数量]

---

## 🏗️ 1. 项目全景视图

### 1.1 核心技术栈

| 类型 | 技术 | 版本 | 来源 |
|------|------|------|------|
| 开发语言 | [语言] | [版本] | [检测来源] |
| 主要框架 | [框架] | [版本] | [检测来源] |
| 数据存储 | [数据库] | [版本] | [检测来源] |
| 缓存系统 | [缓存] | [版本] | [检测来源] |
| 消息队列 | [MQ] | [版本] | [检测来源] |

### 1.2 项目架构

```
[绘制完整的项目架构图]
例如：

客户端请求
    ↓
┌─────────────────────────────────────────┐
│  API Layer (Controller)                 │
│  - 请求参数接收与验证                    │
│  - 响应格式统一封装                      │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  Service Layer                          │
│  - 核心业务逻辑                          │
│  - 事务管理                              │
│  - 跨模块协调                            │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  Repository/Model Layer                 │
│  - 数据访问封装                          │
│  - 数据验证转换                          │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  Storage Layer                          │
│  PostgreSQL / Redis / MongoDB           │
└─────────────────────────────────────────┘
```

### 1.3 模块清单与职责

#### 模块总览

| 模块 | 路径 | 类型 | 文件数 | 代码行数 | 健康度 |
|------|------|------|--------|----------|--------|
| Controllers | `src/controllers/` | API 层 | 8 | ~1,200 | 85% ⭐⭐⭐⭐ |
| Services | `src/services/` | 业务层 | 12 | ~2,500 | 78% ⭐⭐⭐⭐ |
| Models | `src/models/` | 数据层 | 15 | ~1,800 | 92% ⭐⭐⭐⭐⭐ |
| Utils | `src/utils/` | 工具层 | 6 | ~500 | 95% ⭐⭐⭐⭐⭐ |

#### 详细职责定义

**Controller 层** (`src/controllers/`)
| 职责 | 说明 |
|------|------|
| ✅ **应该做** | • 接收和校验请求参数<br>• 调用 Service 层处理业务<br>• 返回统一格式的响应<br>• 记录访问日志 |
| ❌ **不应该做** | • 包含复杂业务逻辑<br>• 直接操作数据库<br>• 处理数据转换（应在 Service 层）<br>• 调用其他 Controller |

**Service 层** (`src/services/`)
| 职责 | 说明 |
|------|------|
| ✅ **应该做** | • 实现核心业务逻辑<br>• 协调多个 Repository/Model<br>• 处理事务边界<br>• 业务异常处理 |
| ❌ **不应该做** | • 处理 HTTP 请求细节<br>• 直接构造 SQL 语句<br>• 包含表现层逻辑<br>• 调用 Controller |

**Model/Repository 层** (`src/models/`)
| 职责 | 说明 |
|------|------|
| ✅ **应该做** | • 数据访问和持久化<br>• 数据验证和转换<br>• 封装数据库操作<br>• 定义数据模型 |
| ❌ **不应该做** | • 包含业务逻辑<br>• 调用 Service 层<br>• 处理 HTTP 相关逻辑<br>• 直接暴露给 Controller |

**Utils 层** (`src/utils/`)
| 职责 | 说明 |
|------|------|
| ✅ **应该做** | • 提供通用工具函数<br>• 无状态、可复用<br>• 格式化、验证等辅助功能 |
| ❌ **不应该做** | • 包含业务逻辑<br>• 依赖特定模块<br>• 有状态的操作 |

### 1.4 模块依赖关系

```
Controllers (8 文件)
    ├─→ OrderController.ts
    │   ├─→ OrderService
    │   └─→ InventoryService
    │
    ├─→ PaymentController.ts
    │   ├─→ PaymentService
    │   └─→ OrderService
    │
    └─→ UserController.ts
        └─→ UserService

Services (12 文件)
    ├─→ OrderService.ts
    │   ├─→ OrderRepository
    │   ├─→ InventoryService
    │   └─→ NotificationService
    │
    ├─→ PaymentService.ts
    │   ├─→ PaymentRepository
    │   └─→ OrderRepository
    │
    └─→ UserService.ts
        └─→ UserRepository

Models/Repositories (15 文件)
    ├─→ OrderRepository.ts → Database
    ├─→ PaymentRepository.ts → Database
    └─→ UserRepository.ts → Database
```

**⚠️ 检测到的依赖问题**:
- [如有] 循环依赖: ServiceA ↔ ServiceB
- [如有] 跨层调用: ControllerX 直接调用 RepositoryY

---

## 🔍 2. 按模块诊断结果

### 2.1 Controller 层 (`src/controllers/`)

#### 📊 模块健康度: 85% ⭐⭐⭐⭐

**扫描文件**: 8 个  
**发现问题**: 5 个 (🔴 1 | 🟠 2 | 🟡 2)

#### 发现的问题

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
  未验证 `userId` 是否存在，`items` 是否为空数组。恶意用户可传入空数据导致业务错误。
  
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

**🟠 [High] 错误处理不完整**
- **位置**: `OrderController.ts:78`
- **问题描述**: Service 层可能抛出异常，但 Controller 未捕获，导致返回 500 错误且无日志
- **影响**: 用户体验差，问题难以追踪
- **建议**: 添加 try-catch 并记录错误日志

##### ✅ **UserController.ts** - 无问题

##### ✅ **PaymentController.ts** - 无问题

---

### 2.2 Service 层 (`src/services/`)

#### 📊 模块健康度: 78% ⭐⭐⭐⭐

**扫描文件**: 12 个  
**发现问题**: 8 个 (🔴 2 | 🟠 3 | 🟡 3)

#### 发现的问题

##### ⚠️ **OrderService.ts**

**🔴 [Critical] 事务一致性缺失**
- **位置**: `OrderService.ts:125-145`
- **问题描述**:
  ```typescript
  async createOrder(userId: string, items: Item[]) {
    // 1. 扣减库存
    await inventoryService.deduct(items);
    
    // 2. 创建订单 (如果这里失败，库存已扣减，无法回滚)
    const order = await orderRepository.create({
      userId,
      items,
      status: 'pending'
    });
    
    // 3. 发送通知 (如果这里失败，订单已创建，通知未发送)
    await notificationService.send(userId, order.id);
    
    return order;
  }
  ```
  **问题**: 三个操作不在同一事务中，任何步骤失败都会导致数据不一致。
  
- **影响**: 
  - 库存扣减成功但订单创建失败 → 库存数据错误
  - 订单创建成功但通知发送失败 → 用户未收到通知
  
- **建议修复**:
  ```typescript
  async createOrder(userId: string, items: Item[]) {
    // 使用数据库事务
    return await db.transaction(async (trx) => {
      // 1. 扣减库存
      await inventoryService.deduct(items, { transaction: trx });
      
      // 2. 创建订单
      const order = await orderRepository.create({
        userId,
        items,
        status: 'pending'
      }, { transaction: trx });
      
      // 3. 通知操作放到事务外，失败后补偿
      await this.sendNotificationAsync(userId, order.id);
      
      return order;
    });
  }
  ```

**🟠 [High] 并发安全问题**
- **位置**: `OrderService.ts:180-195`
- **问题描述**: 库存扣减没有使用锁，多个请求同时扣减可能导致超卖
- **影响**: 库存管理混乱，可能出现负库存
- **建议**: 使用数据库行锁或分布式锁

---

### 2.3 Model/Repository 层 (`src/models/`)

#### 📊 模块健康度: 92% ⭐⭐⭐⭐⭐

**扫描文件**: 15 个  
**发现问题**: 2 个 (🟡 2)

#### 发现的问题

**🟡 [Medium] N+1 查询问题**
- **位置**: `OrderRepository.ts:58-65`
- **问题描述**: 在循环中查询关联数据
- **影响**: 性能下降，数据库负载高
- **建议**: 使用 JOIN 或批量查询

---

## 🐛 3. 按功能深度分析

### 3.1 功能：创建订单 (POST /api/orders)

#### 调用链路

```
客户端请求
    ↓
OrderController.createOrder() [src/controllers/OrderController.ts:45]
    ↓
OrderService.create() [src/services/OrderService.ts:125]
    ├─→ InventoryService.deduct() [src/services/InventoryService.ts:80]
    │       ↓
    │   InventoryRepository.updateStock() [src/models/InventoryRepository.ts:45]
    │
    ├─→ OrderRepository.create() [src/models/OrderRepository.ts:30]
    │
    └─→ NotificationService.send() [src/services/NotificationService.ts:20]
```

#### 🚨 发现的关键问题

**1. [Critical] 事务一致性缺失**  
详见 `2.2 Service 层 → OrderService.ts`

**2. [High] 并发安全问题**  
详见 `2.2 Service 层 → OrderService.ts`

**3. [Critical] 参数校验缺失**  
详见 `2.1 Controller 层 → OrderController.ts`

#### 风险评估

| 风险类型 | 严重程度 | 发生概率 | 影响范围 |
|---------|---------|---------|---------|
| 数据不一致 | 🔴 高 | 中 | 订单、库存 |
| 超卖问题 | 🔴 高 | 低 | 库存 |
| 无效订单 | 🟠 中 | 中 | 订单 |

#### 修复优先级

1. **立即修复**: 事务一致性问题（影响数据准确性）
2. **高优先级**: 并发安全问题（影响业务准确性）
3. **中优先级**: 参数校验（影响系统健壮性）

---

### 3.2 功能：支付回调 (POST /api/payments/callback)

[类似格式展示其他功能的分析]

---

## 📈 4. 代码质量评估

### 4.1 质量指标

| 指标 | 当前值 | 目标值 | 评级 |
|------|--------|--------|------|
| 测试覆盖率 | 65% | 80% | ⭐⭐⭐ |
| 代码重复率 | 12% | <5% | ⭐⭐⭐ |
| 平均圈复杂度 | 8.5 | <10 | ⭐⭐⭐⭐ |
| 平均函数长度 | 45 行 | <50 行 | ⭐⭐⭐⭐ |
| 技术债时长 | 8 天 | <5 天 | ⭐⭐⭐ |

### 4.2 架构评分

| 维度 | 评分 | 说明 |
|------|------|------|
| 🏗️ 架构清晰度 | 85/100 | 分层清晰，职责基本明确 |
| 🔗 模块耦合度 | 78/100 | 部分模块耦合较重 |
| 🔒 安全性 | 70/100 | 存在输入验证不足 |
| ⚡ 性能 | 75/100 | 存在 N+1 查询问题 |
| 🧪 可测试性 | 80/100 | 依赖注入较好 |
| 📝 代码可读性 | 82/100 | 命名规范，注释适中 |

**综合健康度**: **78/100** ⭐⭐⭐⭐

---

## 🛠️ 5. 修复建议与行动计划

### 5.1 立即修复（1-2天内）

**优先级 P0 - Critical 问题**

1. **修复 OrderService 事务一致性问题**
   - 文件: `src/services/OrderService.ts`
   - 预计耗时: 4 小时
   - 需要: 添加数据库事务支持
   - 测试: 模拟失败场景验证回滚

2. **修复 OrderController 参数校验缺失**
   - 文件: `src/controllers/OrderController.ts`
   - 预计耗时: 2 小时
   - 需要: 添加输入验证中间件
   - 测试: 单元测试覆盖边界情况

### 5.2 高优先级（本周内）

**优先级 P1 - High 问题**

3. **修复库存并发安全问题**
   - 文件: `src/services/OrderService.ts`, `src/services/InventoryService.ts`
   - 预计耗时: 6 小时
   - 需要: 实现分布式锁或数据库悲观锁
   - 测试: 并发测试验证

4. **完善错误处理和日志记录**
   - 文件: 所有 Controller 文件
   - 预计耗时: 4 小时
   - 需要: 统一错误处理中间件
   - 测试: 异常场景测试

### 5.3 中优先级（2周内）

**优先级 P2 - Medium 问题**

5. **优化 N+1 查询问题**
   - 文件: `src/models/OrderRepository.ts`
   - 预计耗时: 3 小时
   - 需要: 重构为 JOIN 查询或批量查询
   - 测试: 性能测试对比

6. **降低模块耦合度**
   - 文件: 多个 Service 文件
   - 预计耗时: 8 小时
   - 需要: 引入事件驱动或依赖注入
   - 测试: 单元测试覆盖

### 5.4 长期优化（1个月内）

7. **提升测试覆盖率到 80%**
8. **建立代码审查checklist**
9. **引入静态代码分析工具**
10. **完善监控和告警**

---

## 📋 6. 总结

### 6.1 主要发现

✅ **做得好的地方**:
- 项目架构清晰，分层合理
- 代码风格统一，命名规范
- 大部分模块职责明确
- 基础的错误处理存在

⚠️ **需要改进的地方**:
- **Critical**: 事务一致性问题需立即修复
- **High**: 并发安全和参数校验需要加强
- **Medium**: 性能优化和代码重复需要关注

### 6.2 风险提示

🚨 **高风险项**:
1. 订单创建流程的事务问题可能导致数据不一致
2. 库存管理的并发问题可能导致超卖
3. 参数验证不足可能被恶意利用

### 6.3 后续建议

1. **立即行动**: 修复所有 Critical 和 High 级别问题
2. **建立规范**: 制定代码审查 checklist，防止类似问题再次出现
3. **持续改进**: 定期运行 Project Doctor，保持代码健康
4. **加强测试**: 提升测试覆盖率，特别是关键业务流程

---

**诊断完成时间**: YYYY-MM-DD HH:MM  
**预计修复总时长**: ~35 小时  
**建议修复周期**: 2-3 周

🤖 **Generated by Project Doctor** - Claude Code Plugin
```

## 执行逻辑

### Phase 1: 项目分析阶段

1. **初始化**
   ```
   🔍 Project Doctor 正在启动...
   📋 准备对项目进行深度健康检查
   ```

2. **Monorepo 检测** (步骤 1.1)
   ```
   🔎 检测项目结构...
   ```
   - 检测显式 Monorepo 配置
   - 检测典型目录结构
   - 检测多配置文件
   - 输出检测结果

3. **项目深度分析** (步骤 1.2)
   ```
   📊 正在分析项目架构...
   ```
   - 检测项目类型与技术栈
   - 分析项目结构与架构模式
   - 分析代码风格
   - 分析数据存储
   - 检测模块与层级
   - 记录完整分析结果

4. **输出分析结果** (步骤 1.3)
   ```
   ✅ 项目分析完成！
   
   [显示检测到的技术栈、架构、模块等信息]
   ```

5. **确认业务领域** (交互)
   ```
   使用 AskUserQuestion 确认识别出的业务领域
   ```

---

### Phase 2: 模块诊断阶段

6. **选择诊断策略** (步骤 2.1)
   ```
   🎯 根据项目类型选择诊断策略...
   策略: [Frontend / Backend / Fullstack]
   ```

7. **按模块系统化诊断** (步骤 2.2)
   ```
   🔍 正在按模块进行深度分析...
   
   [1/4] 分析 Controller 层 (8 个文件)...
   [2/4] 分析 Service 层 (12 个文件)...
   [3/4] 分析 Model 层 (15 个文件)...
   [4/4] 分析 Utils 层 (6 个文件)...
   ```
   
   对每个模块：
   - 读取模块所有文件
   - 职责分析
   - 代码质量检查
   - 依赖关系分析
   - 记录发现的问题

8. **识别核心业务功能** (步骤 2.2 - Step 2)
   ```
   🎯 识别核心业务功能...
   
   从 Controller 层识别到 [数量] 个核心功能
   ```

9. **功能确认交互** (步骤 2.2 - Step 3)
   ```
   使用 AskUserQuestion 让用户选择要深度诊断的功能
   ```

10. **全链路功能诊断** (步骤 2.3)
    ```
    🔬 正在进行全链路分析...
    
    [1/3] 构建调用链: 创建订单...
    [2/3] 构建调用链: 支付回调...
    [3/3] 构建调用链: 生成报表...
    ```
    
    对每个功能：
    - 构建完整调用链
    - 跨层级逻辑分析（数据流、信任边界、事务、错误、并发）
    - 问题分类与优先级

11. **诊断完成总结**
    ```
    ✅ 模块诊断完成！
    
    📊 诊断统计:
    - 扫描模块: [数量] 个
    - 扫描文件: [数量] 个
    - 发现问题: [数量] 个
      • Critical: [数量]
      • High: [数量]
      • Medium: [数量]
      • Low: [数量]
    ```

---

### Phase 3: 报告生成阶段

12. **生成诊断报告** (步骤 3)
    ```
    📝 正在生成详细诊断报告...
    ```
    
    - 整合所有分析结果
    - 生成项目全景视图
    - 生成按模块诊断结果
    - 生成按功能深度分析
    - 生成代码质量评估
    - 生成修复建议与行动计划
    - 生成总结
    - 使用 `Write` 工具生成 `PROJECT_DIAGNOSIS.md`

13. **结束**
    ```
    ✅ 诊断完成！
    
    📄 报告位置: ./PROJECT_DIAGNOSIS.md
    📊 健康度评分: [分数]/100
    
    🚨 发现 [数量] 个 Critical 问题，[数量] 个 High 问题
    💡 建议优先修复 Critical 和 High 级别问题
    ```

14. **后续行动询问** (可选)
    ```
    使用 AskUserQuestion 询问:
    "是否需要立即协助修复某个 Critical 或 High 优先级问题？"
    
    选项:
    - 是，修复 [问题1描述]
    - 是，修复 [问题2描述]
    - 否，我先查看报告
    ```

---

## 注意事项

1. **性能考虑**:
   - 如果模块文件过多（>100 个），考虑采样分析
   - 大文件（>500 行）只读取关键部分
   - 合理使用批量读取减少工具调用次数

2. **用户体验**:
   - 实时显示进度，让用户了解当前状态
   - 对耗时操作给出预期时间
   - 提供明确的交互选项

3. **报告质量**:
   - 每个问题必须包含：位置、描述、影响、建议
   - 代码示例必须准确，来自实际代码
   - 修复建议必须可执行，包含代码示例
   - 优先级评定必须合理

4. **错误处理**:
   - 如果无法读取某个文件，记录并继续
   - 如果分析失败，提供降级方案
   - 始终生成报告，即使部分分析失败

```

