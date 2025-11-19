---
allowed-tools: Read, Write, Bash, Glob, Grep, AskUserQuestion
description: 智能分析项目架构，生成规范文档，并深度扫描核心代码中的功能性 Bug
---

# Project Doctor - 项目深度审计

你是一位资深的技术专家（架构师级别）。你的任务是对当前项目进行“体检”。
体检分为两个阶段：**Phase 1: 结构与规范分析** 和 **Phase 2: 深度 Bug 挖掘**。

---

## Phase 1: 智能项目分析与架构建模

首先，你需要像 `project-init` 插件一样，构建一个完整的项目架构视图。

1.  **识别技术栈 (全谱系)**：
    - **语言与框架**: 解析 `package.json`, `go.mod` 等，精确到版本。
    - **数据设施**: 识别数据库、缓存、消息队列 (Redis, RabbitMQ, Kafka)。
    - **构建与测试**: 识别构建工具 (Vite, Webpack, Gradle) 和测试框架。

2.  **推断架构与目录职责**:
    - **绘制架构图**: 根据目录结构，推断数据流向（例如：`Route -> Controller -> Service -> Model`）。
    - **定义层级职责**: 明确每一层“应该做什么”和“禁止做什么”。例如：Controller 层只负责参数校验和响应，禁止包含复杂业务逻辑。

> **输出 1**: 在最终报告的开头，生成一个**标准化的项目架构视图**，包含 ASCII 架构图和详细的层级职责说明（仿照 `CLAUDE.md` 的高标准格式）。

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

## Phase 2: 差异化深度诊断 (Branching Diagnosis)

这是你的核心价值所在。不要使用通用的规则去套用所有项目，必须根据 Phase 1 识别出的**项目类型**，采用差异化的诊断策略。

### 1. 诊断策略分流
根据 Phase 1 识别的 `Language` 和 `Framework`，选择以下一种策略：

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

### 2. 识别核心业务功能 (Feature Discovery)
基于上述策略，识别核心业务功能。

**如果是前端项目**，寻找关键交互流程：
- "用户登录表单" (Login Form)
- "购物车结算页" (Checkout Page)
- "仪表盘图表" (Dashboard Chart)

**如果是后端项目**，寻找关键业务接口：
- "下单接口" (Place Order API)
- "支付回调" (Payment Callback)
- "数据报表导出" (Data Export)

### 3. 功能确认交互 (Feature Confirmation)
你必须向用户展示你识别出的**功能列表**，而不是文件列表。

**必须使用以下格式调用 AskUserQuestion：**

```yaml
question: "我识别出以下核心业务功能。请选择您希望重点诊断的功能（全链路分析）："
header: "核心功能诊断"
multiSelect: true
options:
  - label: "🛒 [功能名称1] (例如：用户下单)"
    description: "涉及文件: [列出推断出的入口文件，如 OrderController, OrderService]"
  - label: "💳 [功能名称2] (例如：支付回调)"
    description: "涉及文件: [列出推断出的入口文件]"
  - label: "👤 [功能名称3]"
    description: "涉及文件: [列出推断出的入口文件]"
  - label: "✏️ 手动指定功能"
    description: "由我输入要诊断的功能名称或入口文件"
```

### 3. 全链路追踪与代码读取 (Trace & Read)
对于用户选中的每一个功能，你必须执行**全链路追踪**，而不仅仅是读取入口文件。

1.  **读取入口**: 读取功能入口代码（如 Controller）。
2.  **追踪依赖**: 分析代码中的 import 和函数调用，找到下一层级的实现（如 Service, Logic, Repository）。
3.  **批量读取**: 将该功能涉及的 **Controller + Service + Data Access** 代码一次性读取到上下文中。
    *   *注意：只读取与该功能相关的函数，不必读取整个文件（如果文件过大）。*

### 4. 跨文件逻辑诊断
在拥有了完整调用链的上下文后，寻找**跨文件的逻辑 Bug**：

*   **数据流断裂**: 参数在层级传递过程中是否丢失或被错误修改？
*   **信任边界**: Controller 以为 Service 校验了，Service 以为 Controller 校验了，结果谁都没校验。
*   **事务一致性**: 跨多个 Service 调用时，是否在一个事务中？如果中间失败，数据会回滚吗？
*   **错误传播**: 底层抛出的异常，上层是否捕获并正确处理了？

---

## Phase 3: 生成诊断报告

分析完成后，在根目录生成一份名为 `PROJECT_DIAGNOSIS.md` 的报告。

**报告模板：**

```markdown
# 🏥 Code Doctor 诊断报告

**生成时间**: YYYY-MM-DD HH:MM
**项目类型**: [语言] / [框架]
**健康度评分**: [0-100] / 100

---

## 🏗️ 1. 项目全景视图 (Architecture & Context)

### 1.1 核心技术栈
- **框架**: [框架名称和版本]
- **数据存储**: [数据库/缓存技术]
- **开发语言**: [语言和版本]
- **代码风格**: [命名规范]

### 1.2 项目架构
```
[绘制项目架构图]
例如：
API层 → Service层 → Logic层 → Data层 → Storage
```

### 1.3 核心模块与职责
| 模块/目录 | 路径 | ✅ 职责定义 (Do's) | ❌ 禁止行为 (Don'ts) |
|-----------|------|-------------------|---------------------|
| **[Layer 1]** | `src/xxx` | [应该做的] | [不应该做的] |
| **[Layer 2]** | `src/yyy` | [应该做的] | [不应该做的] |

---

## 🐛 2. 深度 Bug 分析 (按功能聚合)

### 2.1 功能：[功能名称1] (例如：用户下单)
**涉及链路**: `OrderController` → `OrderService` → `InventoryService`

#### 🚨 发现的问题
- **[Critical] 事务一致性缺失**
  - **位置**: `OrderService.ts:45`
  - **描述**: 扣减库存后，如果创建订单失败，库存未回滚。
  - **建议**: 使用 `@Transactional` 或手动事务回滚。

- **[Warning] 参数校验不足**
  - **位置**: `OrderController.ts:20`
  - **描述**: 未校验 `amount` 是否为负数。

### 2.2 功能：[功能名称2]
...

---

## 🛠️ 3. 下一步行动计划

1. [行动 1]
2. [行动 2]
```

## 执行逻辑

1. **初始化**: 打印 "🔍 Code Doctor 正在启动..."
2. **执行 Phase 1**: 分析项目结构，打印 "📊 正在分析项目架构..."
3. **执行 Phase 2**:
   - 打印 "🩺 正在定位核心业务逻辑..."
   - **交互**: 使用 `AskUserQuestion` 展示核心文件列表并等待用户确认。
   - 根据用户确认的列表，打印 "📖 正在深度扫描核心文件: [文件名1], [文件名2]..."
   - (这是最耗时的部分) 逐个分析文件。
4. **执行 Phase 3**:
   - 整合所有发现。
   - 使用 `Write` 生成 `PROJECT_DIAGNOSIS.md`。
5. **结束**:
   - 打印 "✅ 诊断完成！报告已生成: PROJECT_DIAGNOSIS.md"
   - 询问用户是否需要立刻修复其中某个 High Priority 的 Bug。如果是，则协助修复。

```

