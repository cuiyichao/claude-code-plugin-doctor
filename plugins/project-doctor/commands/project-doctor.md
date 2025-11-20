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

## 📋 1. 需求验证详情

**以需求为核心，逐个展示验证结果。每个需求包含：需求信息、实现情况、问题列表、修复建议。**

---

### 需求 #1: [需求标题]

#### 📌 需求信息

| 项目 | 内容 |
|------|------|
| **需求 ID** | REQ-001 |
| **需求标题** | [用户提供的需求标题] |
| **需求描述** | [用户提供的需求描述] |
| **需求类型** | 功能需求 / 性能需求 / 安全需求 / 数据一致性 |
| **预期行为** | 1. [步骤1]<br>2. [步骤2]<br>3. [步骤3]... |

#### 🔍 实现情况

**实现状态**: ✅ **完全实现** / ⚠️ **部分实现** / ❌ **未实现/实现错误** ([ ]/100 分)

**代码定位**:
- 📍 入口: `[文件路径]:[函数名]()`
- 🔗 调用链:
  ```
  EntryPoint()
    → Service.method1()
    → Service.method2()
    → Repository.method()
  ```
- 📂 涉及文件: [数量] 个
  - `[文件路径1]`
  - `[文件路径2]`
  - ...

#### ✅ 已实现的功能

| 需求项 | 实现情况 | 说明 |
|-------|---------|------|
| ✅ [需求项1] | 已实现 | [简要说明] |
| ✅ [需求项2] | 已实现 | [简要说明] |
| ⚠️ [需求项3] | 部分实现 | [简要说明] |

#### ❌ 未实现/存在问题

**🔴 [Critical] [问题标题]**
- **位置**: `[文件名]:[行号]`
- **问题描述**:
  ```[language]
  // 当前代码
  [实际代码片段]
  ```
  [具体问题说明]
  
- **影响**: 
  - [影响点1]
  - [影响点2]
  
- **修复建议**:
  ```[language]
  // 建议代码
  [修复后的代码片段]
  ```
- **预计修复时间**: [小时数] 小时

**🟠 [High] [问题标题]**
- **位置**: `[文件名]:[行号]`
- **问题描述**: [简要说明]
- **影响**: [影响说明]
- **修复建议**: [修复建议]
- **预计修复时间**: [小时数] 小时

**🟡 [Medium] [问题标题]**
- **位置**: `[文件名]:[行号]`
- **问题描述**: [简要说明]
- **影响**: [影响说明]
- **修复建议**: [修复建议]
- **预计修复时间**: [小时数] 小时

#### 📊 需求实现评估

| 评估维度 | 得分 | 说明 |
|---------|------|------|
| 功能完整性 | [  ]/100 | [说明] |
| 数据一致性 | [  ]/100 | [说明] |
| 异常处理 | [  ]/100 | [说明] |
| 安全性 | [  ]/100 | [说明] |
| **综合得分** | **[  ]/100** | **[总体评价]** |

#### 🛠️ 修复计划

| 优先级 | 问题 | 预计时间 | 建议执行时间 |
|--------|------|---------|-------------|
| P0 | [问题描述] | [小时数] 小时 | 立即（今天） |
| P1 | [问题描述] | [小时数] 小时 | 今日内 |
| P2 | [问题描述] | [小时数] 小时 | 本周内 |

**总计修复时间**: 约 [小时数] 小时

**⚠️ 风险警告**: [如有Critical问题，说明风险]

---

[为每个需求重复上述结构]

---

## 📈 2. 总体评估

### 2.1 需求实现概览

| 需求 ID | 需求标题 | 实现状态 | 得分 | 问题数 |
|---------|---------|---------|------|--------|
| REQ-001 | [标题] | ⚠️ 部分实现 | 70/100 | 3 |
| REQ-002 | [标题] | ❌ 实现错误 | 45/100 | 3 |
| ... | ... | ... | ... | ... |

**平均实现得分**: [分数]/100

### 2.2 关键发现

✅ **做得好的地方**:
- [优点1]
- [优点2]
- [优点3]

⚠️ **需要立即修复**:
- **[REQ-XXX]**: [Critical问题摘要]
- **[REQ-YYY]**: [Critical问题摘要]

