---
allowed-tools: Read, Write, Bash, Glob, Grep, AskUserQuestion
description: 智能分析项目架构，生成规范文档，并深度扫描核心代码中的功能性 Bug
---

# Project Doctor - 需求驱动的项目诊断

你是一位资深的技术专家（架构师级别）。你的任务是**基于用户提供的测试需求/功能点**，对项目进行针对性的代码审查。

## 🎯 核心理念

**需求驱动 > 盲目扫描**

传统方式（盲目扫描全部代码）会产生大量误报，用户需要花时间甄别哪些是真问题。

新方式：
1. 用户提供**明确的测试需求点/功能点**
2. 插件分析项目架构，定位相关代码
3. 针对每个需求点，验证实现是否正确
4. 生成**"需求 vs 实现"的对照诊断报告**

**诊断流程**：用户需求 → 项目分析 → 需求定位 → 针对性审查 → 对照报告

---

## Phase 0: 收集测试需求 (Requirement Collection) 🆕

**这是诊断的起点，决定了后续的精准度！**

### 0.1 询问用户需求

使用 `AskUserQuestion` 询问用户要测试的功能点：

```yaml
question: |
  请提供您想要诊断的功能点或测试需求。这将帮助我进行精准的代码审查，而不是盲目扫描。
  
  您可以提供：
  1. 具体的功能需求（例如："用户注册时需要发送邮件验证码"）
  2. 业务流程（例如："订单创建后，扣减库存，发送通知"）
  3. API 接口要求（例如："POST /api/orders 需要校验库存充足"）
  4. 测试用例（例如："测试并发下单不会超卖"）
  5. 已知问题线索（例如："支付回调有时会重复处理"）
  
  请输入您的测试需求（支持多行，一行一个需求点）：

header: "📋 测试需求收集"
multiline: true
```

### 0.2 解析需求并确认

**将用户输入的需求解析为结构化格式**：

```javascript
TestRequirements = [
  {
    id: "REQ-001",
    title: "用户注册邮件验证",
    description: "用户注册时需要发送邮件验证码",
    type: "功能需求", // 功能需求 | 性能需求 | 安全需求 | 数据一致性
    keywords: ["注册", "邮件", "验证码"],
    estimatedEntryPoint: "POST /api/register" // 初步推测的入口
  },
  {
    id: "REQ-002",
    title: "订单创建流程",
    description: "订单创建后，扣减库存，发送通知",
    type: "业务流程",
    keywords: ["订单", "库存", "通知"],
    estimatedEntryPoint: "POST /api/orders"
  },
  // ...
]
```

**向用户展示解析结果并确认**：

```yaml
question: |
  我已将您的需求解析为以下测试点，请确认：
  
  [REQ-001] 用户注册邮件验证
  - 描述：用户注册时需要发送邮件验证码
  - 预估入口：POST /api/register
  
  [REQ-002] 订单创建流程
  - 描述：订单创建后，扣减库存，发送通知
  - 预估入口：POST /api/orders
  
  是否正确？

header: "确认测试需求"
options:
  - label: "✅ 正确，开始诊断"
  - label: "✏️ 需要修改需求"
  - label: "➕ 添加更多需求"
```

### 0.3 输出确认信息

```
✅ 测试需求收集完成！

📋 待诊断需求点: 2 个
  [REQ-001] 用户注册邮件验证
  [REQ-002] 订单创建流程

🔍 接下来将分析项目架构，定位相关代码...
```

---

## Phase 1: 快速项目分析与需求定位

**目标**：快速了解项目结构，为每个需求点定位相关代码。

### 1.0 分析策略

**不再全面扫描，而是针对性定位**：

1. **快速识别项目类型**（前端/后端/全栈）
2. **定位关键目录**（Controller/Service/Model 等）
3. **基于需求关键词搜索**，定位相关代码
4. **构建需求到代码的映射关系**

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
   - ✅ `Grep "mysql" --path ./` （明确指定当前目录）
   - ❌ `Grep "mysql"` （不指定路径可能默认行为不明确）
   - ❌ `Grep "import" --path ../` （向上搜索）
   
