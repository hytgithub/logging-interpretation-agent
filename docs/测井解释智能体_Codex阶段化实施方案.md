# 测井解释智能体 Codex 阶段化实施方案

> 目标：将《测井解释智能体技术架构方案》转换为 Codex 可以阶段性执行、阶段性验证、阶段性交付的开发任务。  
> 周期：2 周  
> 推荐方式：1 个主 Agent 负责架构把控与集成，多个子 Agent 按模块并行实现。

---

## 1. 为什么需要单独的 Codex 实施方案

技术架构文档主要回答：

> 系统应该怎么设计。

但 Codex 真正执行时还需要明确：

- 先做什么；
- 后做什么；
- 哪些任务可以并行；
- 哪些模块禁止跨边界修改；
- 每个阶段产出什么；
- 每个阶段如何验证；
- 什么情况下可以进入下一阶段。

因此不建议直接让 Codex 一次性读取技术方案后“完整实现”。

推荐：

```text
技术方案
   ↓
阶段拆分
   ↓
任务拆分
   ↓
子 Agent 执行
   ↓
主 Agent Review
   ↓
测试
   ↓
阶段验收
   ↓
进入下一阶段
```

---

## 2. Codex 总体执行原则

### 2.1 主 Agent 职责

主 Agent 只负责：

- 理解架构文档；
- 建立工程骨架；
- 定义核心接口；
- 拆分子任务；
- 控制模块边界；
- Review 子 Agent 输出；
- 解决模块冲突；
- 运行集成测试；
- 决定阶段是否完成。

主 Agent 不应包办所有代码。

---

### 2.2 子 Agent 职责

子 Agent 按模块执行：

```text
Agent Framework Agent
Tool Agent
Model Agent
Context/Redis Agent
Database Agent
Observability Agent
Testing Agent
Scenario Agent
```

每个子 Agent：

- 只修改授权目录；
- 不擅自调整全局架构；
- 不修改其他模块公共接口；
- 如需修改接口，先反馈主 Agent；
- 必须补测试。

---

### 2.3 接口先行

第一阶段必须优先确定以下接口：

```text
BaseAgent
Task
TaskPlan
TaskResult
TaskExecutor
Tool
ToolResult
ToolRegistry
ModelRegistry
ModelProfile
ContextStore
TaskRepository
TraceContext
```

接口没有稳定前，不允许大量并行开发具体实现。

---

## 3. 阶段总览

| 阶段 | 时间建议 | 核心目标 |
|---|---:|---|
| Phase 0 | 0.5 天 | 项目约束、环境、架构基线 |
| Phase 1 | 1 天 | 工程骨架与核心接口 |
| Phase 2 | 1.5 天 | PostgreSQL、Redis、配置、基础设施 |
| Phase 3 | 1.5 天 | Model Registry、AgentScope Adapter |
| Phase 4 | 2 天 | Agent、Planner、Task Orchestrator |
| Phase 5 | 1.5 天 | Tool Framework + Mock Tool |
| Phase 6 | 1 天 | OpenTelemetry + Langfuse |
| Phase 7 | 1.5 天 | 完整测井解释链路 |
| Phase 8 | 1 天 | 异常、状态恢复、横向扩展检查 |
| Phase 9 | 1 天 | 场景测试、Demo、收尾 |

总计约 12 天，预留约 2 天处理集成问题和 Prompt 调整。

---

# Phase 0：建立开发基线

## 目标

让所有 Codex 子 Agent 在相同约束下开发。

## 主 Agent 任务

创建：

```text
README.md
ARCHITECTURE.md
CONTRIBUTING.md
pyproject.toml
.env.example
docker-compose.yml
```

确定：

- Python 版本；
- uv；
- Ruff；
- pytest；
- 包命名；
- import 规则；
- 配置规则；
- 目录边界；
- 测试规则。

## 禁止事项

Phase 0 不实现业务 Agent。

## 完成标准

```text
uv sync
pytest
ruff check
```

能够正常运行。

---

# Phase 1：工程骨架和核心接口

## 目标

建立所有后续模块依赖的稳定协议。

## 主 Agent

负责创建接口。

### 核心类型

```text
Task
TaskPlan
TaskResult
AgentResult
ToolResult
ModelProfile
```

### 核心接口