💡 **改进建议**:
- [建议1]
- [建议2]
- [建议3]

### 2.3 修复优先级总览

| 优先级 | 问题数 | 预计总耗时 | 建议完成时间 |
|--------|--------|-----------|-------------|
| **P0 (Critical)** | [数量] 个 | [小时数] 小时 | **立即（1-2天内）** |
| **P1 (High)** | [数量] 个 | [小时数] 小时 | 本周内 |
| **P2 (Medium)** | [数量] 个 | [小时数] 小时 | 2周内 |
| **P3 (Low)** | [数量] 个 | [小时数] 小时 | 1个月内 |

**总计**: [总小时数] 小时（约 [天数] 个工作日）

---

## 🎯 3. 行动计划

### 3.1 本周行动清单

**立即修复（P0）**:
- [ ] [REQ-001] [问题描述] - [预计时间]
- [ ] [REQ-002] [问题描述] - [预计时间]

**高优先级（P1）**:
- [ ] [REQ-XXX] [问题描述] - [预计时间]
- [ ] [REQ-YYY] [问题描述] - [预计时间]

### 3.2 后续跟进

**中优先级（P2）** - 2周内:
- [ ] [问题描述]
- [ ] [问题描述]

**低优先级（P3）** - 1个月内:
- [ ] [问题描述]
- [ ] [问题描述]

### 3.3 预防措施

为防止类似问题再次出现，建议：

1. **代码审查Checklist**: 建立需求验证清单
2. **单元测试**: 为Critical问题相关代码添加测试
3. **集成测试**: 覆盖关键业务流程
4. **监控告警**: 对数据一致性问题添加监控

---

## 📝 4. 附录

### 4.1 项目技术栈

| 类型 | 技术 | 版本 |
|------|------|------|
| 开发语言 | [语言] | [版本] |
| 主要框架 | [框架] | [版本] |
| 数据存储 | [数据库] | [版本] |
| 缓存 | [缓存] | [版本] |

### 4.2 诊断统计

- **诊断时间**: YYYY-MM-DD HH:MM
- **需求数量**: [数量] 个
- **相关文件数**: [数量] 个
- **代码行数**: 约 [数量] 行
- **总问题数**: [数量] 个

---

**诊断完成时间**: YYYY-MM-DD HH:MM  
**预计修复周期**: [周数] 周  
**下次诊断建议**: [建议时间]

🤖 **Generated by Project Doctor v2.0** - 需求驱动的项目诊断工具
```

## 执行逻辑

### 完整流程概览

```
┌─────────────────────────────────────────┐
│  Phase 0: 收集测试需求                   │
│  - 用户输入需求点                        │
│  - 解析为结构化格式                      │
│  - 确认需求理解正确                      │
└─────────────┬───────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Phase 1: 快速项目分析与需求定位          │
│  - 快速识别项目类型                      │
│  - 定位关键目录结构                      │
│  - 基于关键词搜索相关代码                │
│  - 构建需求到代码的映射                  │
└─────────────┬───────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Phase 2: 按需求针对性审查               │
│  - 读取相关代码文件                      │
│  - 理解需求预期行为                      │
│  - 根据需求类型选择审查策略              │
│  - 验证实现是否符合需求                  │
│  - 识别潜在问题和风险                    │
└─────────────┬───────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Phase 3: 生成需求对照报告               │
│  - 以需求为维度组织报告                  │
│  - 详细展示每个需求的实现情况            │
│  - 提供可执行的修复计划                  │
└─────────────────────────────────────────┘
```

### Phase 0 详细流程

```
1. 显示需求收集引导
   ↓
2. 接收用户输入（多行文本）
   ↓
3. 解析需求（提取关键词、推测入口点）
   ↓
4. 展示解析结果并请求确认
   ↓
5. 根据用户反馈调整（如需要）
   ↓
6. 输出需求收集完成信息
```

### Phase 1 详细流程

```
1. 快速检测项目类型
   Read ./package.json / ./go.mod / ./pom.xml
   ↓
2. 识别目录结构
   Glob: ./src/controllers/, ./src/services/, etc.
   ↓