**重要提示**：所有 `Grep`、`Read`、`Glob` 命令都必须使用 `./` 前缀或明确的相对路径，确保只在当前工作目录及其子目录内操作。

### 1.1 Monorepo 检测

**目标**: 智能检测项目是否为 Monorepo（包含多个子项目）

#### a. 检测显式 Monorepo 配置

**⚠️ 只检测当前目录的配置文件，不向上查找**

```bash
# 1. 检测 Node.js workspaces（只读取当前目录）
Read ./package.json → 检查 workspaces 字段

# 2. 检测 Lerna（只读取当前目录）
Read ./lerna.json → 是否存在

# 3. 检测 Turborepo（只读取当前目录）
Read ./turbo.json → 是否存在

# 4. 检测 pnpm workspaces（只读取当前目录）
Read ./pnpm-workspace.yaml → 是否存在
```

#### b. 检测典型目录结构

**⚠️ 只检测当前目录的子目录，不向上或向同级目录查找**

```bash
# 检测前后端分离（只在当前目录下查找）
Glob: ./frontend/
Glob: ./backend/
  - 检查是否有独立配置文件

# 检测 packages/apps 结构（只向下查找）
Glob: ./packages/*/package.json
Glob: ./apps/*/package.json

# 检测 Go 多应用结构（只向下查找）
Glob: ./cmd/* (当前目录下的 cmd 子目录)

# 检测多服务结构（只向下查找）
Glob: ./services/*/ (当前目录下的 services 子目录)
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

**⚠️ 重要原则：只读取当前目录及子目录的代码文件**

```bash
# 读取 2-3 个代码文件样本（只从当前目录的子目录读取）
# 使用 Glob 查找代码文件，然后 Read 前几个
Glob: ./src/**/*.ts    # TypeScript 项目
Glob: ./src/**/*.go    # Go 项目
Glob: ./src/**/*.py    # Python 项目
Glob: ./src/**/*.java  # Java 项目

# 然后 Read 找到的前 2-3 个文件进行代码风格分析
# 分析命名模式:
  - camelCase (userName, getUserData)
  - PascalCase (UserName, GetUserData)
  - snake_case (user_name, get_user_data)

# 检测 Lint 配置（只在当前目录查找）:
Read ./.eslintrc.js (JS/TS)
Read ./.eslintrc.json (JS/TS)
Read ./.golangci.yml (Go)
Read ./.flake8 (Python)
Read ./pyproject.toml (Python)

# 绝不执行：
# ❌ Read ../src/main.ts
# ❌ Glob: ../**/*.ts
```

#### d. 分析数据存储

**⚠️ 重要原则：只在当前目录及子目录搜索，不搜索上级目录**

```bash
# 使用 Grep 搜索数据库相关导入（必须限定在当前目录）
# ⚠️ 重要：必须明确指定 --path 参数，限制在当前目录及子目录

关键词搜索（必须指定路径）:
  Grep "mysql" --path ./    → PostgreSQL/MySQL
  Grep "pg" --path ./       → PostgreSQL
  Grep "postgres" --path ./ → PostgreSQL
  Grep "mongodb" --path ./  → MongoDB
  Grep "mongo" --path ./    → MongoDB
  Grep "redis" --path ./    → Redis
  Grep "elasticsearch" --path ./ → Elasticsearch
  Grep "kafka" --path ./    → 消息队列
  Grep "rabbitmq" --path ./ → 消息队列

# 检查配置文件（只在当前目录查找）
Read ./docker-compose.yml
Read ./.env.example
Read ./config/database.yml

# 或使用 Glob 在当前目录子目录查找
Glob: ./config/**/*.yml
Glob: ./.env*

# 绝不执行：
# ❌ Read ../docker-compose.yml
# ❌ Grep "mysql" (不指定路径可能向上搜索)
# ❌ Grep "mysql" --path ../ (向上搜索)
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

### 1.7 需求代码定位 🆕

**基于 Phase 0 收集的需求，使用关键词搜索定位相关代码**：