```text
Agent
TaskExecutor
Tool
ToolRegistry
ContextStore
TaskRepository
ModelProvider
TraceProvider
```

## 子 Agent 可并行

### 子 Agent A：Agent Interface

目录：

```text
src/agents/base/
```

### 子 Agent B：Tool Interface

目录：

```text
src/tools/base/
```

### 子 Agent C：Repository / Context Interface

目录：

```text
src/repositories/interfaces/
src/context/interfaces.py
```

## 完成标准

- 类型可 import；
- 不存在循环依赖；
- 核心接口均有 docstring；
- 具备最基础单元测试。

## Checkpoint

主 Agent 必须 Review 所有接口。

接口确认后再进入 Phase 2。

---

# Phase 2：PostgreSQL、Redis 与基础设施

## 目标

保证 Agent Service 从一开始就具备共享状态和持久化能力。

## 子 Agent A：PostgreSQL

负责：

```text
SQLAlchemy
Alembic
Repository
```

首批实体：

```text
sessions
tasks
workflow_executions
agent_executions
tool_executions
interpretation_results
```

不要一次性设计过多表。

### 验收

- migration 可以执行；
- Repository CRUD 测试通过；
- PostgreSQL 重启后数据存在。

---

## 子 Agent B：Redis

负责：

```text
RedisContextStore
TaskStateStore
SessionContextStore
```

### Key 最小规范

```text
session:{session_id}
task:{task_id}:state
task:{task_id}:context
workflow:{workflow_id}:state
```

### 验收

- 任意应用实例能读取另一个实例写入的 Context；
- TTL 正常；
- JSON / Pydantic 序列化稳定。

---

## 子 Agent C：Configuration

负责：

```text
Pydantic Settings
.env
配置分层
```

## Phase 2 完成标准

```text
docker compose up
```

后可以使用：

- PostgreSQL；
- Redis；
- Agent Application。

---

# Phase 3：模型访问层与 AgentScope Adapter

## 目标

Agent 不绑定具体模型。

## 子 Agent A：Model Registry

实现：

```text
ModelProfile
ModelRegistry
Agent → Profile Mapping
```

支持：

```text
planning
reasoning
executor
validator
```

当前全部可以映射到同一模型。

---

## 子 Agent B：Model Adapter

基于 AgentScope Model API 做薄封装。

禁止重新实现完整 LLM SDK。

---

## 子 Agent C：AgentScope Adapter

实现 AgentScope 与业务 Agent Interface 之间的适配。

## 验收

做到：

```python
model_registry.get_for_agent("planner")
model_registry.get_for_agent("data_agent")
```

可以返回正确配置。

通过配置切换模型时 Agent 代码无需修改。

---

# Phase 4：Main Agent、Planner 与 Task Orchestrator

## 目标

完成 Agent 系统“大脑和骨架”。

## 子 Agent A：Main Agent

实现：

```text
理解请求
调用 Planner
提交 Plan
等待结果
汇总回答
```

---

## 子 Agent B：Planner

实现结构化 Plan。

要求：

- Task 有 ID；
- Task 有 type；
- Task 有 assigned_agent；
- Task 有 depends_on；
- Task 可标识 parallelizable。

Planner 输出必须经过 Pydantic 校验。

---

## 子 Agent C：Task Orchestrator

实现：

```text
读取 Plan
判断依赖
推进 Task State
调用 TaskExecutor
保存状态
获取 Result
执行后续 Task
```

本期只支持：

- 串行；
- 简单并行；
- 基础失败处理。

不实现复杂 BPMN / DAG Engine。

---

## 子 Agent D：LocalTaskExecutor

负责：

```text
Task
 ↓
Agent Registry
 ↓
对应 Agent
```

为未来 MQ Executor 预留接口。

---

## Phase 4 验收

输入：

```text
“解释 WELL-001 的目标井段”
```

Planner 能输出结构化 Plan。

Task Orchestrator 能完成一组 Dummy Agent 的调用。

此阶段不要求真实测井逻辑。

---

# Phase 5：Tool Framework 与 Mock Tool

## 目标

打通：

```text
Agent
→ Tool Selection
→ Tool Registry
→ Tool Execution
→ Tool Result
```

## 子 Agent A：Tool Registry

实现：

- register；
- get；
- list；
- allowed_agents；
- schema validation。

---

## 子 Agent B：Tool Executor