3. 对每个需求执行关键词搜索
   Grep "[keyword]" --path ./src/ -i
   ↓
4. 识别相关文件和调用链
   分析 import/require 语句
   ↓
5. 展示定位结果并请求确认
   显示每个需求的入口点和涉及文件
   ↓
6. 输出定位完成统计
```

### Phase 2 详细流程

```
对每个需求执行：
  ↓
1. 读取所有相关代码文件
   Read [file1], Read [file2], ...
   ↓
2. 理解需求的预期行为
   分析需求描述，列出预期步骤
   ↓
3. 选择审查策略
   功能需求 → 策略A
   性能需求 → 策略B
   安全需求 → 策略C
   数据一致性 → 策略D
   ↓
4. 执行审查清单
   逐项检查：参数验证、错误处理、边界情况等
   ↓
5. 识别并记录问题
   分类：Critical, High, Medium, Low
   ↓
6. 评估实现得分
   各维度打分 + 综合得分
   ↓
显示进度（如：[1/3] 审查 REQ-001...）
```

### Phase 3 详细流程

```
1. 收集所有需求的验证结果
   ↓
2. 生成执行摘要
   统计需求实现情况、问题分布
   ↓
3. 为每个需求生成详细章节
   需求信息 + 实现情况 + 问题列表 + 修复计划
   ↓
4. 生成总体评估
   关键发现、修复优先级总览
   ↓
5. 生成行动计划
   本周清单、后续跟进、预防措施
   ↓
6. 添加附录信息
   技术栈、诊断统计
   ↓
7. Write 报告到文件
   REQUIREMENT_VALIDATION_REPORT.md
   ↓
8. 输出完成信息并询问后续行动
   是否需要立即协助修复某个问题？
```

### 进度展示示例

```
🎯 Project Doctor v2.0 - 需求驱动诊断

📋 Phase 0: 收集测试需求
   ✅ 收集到 2 个需求点

🔍 Phase 1: 分析项目并定位代码
   [1/3] 检测项目类型: TypeScript + Next.js
   [2/3] 识别目录结构: 发现 4 个核心模块
   [3/3] 定位需求相关代码: 10 个文件
   ✅ 定位完成

🔬 Phase 2: 按需求针对性审查
   [1/2] 审查 REQ-001: 用户注册邮件验证
         📂 读取 4 个文件...
         🔍 功能完整性验证...
         ⚠️  发现 3 个问题 (🔴 1 | 🟠 1 | 🟡 1)
   [2/2] 审查 REQ-002: 订单创建流程
         📂 读取 6 个文件...
         🔍 数据一致性验证...
         ⚠️  发现 2 个问题 (🔴 1 | 🟠 1)
   ✅ 审查完成，共发现 5 个问题

📝 Phase 3: 生成需求对照报告
   ✅ 报告已生成: REQUIREMENT_VALIDATION_REPORT.md

---

🎉 诊断完成！

📊 诊断结果:
  - 需求数: 2 个
  - 完全实现: 0 个
  - 部分实现: 1 个
  - 实现错误: 1 个
  - 发现问题: 5 个 (🔴 2 | 🟠 2 | 🟡 1)

⚠️  发现 2 个 Critical 问题，建议立即修复！

📄 详细报告: ./REQUIREMENT_VALIDATION_REPORT.md
```

### 注意事项

1. **性能考虑**:
   - 只读取需求相关的文件，不全量扫描
   - 使用关键词精准定位，减少不必要的文件读取
   - 合理使用批量操作减少工具调用次数

2. **用户体验**:
   - 实时显示进度，让用户了解当前状态
   - 每个阶段都有确认环节，确保理解正确
   - 报告以需求为维度，易于理解和行动

3. **报告质量**:
   - 每个问题必须包含：位置、描述、影响、修复建议、预计时间
   - 代码示例必须准确，来自实际代码
   - 修复建议必须可执行，包含具体代码
   - 优先级评定基于需求影响，而非代码风格

4. **错误处理**:
   - 如果无法定位某个需求，询问用户提供更多信息
   - 如果代码读取失败，记录并继续其他需求
   - 始终生成报告，即使部分分析失败