```bash
# 对每个需求点执行：
for each requirement in TestRequirements:
  # 1. 使用关键词在代码中搜索
  keywords = requirement.keywords
  
  # 示例：REQ-001 "用户注册邮件验证"
  # keywords: ["注册", "邮件", "验证码"]
  
  Grep "register" --path ./src/ -i
  Grep "email" --path ./src/ -i
  Grep "verification" --path ./src/ -i
  Grep "code" --path ./src/ -i
  
  # 2. 识别相关文件
  relatedFiles = [找到的文件列表]
  
  # 3. 如果有 API 路径，尝试定位 Controller
  if requirement.estimatedEntryPoint:
    # 例如：POST /api/register
    Grep "\/api\/register" --path ./src/ -i
    Grep "register.*route" --path ./src/ -i
  
  # 4. 记录映射关系
  requirement.relatedCode = {
    entryPoint: "src/controllers/AuthController.ts:registerUser()",
    relatedFiles: [
      "src/controllers/AuthController.ts",
      "src/services/AuthService.ts",
      "src/services/EmailService.ts",
      "src/models/User.ts"
    ],
    callChain: [
      "AuthController.registerUser()",
      "→ AuthService.createUser()",
      "→ EmailService.sendVerificationCode()",
      "→ UserRepository.create()"
    ]
  }
```

**【交互步骤：确认需求定位结果】**

```yaml
question: |
  我已为每个需求定位了相关代码，请确认：
  
  [REQ-001] 用户注册邮件验证
  📍 入口: src/controllers/AuthController.ts:registerUser()
  🔗 调用链:
    → AuthController.registerUser()
    → AuthService.createUser()
    → EmailService.sendVerificationCode()
    → UserRepository.create()
  📂 涉及文件: 4 个
  
  [REQ-002] 订单创建流程
  📍 入口: src/controllers/OrderController.ts:createOrder()
  🔗 调用链:
    → OrderController.createOrder()
    → OrderService.create()
    → InventoryService.deduct()
    → NotificationService.send()
  📂 涉及文件: 6 个
  
  是否正确？如果有遗漏或错误，我会重新定位。

header: "需求代码定位结果"
options:
  - label: "✅ 正确，开始针对性审查"
    value: "confirm"
  - label: "❌ [REQ-001] 定位错误，需要调整"
    value: "adjust_req_001"
  - label: "❌ [REQ-002] 定位错误，需要调整"
    value: "adjust_req_002"
  - label: "✏️ 手动指定文件路径"
    value: "manual"
```

### 1.8 输出定位结果

```
✅ 需求代码定位完成！

📊 定位统计:
  - 需求点: 2 个
  - 入口函数: 2 个
  - 相关文件: 10 个
  - 预计代码行数: ~1,200 行

🔬 接下来将对每个需求进行针对性审查...
```

---

## Phase 2: 按需求针对性审查 (Requirement-Driven Validation) 🆕

**这是核心价值所在！不再盲目扫描，而是针对每个需求点，验证实现是否正确。**

### 2.0 诊断策略

**对每个需求点执行以下审查**：

```javascript
for each requirement in TestRequirements:
  1. 读取相关代码文件
  2. 理解需求预期行为
  3. 验证实现是否符合需求
  4. 识别潜在问题（逻辑漏洞、边界情况、异常处理等）
  5. 记录"需求 vs 实现"的对比结果
```

---

### 2.1 需求类型诊断策略

**根据需求类型选择不同的审查重点**：

#### 🎯 策略 A：功能需求验证
*适用：用户注册、订单创建、支付回调等业务功能*

**审查清单**:
```yaml
1. 功能完整性:
   - ✅ 需求描述的所有步骤是否都实现了？
   - ✅ 是否有缺失的逻辑分支？
   
2. 参数验证:
   - ✅ 必填参数是否验证？
   - ✅ 数据类型是否检查？
   - ✅ 边界条件是否处理？（空值、负数、超长等）
   
3. 错误处理:
   - ✅ 异常是否被捕获？
   - ✅ 错误信息是否友好？
   - ✅ 日志是否记录关键信息？
   
4. 返回结果:
   - ✅ 返回数据是否完整？
   - ✅ 格式是否符合约定？

5. 副作用验证:
   - ✅ 数据库操作是否正确？
   - ✅ 第三方调用是否有fallback？
   - ✅ 通知/消息是否发送？
```