统一：

- 参数校验；
- timeout；
- exception handling；
- trace；
- Tool Result。

---

## 子 Agent C：Mock Well Tool

至少实现：

```text
get_well_data
get_curve_data
check_data_quality
get_formation_info
calculate_porosity
calculate_saturation
```

---

## 子 Agent D：Mock 数据

准备至少 3 组井场景：

### Scenario A

数据正常。

### Scenario B

部分曲线缺失。

### Scenario C

数据存在异常，需要 Agent 判断。

---

## Phase 5 验收

Agent 不允许直接读取 mocks JSON。

必须：

```text
Agent
 ↓
Tool
 ↓
Provider
 ↓
Mock Data
```

---

# Phase 6：日志、OpenTelemetry 与 Langfuse

## 目标

所有核心执行步骤可追踪。

## 子 Agent：Observability

负责：

```text
structlog
OpenTelemetry
Langfuse
```

至少建立：

```text
request span
main_agent span
planner span
task span
agent span
model span
tool span
```

## 验收

执行一个请求后可以在 Langfuse 中看到：

```text
Main Agent
Planner
Task Orchestrator
Sub Agent
Tool
LLM Call
```

调用链。

---

# Phase 7：测井解释完整链路

## 目标

第一次真正完成端到端测井解释任务。

## Agent 建议

```text
Data Agent
Logging Analysis Agent
Interpretation Agent
Validation Agent
```

## 推荐流程

```text
用户任务
   ↓
Main Agent
   ↓
Planner
   ↓
获取井数据
   ↓
数据质量检查
   ↓
测井曲线分析
   ↓
参数计算
   ↓
综合解释
   ↓
结果校验
   ↓
最终解释
```

## 并行建议

例如：

```text
GR Analysis
RT Analysis
DEN Analysis
```

可在同一层并行。

不要为了“多 Agent”强行拆过多 Agent。

---

## Phase 7 验收

必须至少稳定跑通：

### Case 1：正常井

得到完整解释。

### Case 2：数据缺失

Agent 能识别缺失，不胡乱继续。

### Case 3：多 Tool + 多 Agent

Planner 能拆解，多个 Agent 协作得到最终结果。

---

# Phase 8：异常处理与横向扩展检查

## 目标

验证架构不是只能单实例 Demo。

## 测试 1：Tool Error

模拟 Tool 报错。

要求：

- 不导致进程崩溃；
- Task State 可查询；
- Trace 可看到异常。

---

## 测试 2：Model Error

模拟模型失败。

要求：

- 有基础 Retry；
- 失败可追踪；
- 状态正确。

---

## 测试 3：进程状态隔离

启动两个 Agent Service 实例。

Instance A 写入：

```text
Session
Task State
Context
```

Instance B 能继续读取。

这是本期“支持横向扩展”的关键测试。

---

## 测试 4：Mock → Real 可替换检查

至少人工 Review：

```text
Tool
Repository
ContextStore
Model
TaskExecutor
```

是否均通过接口隔离。

---

# Phase 9：Scenario Test 与 Demo

## 目标

让系统可以演示，而不是只“代码完成”。

## Testing Agent

整理：

```text
tests/scenarios/
```

建议 10～20 个 Case。

覆盖：

```text
正常解释
数据缺失
异常数据
多 Tool
多 Agent
任务并行
Tool 失败
模型失败
Plan 不合理
结果校验
```

---

## Demo 必须展示

### 1. 用户自然语言输入

不是手工调用某个 Agent。

### 2. Planner 输出

展示结构化 Plan。

### 3. Agent 执行链

展示哪些 Agent 被调用。

### 4. Tool Call

展示 Tool 输入输出。

### 5. Trace

在 Langfuse 查看完整执行链。

### 6. 最终结果

输出完整解释结论。

---

# 4. Codex 多 Agent 并行建议

推荐：

```text
                   Codex Main Agent
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
Architecture       Infrastructure       Agent Core
Agent              Agent                Agent
       │                 │                 │
       ▼                 ▼                 ▼
接口/边界          PG/Redis/Config      Agent/Planner
                                            │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
Tool Agent         Observability       Test Agent
                   Agent
```

不要让所有子 Agent 同时改：

```text
src/common
src/config
pyproject.toml
docker-compose.yml
```

这些文件建议由主 Agent 控制。