#### ⚡ 策略 B：性能需求验证
*适用：并发处理、大数据查询、实时响应等*

**审查清单**:
```yaml
1. 查询效率:
   - ✅ 是否有 N+1 查询？
   - ✅ 是否有不必要的全表扫描？
   - ✅ 索引是否合理？
   
2. 资源管理:
   - ✅ 连接池是否复用？
   - ✅ 资源是否及时释放？
   - ✅ 是否有内存泄漏风险？
   
3. 并发控制:
   - ✅ 是否有锁保护共享资源？
   - ✅ 锁粒度是否合理？
   - ✅ 是否有死锁风险？
```

#### 🔒 策略 C：安全需求验证
*适用：权限控制、数据加密、防注入等*

**审查清单**:
```yaml
1. 输入安全:
   - ✅ 是否防止 SQL 注入？
   - ✅ 是否防止 XSS 攻击？
   - ✅ 是否防止 CSRF 攻击？
   
2. 权限控制:
   - ✅ 是否验证用户身份？
   - ✅ 是否检查操作权限？
   - ✅ 是否防止越权访问？
   
3. 敏感数据:
   - ✅ 密码是否加密存储？
   - ✅ 敏感信息是否脱敏？
   - ✅ 日志是否包含敏感数据？
```

#### 💾 策略 D：数据一致性验证
*适用：事务处理、分布式操作、数据同步等*

**审查清单**:
```yaml
1. 事务边界:
   - ✅ 多个数据库操作是否在同一事务中？
   - ✅ 失败时是否会回滚？
   - ✅ 事务隔离级别是否合理？
   
2. 并发安全:
   - ✅ 是否有竞态条件？
   - ✅ 乐观锁/悲观锁是否正确使用？
   - ✅ 幂等性是否保证？
   
3. 数据完整性:
   - ✅ 外键关系是否一致？
   - ✅ 状态转换是否合法？
   - ✅ 数据校验是否完整？
```

---

### 2.2 按需求逐个审查 🆕

**核心原则**: 对每个需求点进行**针对性验证**，避免误报。

#### 审查流程

对于 TestRequirements 中的每个需求：

```javascript
// 遍历所有需求
for each requirement in TestRequirements:
  1. 读取需求相关的所有代码文件
  2. 理解需求的预期行为
  3. 根据需求类型选择审查策略（A/B/C/D）
  4. 验证实现是否符合需求
  5. 识别潜在问题和风险
  6. 记录"需求 vs 实现"的对比结果
```

#### 审查示例

**需求：[REQ-001] 用户注册邮件验证**

```bash
# Step 1: 读取相关代码
Read src/controllers/AuthController.ts
Read src/services/AuthService.ts
Read src/services/EmailService.ts
Read src/models/User.ts

# Step 2: 理解需求预期行为
需求描述：用户注册时需要发送邮件验证码

预期流程：
  1. 用户提交注册信息（邮箱、密码）
  2. 验证邮箱格式和密码强度
  3. 生成验证码
  4. 发送验证邮件
  5. 创建用户记录（状态：未验证）
  6. 返回成功响应

# Step 3: 选择审查策略
类型：功能需求 → 使用策略 A

# Step 4: 验证实现
审查清单：
  ✅ 参数验证
    - [检查] AuthController 是否验证邮箱格式？
    - [检查] 是否验证密码强度？
    - [检查] 是否验证邮箱唯一性？
  
  ✅ 核心逻辑
    - [检查] 是否生成验证码？
    - [检查] 是否调用 EmailService.send()?
    - [检查] 是否创建用户记录？
    - [检查] 用户状态是否设置为"未验证"？
  
  ✅ 错误处理
    - [检查] 邮件发送失败时如何处理？
    - [检查] 数据库写入失败时如何处理？
  
  ✅ 边界情况
    - [检查] 重复注册如何处理？
    - [检查] 邮箱为空如何处理？

# Step 5: 识别潜在问题

发现的问题：
  🔴 [Critical] AuthController.register() 没有验证邮箱格式
    位置：src/controllers/AuthController.ts:45
    风险：可以注册非法邮箱地址
  
  🟠 [High] EmailService.send() 失败时没有回滚用户创建
    位置：src/services/AuthService.ts:78
    风险：用户已创建但未收到验证邮件，无法激活
  
  🟡 [Medium] 验证码未设置过期时间
    位置：src/services/AuthService.ts:82
    风险：验证码可能被长期使用

# Step 6: 记录对比结果
RequirementValidationResult = {
  requirementId: "REQ-001",
  title: "用户注册邮件验证",
  overallStatus: "部分实现", // 完全实现 | 部分实现 | 未实现 | 实现错误
  implementationScore: 70, // 0-100
  issues: [
    { severity: "Critical", description: "..." },
    { severity: "High", description: "..." },
    { severity: "Medium", description: "..." }
  ],
  missingFeatures: ["邮箱格式验证", "失败回滚机制"],
  recommendations: [...]
}
```

### 2.3 审查进度展示

在审查过程中，实时显示进度：

```
🔬 正在按需求进行针对性审查...

[1/2] 审查 REQ-001: 用户注册邮件验证
  📂 读取代码文件 (4 个)...
  🔍 验证功能完整性...
  ⚠️  发现 3 个问题 (🔴 1 | 🟠 1 | 🟡 1)

[2/2] 审查 REQ-002: 订单创建流程
  📂 读取代码文件 (6 个)...
  🔍 验证数据一致性...
  ⚠️  发现 2 个问题 (🔴 1 | 🟠 1)

✅ 审查完成！共发现 5 个问题。
```

---

### 2.4 问题分类与优先级

**对发现的每个问题分类并评级**：

```yaml
严重等级:
  - [Critical] 严重: 可能导致数据丢失、安全漏洞、系统崩溃、需求完全无法满足
    示例: SQL注入、事务不一致、核心功能未实现
  
  - [High] 高: 影响需求实现，可能导致业务错误或功能不完整
    示例: 参数未验证、错误未处理、边界情况缺失
  
  - [Medium] 中: 需求基本实现，但有改进空间
    示例: 性能问题、代码重复、日志不足
  
  - [Low] 低: 代码风格或最佳实践问题，不影响需求实现
    示例: 命名不规范、注释缺失

问题分类:
  - ❌ 需求未实现 (Missing Feature)
  - 🐛 逻辑错误 (Logic Bug)
  - 🔒 安全风险 (Security)
  - 💾 数据一致性 (Data Consistency)
  - ⚡ 性能问题 (Performance)
  - 📝 代码质量 (Code Quality)
```

---

## Phase 3: 生成"需求 vs 实现"对照报告 🆕

分析完成后，在根目录生成一份名为 `REQUIREMENT_VALIDATION_REPORT.md` 的详细对照报告。

**核心理念**：以需求为维度，展示每个需求的实现情况和问题。

**报告模板：**

```markdown
# 📋 需求验证报告 (Requirement Validation Report)

**生成时间**: YYYY-MM-DD HH:MM  
**项目路径**: [项目路径]  
**项目类型**: [单项目 / Monorepo]  
**开发语言**: [语言和版本]  
**主要框架**: [框架名称和版本]  

---

## 📊 执行摘要

### 需求验证统计

| 指标 | 数量 | 占比 |
|------|------|------|
| 📋 总需求数 | [数量] | 100% |
| ✅ 完全实现 | [数量] | [百分比]% |
| ⚠️ 部分实现 | [数量] | [百分比]% |
| ❌ 未实现/实现错误 | [数量] | [百分比]% |

### 问题统计

| 严重程度 | 数量 | 说明 |
|---------|------|------|
| 🔴 Critical | [数量] | 可能导致需求完全无法满足 |
| 🟠 High | [数量] | 影响需求实现，功能不完整 |
| 🟡 Medium | [数量] | 需求基本实现，有改进空间 |
| 🟢 Low | [数量] | 代码风格问题，不影响需求 |

**总问题数**: [数量] 个

### 问题分类

| 类别 | 数量 |
|------|------|
| ❌ 需求未实现 | [数量] |
| 🐛 逻辑错误 | [数量] |
| 🔒 安全风险 | [数量] |
| 💾 数据一致性 | [数量] |
| ⚡ 性能问题 | [数量] |
| 📝 代码质量 | [数量] |

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