---

# 5. 子 Agent 工作模板

给 Codex 子 Agent 的任务建议统一采用：

```text
目标：
负责的目录：
允许修改：
禁止修改：
依赖接口：
实现内容：
测试要求：
验收标准：
输出内容：
```

示例：

```text
目标：
实现 Redis Context Store。

负责目录：
src/context/
tests/unit/context/

允许修改：
上述目录。

禁止修改：
agents/
tools/
orchestration/

依赖接口：
ContextStore

实现：
RedisContextStore
SessionContext
TaskContext
TTL

测试：
pytest tests/unit/context/

验收：
1. set/get 正常
2. TTL 正常
3. 两实例共享状态
4. Redis 不可用时错误结构化
```

---

# 6. 主 Agent 每阶段 Review 清单

每个阶段结束必须检查：

```text
[ ] 是否破坏模块边界
[ ] 是否引入跨层依赖
[ ] 是否出现循环 import
[ ] 是否直接绑定具体基础设施
[ ] 是否有测试
[ ] 是否存在硬编码模型
[ ] 是否 Agent 直接读数据库
[ ] 是否 Agent 直接读 Mock 文件
[ ] 是否状态只存在单实例内存
[ ] 是否新增接口没有文档
[ ] 是否 Trace 覆盖关键路径
```

---

# 7. Git / 提交建议

如果 Codex 环境支持多分支或 Worktree，建议按模块隔离。

例如：

```text
feature/core-interfaces
feature/postgres
feature/redis
feature/model-registry
feature/agent-core
feature/task-orchestrator
feature/tool-framework
feature/observability
feature/demo-scenarios
```

合并顺序遵循阶段依赖。

不要同时大范围重构。

---

# 8. 两周最终 Definition of Done

两周结束时，只有同时满足以下条件才认为 MVP 架构完成。

## 架构

- AgentScope 集成完成；
- Main / Planner / Sub Agent 能运行；
- Task Orchestrator 能执行 Plan；
- Tool Registry 能动态扩展；
- Model Profile 能配置；
- PostgreSQL 持久化完成；
- Redis 共享 Runtime State；
- Agent Service 不依赖单机内存状态。

## Agent 能力

- 能理解自然语言任务；
- 能规划；
- 能调用多个 Agent；
- 能选择 Tool；
- 能理解 Tool Result；
- 能处理数据异常；
- 能生成最终解释结果。

## 工程

- Docker Compose 可启动；
- pytest 通过；
- 核心模块有单元测试；
- 至少有完整集成测试；
- OpenTelemetry / Langfuse 可追踪完整调用链。

## Demo

必须至少展示 3 个核心场景：

```text
正常测井解释
数据缺失 / 异常
复杂多 Agent + 多 Tool 任务
```

---

# 9. 本阶段明确不做

Codex 不允许在本期主动扩展以下内容，除非完成全部核心目标后仍有余量：

```text
Kubernetes
完整 MQ
复杂 RAG
生产级 Vector DB
复杂权限系统
OAuth
高可用集群
自动扩缩容
复杂性能压测
完整 CI/CD
复杂长期 Memory
智能模型动态路由
```

任何子 Agent 如认为必须引入上述能力，应先反馈主 Agent，不允许直接增加系统复杂度。

---

# 10. 推荐 Codex 执行入口

主 Agent 首次执行时建议使用下面的任务描述：

```text
请严格按照《测井解释智能体技术架构方案》和《测井解释智能体 Codex 阶段化实施方案》实施。

当前只执行当前 Phase，不允许提前大规模实现后续阶段。

工作原则：
1. 先检查当前仓库状态。
2. 阅读架构文档。
3. 输出当前 Phase 的实现计划。
4. 明确需要创建或修改的文件。
5. 能并行的任务分派给子 Agent。
6. 子 Agent 只能修改授权目录。
7. 主 Agent 负责公共接口和最终集成。
8. 每个任务必须补测试。
9. 每个 Phase 完成后运行测试和静态检查。
10. 输出阶段完成情况、未完成项、风险和下一阶段入口。
11. 未通过本阶段验收标准，不进入下一阶段。
12. 不得擅自引入本期明确不做的复杂技术。
```

之后每一阶段单独启动一次 Codex 任务，比一次要求“全部做完”更稳定、更容易 Review。
