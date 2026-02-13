# Trae 自动化工作流实现方案

## 一、现状对比分析

### 1.1 OpenWork 核心能力 vs Trae 现有能力

| 能力维度 | OpenWork | Trae 现状 | 差距 |
|----------|----------|-----------|------|
| **Skill 系统** | ✅ `.opencode/skills` | ✅ `.trae/skills` | 基本对等 |
| **Workflow 系统** | ✅ Commands | ✅ workflows/ (30+) | 基本对等 |
| **Agent 系统** | ✅ Agents | ✅ autonomous_agent.py | 基本对等 |
| **MCP 集成** | ✅ MCP Servers | ✅ FastMCP | 基本对等 |
| **权限管理** | ✅ 双层授权 | ❌ 无 | **需实现** |
| **实时事件流** | ✅ SSE | ❌ 无 | **需实现** |
| **Plan→Approve→Execute** | ✅ | ❌ 无 | **需实现** |
| **Artifact 管理** | ✅ 产出物展示 | ❌ 无 | **需实现** |
| **Audit Log** | ✅ 审计日志 | ❌ 无 | **需实现** |
| **双模式架构** | ✅ Host/Client | ❌ 单机 | **可选** |

### 1.2 核心差距

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenWork 工作流核心                       │
├─────────────────────────────────────────────────────────────┤
│  用户目标 → [生成计划] → [用户批准] → [执行] → [产出物]      │
│                ↑              ↑                             │
│           [权限请求]     [实时事件流]                        │
│                              ↓                              │
│                        [审计日志]                            │
└─────────────────────────────────────────────────────────────┘

Trae 缺失的关键环节：
1. 计划生成与审批
2. 权限请求与响应
3. 实时事件推送
4. 产出物管理
5. 审计日志
```

---

## 二、实现方案总览

### 2.1 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                     Trae 自动化工作流系统                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    用户交互层 (UI)                        │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │   │
│  │  │ 计划审批 │ │ 权限弹窗 │ │ 进度展示 │ │ 产出物   │    │   │
│  │  │ Panel   │ │ Dialog  │ │ Timeline │ │ Gallery  │    │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    工作流引擎层                           │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │   │
│  │  │PlanEngine│ │PermEngine│ │EventBus  │ │AuditLog  │    │   │
│  │  │ 计划引擎 │ │ 权限引擎 │ │ 事件总线 │ │ 审计日志 │    │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    执行层 (现有)                          │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │   │
│  │  │Workflow  │ │Autonomous│ │SkillMgr  │ │MCP Server│    │   │
│  │  │Manager V2│ │  Agent   │ │ 技能管理 │ │ 神经桥接 │    │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    存储层                                 │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │   │
│  │  │Plan Store│ │Perm Store│ │Audit DB  │                 │   │
│  │  │ 计划存储 │ │ 权限存储 │ │ 审计存储 │                 │   │
│  │  └──────────┘ └──────────┘ └──────────┘                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 模块清单

| 模块 | 文件路径 | 功能 |
|------|----------|------|
| 计划引擎 | `.trae/workflows/plan_engine.py` | 生成/编辑/存储执行计划 |
| 权限引擎 | `.trae/workflows/permission_engine.py` | 权限请求/审批/存储 |
| 事件总线 | `.trae/workflows/event_bus.py` | SSE 实时事件推送 |
| 审计日志 | `.trae/workflows/audit_logger.py` | 操作记录/导出 |
| 产出物管理 | `.trae/workflows/artifact_manager.py` | 产出物收集/展示 |
| 工作流编排器 | `.trae/workflows/orchestrator.py` | 统一入口，协调各引擎 |

---

## 三、详细设计

### 3.1 计划引擎 (PlanEngine)

#### 3.1.1 数据结构

```python
# .trae/workflows/plan_engine.py

from dataclasses import dataclass, field
from enum import Enum
from datetime import datetime
from typing import Optional
import uuid

class PlanStatus(Enum):
    DRAFT = "draft"           # 草稿
    PENDING = "pending"       # 待审批
    APPROVED = "approved"     # 已批准
    REJECTED = "rejected"     # 已拒绝
    EXECUTING = "executing"   # 执行中
    COMPLETED = "completed"   # 已完成
    FAILED = "failed"         # 失败

class StepStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"

@dataclass
class PlanStep:
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    name: str = ""
    description: str = ""
    action: str = ""              # run_command, file_write, etc.
    params: dict = field(default_factory=dict)
    status: StepStatus = StepStatus.PENDING
    permission_required: Optional[str] = None  # "file_write", "file_delete", etc.
    started_at: Optional[datetime] = None
    finished_at: Optional[datetime] = None
    output: Optional[str] = None
    error: Optional[str] = None

@dataclass
class ExecutionPlan:
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    goal: str = ""                # 用户目标
    steps: list[PlanStep] = field(default_factory=list)
    status: PlanStatus = PlanStatus.DRAFT
    created_at: datetime = field(default_factory=datetime.now)
    approved_at: Optional[datetime] = None
    approved_by: Optional[str] = None  # "user" or "auto"
    workspace: str = ""           # 工作目录
    artifacts: list[str] = field(default_factory=list)  # 产出物路径列表
```

#### 3.1.2 核心方法

```python
class PlanEngine:
    def __init__(self, workspace: str):
        self.workspace = workspace
        self.plans_dir = Path(workspace) / ".trae" / "plans"
        self.plans_dir.mkdir(parents=True, exist_ok=True)
    
    def generate_plan(self, goal: str, context: dict = None) -> ExecutionPlan:
        """
        根据用户目标生成执行计划
        1. 分析目标类型（文件操作/代码修改/部署等）
        2. 查找匹配的 Workflow 模板
        3. 生成步骤序列
        4. 标记需要权限的步骤
        """
        pass
    
    def edit_plan(self, plan_id: str, edits: dict) -> ExecutionPlan:
        """编辑计划（用户可修改步骤）"""
        pass
    
    def submit_for_approval(self, plan_id: str) -> ExecutionPlan:
        """提交审批"""
        pass
    
    def approve_plan(self, plan_id: str, user_decision: str) -> ExecutionPlan:
        """审批计划：approve / reject / modify"""
        pass
    
    def save_plan(self, plan: ExecutionPlan) -> str:
        """持久化计划到 .trae/plans/{plan_id}.json"""
        pass
    
    def load_plan(self, plan_id: str) -> ExecutionPlan:
        """加载计划"""
        pass
    
    def list_plans(self, status: PlanStatus = None) -> list[ExecutionPlan]:
        """列出计划"""
        pass
```

#### 3.1.3 计划生成策略

```python
class PlanGenerationStrategy:
    """计划生成策略"""
    
    def analyze_goal(self, goal: str) -> dict:
        """
        分析用户目标
        返回: {
            "type": "file_operation|code_change|deployment|research|...",
            "risk_level": "low|medium|high",
            "estimated_steps": 5,
            "required_permissions": ["file_write", "command_execute"]
        }
        """
        pass
    
    def find_matching_workflow(self, goal_analysis: dict) -> Optional[str]:
        """从 .trae/workflows/ 查找匹配的工作流模板"""
        pass
    
    def generate_steps_from_workflow(self, workflow_path: str, params: dict) -> list[PlanStep]:
        """从工作流模板生成步骤"""
        pass
    
    def generate_steps_from_scratch(self, goal: str, context: dict) -> list[PlanStep]:
        """无匹配模板时，基于 LLM 生成步骤"""
        pass
```

---

### 3.2 权限引擎 (PermissionEngine)

#### 3.2.1 数据结构

```python
# .trae/workflows/permission_engine.py

from dataclasses import dataclass, field
from enum import Enum
from datetime import datetime
from typing import Optional
import uuid

class PermissionType(Enum):
    FILE_READ = "file_read"
    FILE_WRITE = "file_write"
    FILE_DELETE = "file_delete"
    COMMAND_EXECUTE = "command_execute"
    NETWORK_ACCESS = "network_access"
    ENV_VAR_READ = "env_var_read"
    ENV_VAR_WRITE = "env_var_write"

class PermissionDecision(Enum):
    ALLOW_ONCE = "allow_once"       # 仅本次允许
    ALLOW_SESSION = "allow_session" # 本次会话允许
    ALLOW_ALWAYS = "allow_always"   # 始终允许
    DENY = "deny"                   # 拒绝

@dataclass
class PermissionRequest:
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    permission_type: PermissionType = PermissionType.FILE_WRITE
    resource: str = ""              # 文件路径/命令/URL
    reason: str = ""                # 请求原因
    context: dict = field(default_factory=dict)  # 相关上下文
    step_id: str = ""               # 关联的步骤ID
    plan_id: str = ""               # 关联的计划ID
    created_at: datetime = field(default_factory=datetime.now)
    decision: Optional[PermissionDecision] = None
    decided_at: Optional[datetime] = None

@dataclass
class PermissionRule:
    """持久化的权限规则"""
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    permission_type: PermissionType
    pattern: str                    # 资源匹配模式（支持通配符）
    decision: PermissionDecision
    created_at: datetime = field(default_factory=datetime.now)
    expires_at: Optional[datetime] = None
```

#### 3.2.2 核心方法

```python
class PermissionEngine:
    def __init__(self, workspace: str):
        self.workspace = workspace
        self.rules_file = Path(workspace) / ".trae" / "permission_rules.json"
        self.requests_dir = Path(workspace) / ".trae" / "permission_requests"
        self.authorized_roots: list[str] = []  # 用户授权的根目录
    
    def request_permission(
        self, 
        permission_type: PermissionType,
        resource: str,
        reason: str,
        context: dict = None
    ) -> PermissionRequest:
        """
        创建权限请求
        1. 检查是否已有匹配的持久规则
        2. 检查是否在授权根目录内
        3. 创建请求并等待用户响应
        """
        pass
    
    def check_existing_rule(self, permission_type: PermissionType, resource: str) -> Optional[PermissionDecision]:
        """检查是否已有匹配的权限规则"""
        pass
    
    def respond_permission(self, request_id: str, decision: PermissionDecision) -> PermissionRequest:
        """响应用户的权限决定"""
        pass
    
    def add_rule(self, rule: PermissionRule) -> None:
        """添加持久权限规则"""
        pass
    
    def remove_rule(self, rule_id: str) -> None:
        """移除权限规则"""
        pass
    
    def list_rules(self) -> list[PermissionRule]:
        """列出所有权限规则"""
        pass
    
    def set_authorized_roots(self, roots: list[str]) -> None:
        """设置用户授权的根目录"""
        pass
    
    def is_within_authorized_root(self, path: str) -> bool:
        """检查路径是否在授权根目录内"""
        pass
    
    def get_pending_requests(self) -> list[PermissionRequest]:
        """获取待处理的权限请求"""
        pass
```

#### 3.2.3 权限检查流程

```
操作请求
    ↓
检查是否在授权根目录内
    ├── 是 → 检查是否有持久规则
    │         ├── 有 → 自动批准/拒绝
    │         └── 无 → 创建权限请求 → 等待用户响应
    └── 否 → 直接创建权限请求 → 等待用户响应
```

---

### 3.3 事件总线 (EventBus)

#### 3.3.1 事件类型

```python
# .trae/workflows/event_bus.py

from dataclasses import dataclass, field
from enum import Enum
from datetime import datetime
from typing import Any, Callable
import uuid

class EventType(Enum):
    # 计划事件
    PLAN_CREATED = "plan_created"
    PLAN_UPDATED = "plan_updated"
    PLAN_APPROVED = "plan_approved"
    PLAN_REJECTED = "plan_rejected"
    
    # 步骤事件
    STEP_STARTED = "step_started"
    STEP_PROGRESS = "step_progress"
    STEP_COMPLETED = "step_completed"
    STEP_FAILED = "step_failed"
    
    # 权限事件
    PERMISSION_REQUESTED = "permission_requested"
    PERMISSION_GRANTED = "permission_granted"
    PERMISSION_DENIED = "permission_denied"
    
    # 产出物事件
    ARTIFACT_CREATED = "artifact_created"
    
    # 系统事件
    ERROR = "error"
    LOG = "log"

@dataclass
class Event:
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    type: EventType
    payload: dict = field(default_factory=dict)
    timestamp: datetime = field(default_factory=datetime.now)
    plan_id: Optional[str] = None
    step_id: Optional[str] = None
```

#### 3.3.2 核心方法

```python
from queue import Queue
from threading import Thread
import json

class EventBus:
    def __init__(self):
        self.subscribers: dict[EventType, list[Callable]] = {}
        self.event_queue: Queue[Event] = Queue()
        self.event_history: list[Event] = []
        self.max_history = 1000
    
    def subscribe(self, event_type: EventType, callback: Callable[[Event], None]):
        """订阅事件"""
        pass
    
    def unsubscribe(self, event_type: EventType, callback: Callable):
        """取消订阅"""
        pass
    
    def publish(self, event: Event):
        """发布事件"""
        pass
    
    def get_events_since(self, timestamp: datetime) -> list[Event]:
        """获取指定时间后的事件"""
        pass
    
    def get_events_by_plan(self, plan_id: str) -> list[Event]:
        """获取计划相关的所有事件"""
        pass
    
    def export_sse_stream(self) -> str:
        """导出为 SSE 格式流"""
        # data: {"type": "step_started", "payload": {...}}
        pass
```

#### 3.3.3 SSE 接口设计

```python
# 集成到 neuro_mcp_server.py

@app.tool()
def subscribe_events(last_event_id: str = None) -> str:
    """
    订阅实时事件流
    返回 SSE 格式的事件流
    """
    pass

@app.tool()
def get_event_history(plan_id: str = None, limit: int = 100) -> list[dict]:
    """获取事件历史"""
    pass
```

---

### 3.4 审计日志 (AuditLogger)

#### 3.4.1 数据结构

```python
# .trae/workflows/audit_logger.py

from dataclasses import dataclass, field
from enum import Enum
from datetime import datetime
from typing import Optional
import uuid

class AuditAction(Enum):
    # 计划操作
    PLAN_CREATED = "plan_created"
    PLAN_EDITED = "plan_edited"
    PLAN_APPROVED = "plan_approved"
    PLAN_REJECTED = "plan_rejected"
    PLAN_EXECUTED = "plan_executed"
    
    # 步骤操作
    STEP_STARTED = "step_started"
    STEP_COMPLETED = "step_completed"
    STEP_FAILED = "step_failed"
    
    # 权限操作
    PERMISSION_REQUESTED = "permission_requested"
    PERMISSION_GRANTED = "permission_granted"
    PERMISSION_DENIED = "permission_denied"
    RULE_ADDED = "rule_added"
    RULE_REMOVED = "rule_removed"
    
    # 文件操作
    FILE_READ = "file_read"
    FILE_WRITTEN = "file_written"
    FILE_DELETED = "file_deleted"
    
    # 命令操作
    COMMAND_EXECUTED = "command_executed"

@dataclass
class AuditEntry:
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    timestamp: datetime = field(default_factory=datetime.now)
    action: AuditAction
    actor: str = "agent"           # agent / user / system
    resource: str = ""             # 操作的资源
    details: dict = field(default_factory=dict)
    plan_id: Optional[str] = None
    step_id: Optional[str] = None
    session_id: Optional[str] = None
    result: str = "success"        # success / failure / denied
    error_message: Optional[str] = None
```

#### 3.4.2 核心方法

```python
import json
from pathlib import Path

class AuditLogger:
    def __init__(self, workspace: str):
        self.workspace = workspace
        self.log_file = Path(workspace) / ".trae" / "audit.log"
        self.log_file.parent.mkdir(parents=True, exist_ok=True)
    
    def log(self, entry: AuditEntry):
        """记录审计条目"""
        pass
    
    def log_plan_action(self, action: AuditAction, plan: ExecutionPlan, details: dict = None):
        """记录计划操作"""
        pass
    
    def log_permission_action(self, action: AuditAction, request: PermissionRequest):
        """记录权限操作"""
        pass
    
    def log_step_action(self, action: AuditAction, step: PlanStep, plan_id: str):
        """记录步骤操作"""
        pass
    
    def query(
        self,
        start_time: datetime = None,
        end_time: datetime = None,
        action: AuditAction = None,
        plan_id: str = None,
        actor: str = None
    ) -> list[AuditEntry]:
        """查询审计日志"""
        pass
    
    def export(self, format: str = "json") -> str:
        """导出审计日志 (json / csv / markdown)"""
        pass
    
    def get_summary(self, plan_id: str = None) -> dict:
        """获取审计摘要"""
        pass
```

---

### 3.5 产出物管理 (ArtifactManager)

#### 3.5.1 数据结构

```python
# .trae/workflows/artifact_manager.py

from dataclasses import dataclass, field
from enum import Enum
from datetime import datetime
from typing import Optional
import uuid

class ArtifactType(Enum):
    FILE = "file"
    DOCUMENT = "document"
    IMAGE = "image"
    CODE = "code"
    REPORT = "report"
    LOG = "log"
    ARCHIVE = "archive"

@dataclass
class Artifact:
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    name: str = ""
    type: ArtifactType = ArtifactType.FILE
    path: str = ""                 # 文件路径
    size: int = 0                  # 字节数
    mime_type: str = ""
    created_at: datetime = field(default_factory=datetime.now)
    plan_id: str = ""
    step_id: str = ""
    description: str = ""
    preview: Optional[str] = None  # 预览内容（文本文件）
    metadata: dict = field(default_factory=dict)
```

#### 3.5.2 核心方法

```python
class ArtifactManager:
    def __init__(self, workspace: str):
        self.workspace = workspace
        self.artifacts_dir = Path(workspace) / ".trae" / "artifacts"
        self.registry_file = Path(workspace) / ".trae" / "artifacts.json"
    
    def register_artifact(
        self,
        path: str,
        plan_id: str,
        step_id: str = None,
        description: str = ""
    ) -> Artifact:
        """注册产出物"""
        pass
    
    def get_artifact(self, artifact_id: str) -> Optional[Artifact]:
        """获取产出物"""
        pass
    
    def list_artifacts(self, plan_id: str = None) -> list[Artifact]:
        """列出产出物"""
        pass
    
    def generate_preview(self, artifact: Artifact) -> str:
        """生成预览"""
        pass
    
    def export_artifact(self, artifact_id: str, output_path: str) -> str:
        """导出产出物"""
        pass
    
    def cleanup_old_artifacts(self, days: int = 30):
        """清理旧产出物"""
        pass
```

---

### 3.6 工作流编排器 (Orchestrator)

#### 3.6.1 统一入口

```python
# .trae/workflows/orchestrator.py

from dataclasses import dataclass
from typing import Optional

@dataclass
class ExecutionResult:
    success: bool
    plan_id: str
    status: str
    artifacts: list[str]
    audit_summary: dict
    error: Optional[str] = None

class WorkflowOrchestrator:
    """
    统一工作流编排器
    协调 PlanEngine, PermissionEngine, EventBus, AuditLogger, ArtifactManager
    """
    
    def __init__(self, workspace: str):
        self.workspace = workspace
        self.plan_engine = PlanEngine(workspace)
        self.permission_engine = PermissionEngine(workspace)
        self.event_bus = EventBus()
        self.audit_logger = AuditLogger(workspace)
        self.artifact_manager = ArtifactManager(workspace)
        self.workflow_manager = WorkflowManagerV2(workspace)
    
    def create_plan(self, goal: str, context: dict = None) -> ExecutionPlan:
        """
        创建执行计划
        1. 调用 PlanEngine.generate_plan()
        2. 发布 PLAN_CREATED 事件
        3. 记录审计日志
        """
        pass
    
    def submit_plan(self, plan_id: str) -> ExecutionPlan:
        """提交计划审批"""
        pass
    
    def approve_plan(self, plan_id: str, decision: str = "approve") -> ExecutionPlan:
        """审批计划"""
        pass
    
    def execute_plan(self, plan_id: str) -> ExecutionResult:
        """
        执行计划
        1. 检查计划状态是否为 APPROVED
        2. 按步骤执行
        3. 遇到权限需求时暂停，请求权限
        4. 收集产出物
        5. 记录审计日志
        6. 发布事件
        """
        pass
    
    def execute_step(self, plan_id: str, step_id: str) -> bool:
        """执行单个步骤"""
        pass
    
    def abort_plan(self, plan_id: str, reason: str = "") -> ExecutionPlan:
        """中止计划"""
        pass
    
    def get_plan_status(self, plan_id: str) -> dict:
        """获取计划状态"""
        pass
    
    def get_execution_timeline(self, plan_id: str) -> list[dict]:
        """获取执行时间线"""
        pass
```

#### 3.6.2 执行流程

```
execute_plan(plan_id)
    │
    ├─→ 检查计划状态 (APPROVED?)
    │       └─→ 否 → 抛出异常
    │
    ├─→ 更新状态为 EXECUTING
    │
    ├─→ 遍历步骤
    │   │
    │   ├─→ 检查步骤权限需求
    │   │   └─→ 有需求 → permission_engine.request_permission()
    │   │       └─→ 等待用户响应
    │   │           ├─→ DENY → 跳过步骤或中止计划
    │   │           └─→ ALLOW → 继续
    │   │
    │   ├─→ 发布 STEP_STARTED 事件
    │   │
    │   ├─→ 执行步骤动作
    │   │   └─→ workflow_manager.execute_step()
    │   │
    │   ├─→ 收集产出物
    │   │   └─→ artifact_manager.register_artifact()
    │   │
    │   ├─→ 发布 STEP_COMPLETED/STEP_FAILED 事件
    │   │
    │   └─→ 记录审计日志
    │
    ├─→ 更新状态为 COMPLETED/FAILED
    │
    └─→ 返回 ExecutionResult
```

---

## 四、MCP 工具集成

### 4.1 新增 MCP 工具

```python
# 添加到 src/neuro_mcp_server.py

# ==================== 计划管理 ====================

@app.tool()
def create_execution_plan(goal: str, context: dict = None) -> dict:
    """
    创建执行计划
    
    Args:
        goal: 用户目标描述
        context: 额外上下文（可选）
    
    Returns:
        计划详情，包含步骤列表
    """
    orchestrator = WorkflowOrchestrator(os.getcwd())
    plan = orchestrator.create_plan(goal, context)
    return plan.to_dict()

@app.tool()
def get_plan(plan_id: str) -> dict:
    """获取计划详情"""
    pass

@app.tool()
def approve_plan(plan_id: str, decision: str = "approve") -> dict:
    """
    审批计划
    
    Args:
        plan_id: 计划ID
        decision: approve / reject
    """
    pass

@app.tool()
def execute_plan(plan_id: str) -> dict:
    """执行计划"""
    pass

@app.tool()
def abort_plan(plan_id: str, reason: str = "") -> dict:
    """中止计划"""
    pass

# ==================== 权限管理 ====================

@app.tool()
def get_pending_permissions() -> list[dict]:
    """获取待处理的权限请求"""
    pass

@app.tool()
def respond_permission(request_id: str, decision: str) -> dict:
    """
    响应权限请求
    
    Args:
        request_id: 请求ID
        decision: allow_once / allow_session / allow_always / deny
    """
    pass

@app.tool()
def list_permission_rules() -> list[dict]:
    """列出权限规则"""
    pass

@app.tool()
def add_permission_rule(
    permission_type: str,
    pattern: str,
    decision: str
) -> dict:
    """添加权限规则"""
    pass

@app.tool()
def remove_permission_rule(rule_id: str) -> bool:
    """移除权限规则"""
    pass

@app.tool()
def set_authorized_folders(folders: list[str]) -> dict:
    """设置授权文件夹"""
    pass

# ==================== 事件订阅 ====================

@app.tool()
def subscribe_events(event_types: list[str] = None) -> str:
    """
    订阅事件流
    
    Args:
        event_types: 订阅的事件类型列表（可选，默认全部）
    
    Returns:
        SSE 格式的事件流
    """
    pass

@app.tool()
def get_event_history(plan_id: str = None, limit: int = 100) -> list[dict]:
    """获取事件历史"""
    pass

# ==================== 审计日志 ====================

@app.tool()
def query_audit_log(
    start_time: str = None,
    end_time: str = None,
    action: str = None,
    plan_id: str = None
) -> list[dict]:
    """查询审计日志"""
    pass

@app.tool()
def export_audit_log(format: str = "json") -> str:
    """导出审计日志"""
    pass

# ==================== 产出物管理 ====================

@app.tool()
def list_artifacts(plan_id: str = None) -> list[dict]:
    """列出产出物"""
    pass

@app.tool()
def get_artifact(artifact_id: str) -> dict:
    """获取产出物详情"""
    pass

@app.tool()
def preview_artifact(artifact_id: str) -> str:
    """预览产出物"""
    pass
```

---

## 五、Skill 集成

### 5.1 自动化工作流 Skill

```markdown
# .trae/skills/automation-workflow/SKILL.md

---
name: automation-workflow
description: 自动化工作流系统，支持计划生成、权限审批、实时事件、审计日志
triggers:
  - "自动化工作流"
  - "执行计划"
  - "审批计划"
  - "权限管理"
  - "审计日志"
---

# 自动化工作流 Skill

## 功能

1. **计划管理** - 创建、编辑、审批、执行计划
2. **权限管理** - 权限请求、审批、规则管理
3. **实时事件** - SSE 事件流订阅
4. **审计日志** - 操作记录、导出
5. **产出物管理** - 产出物收集、展示

## 使用方式

### 创建计划
```
创建执行计划：帮我重构用户认证模块
```

### 审批计划
```
审批计划 plan_abc123：approve
```

### 执行计划
```
执行计划 plan_abc123
```

### 权限管理
```
列出权限规则
添加权限规则：file_write, *.md, allow_always
```

### 审计日志
```
查询审计日志：2026-02-01 到 2026-02-12
导出审计日志：json
```
```

### 5.2 Skill 实现代码

```python
# .trae/skills/automation-workflow/skill.py

from pathlib import Path
import sys
sys.path.insert(0, str(Path(__file__).parent.parent.parent / "workflows"))

from orchestrator import WorkflowOrchestrator

def handle_request(request: str, context: dict = None) -> str:
    """处理 Skill 请求"""
    orchestrator = WorkflowOrchestrator(context.get("workspace", os.getcwd()))
    
    # 解析请求类型
    if request.startswith("创建执行计划"):
        goal = request.replace("创建执行计划：", "").strip()
        plan = orchestrator.create_plan(goal, context)
        return format_plan_response(plan)
    
    elif request.startswith("审批计划"):
        parts = request.replace("审批计划", "").strip().split("：")
        plan_id = parts[0].strip()
        decision = parts[1].strip() if len(parts) > 1 else "approve"
        plan = orchestrator.approve_plan(plan_id, decision)
        return format_plan_response(plan)
    
    elif request.startswith("执行计划"):
        plan_id = request.replace("执行计划", "").strip()
        result = orchestrator.execute_plan(plan_id)
        return format_result_response(result)
    
    # ... 其他请求处理
```

---

## 六、配置文件

### 6.1 工作流配置

```yaml
# .trae/workflows/automation-config.yaml

name: automation-config
version: "1.0.0"
description: 自动化工作流系统配置

# 权限配置
permissions:
  # 默认授权的文件夹
  authorized_roots:
    - "{{workspace}}"
    - "{{workspace}}/src"
    - "{{workspace}}/tests"
  
  # 高风险操作需要显式审批
  high_risk_operations:
    - file_delete
    - command_execute
    - env_var_write
  
  # 自动批准规则
  auto_approve:
    - permission_type: file_read
      pattern: "{{workspace}}/**"
    - permission_type: file_write
      pattern: "{{workspace}}/.trae/**"

# 计划配置
plan:
  # 最大步骤数
  max_steps: 50
  
  # 单步骤超时（秒）
  step_timeout: 300
  
  # 是否允许用户编辑计划
  allow_edit: true
  
  # 是否需要审批
  require_approval: true

# 审计配置
audit:
  # 日志保留天数
  retention_days: 90
  
  # 是否记录详细日志
  verbose: true
  
  # 导出格式
  export_formats:
    - json
    - csv
    - markdown

# 事件配置
events:
  # 最大历史记录数
  max_history: 1000
  
  # SSE 心跳间隔（秒）
  heartbeat_interval: 30
```

### 6.2 项目规则更新

```markdown
# .trae/rules/project_rules.md 追加内容

## 自动化工作流规则

### 计划执行规则

1. **必须生成计划** - 执行任何多步骤操作前，必须先生成计划
2. **必须审批** - 计划必须经用户审批后才能执行
3. **权限检查** - 涉及文件写入/删除/命令执行时，必须请求权限
4. **审计记录** - 所有操作必须记录到审计日志

### 权限规则

1. **默认拒绝** - 未明确授权的操作默认拒绝
2. **最小权限** - 只请求完成任务所需的最小权限
3. **透明展示** - 权限请求必须展示操作内容和原因
4. **可撤销** - 所有授权规则都可以撤销

### 事件规则

1. **实时推送** - 步骤状态变化必须实时推送
2. **历史可查** - 事件历史必须可查询
3. **关联追溯** - 事件必须关联到计划和步骤
```

---

## 七、实施计划

### 7.1 阶段划分

| 阶段 | 内容 | 预计工作量 |
|------|------|-----------|
| **Phase 1** | 计划引擎 + 权限引擎 | 2-3 天 |
| **Phase 2** | 事件总线 + 审计日志 | 1-2 天 |
| **Phase 3** | 产出物管理 + 编排器 | 1-2 天 |
| **Phase 4** | MCP 工具集成 | 1 天 |
| **Phase 5** | Skill 封装 + 测试 | 1-2 天 |
| **Phase 6** | 文档 + 示例 | 1 天 |

### 7.2 文件清单

```
.trae/
├── workflows/
│   ├── plan_engine.py           # 新增
│   ├── permission_engine.py     # 新增
│   ├── event_bus.py             # 新增
│   ├── audit_logger.py          # 新增
│   ├── artifact_manager.py      # 新增
│   ├── orchestrator.py          # 新增
│   └── automation-config.yaml   # 新增
├── skills/
│   └── automation-workflow/
│       ├── SKILL.md             # 新增
│       └── skill.py             # 新增
├── plans/                       # 新增目录
├── permission_requests/         # 新增目录
├── artifacts/                   # 新增目录
└── audit.log                    # 新增文件

src/
└── neuro_mcp_server.py          # 修改：添加新工具
```

### 7.3 依赖关系

```
orchestrator.py
    ├── plan_engine.py
    ├── permission_engine.py
    ├── event_bus.py
    ├── audit_logger.py
    ├── artifact_manager.py
    └── workflow_manager_v2.py (现有)
```

---

## 八、使用示例

### 8.1 完整工作流示例

```python
# 用户：帮我重构用户认证模块，把 JWT 改成 OAuth2

# Step 1: 创建计划
plan = orchestrator.create_plan(
    goal="重构用户认证模块，把 JWT 改成 OAuth2",
    context={"workspace": "/path/to/project"}
)

# 返回计划：
# Plan ID: plan_abc123
# Steps:
#   1. 分析现有 JWT 认证代码
#   2. 设计 OAuth2 认证流程
#   3. 创建 OAuth2 配置文件
#   4. 重写认证中间件
#   5. 更新测试用例
#   6. 更新文档

# Step 2: 用户审批
plan = orchestrator.approve_plan("plan_abc123", "approve")

# Step 3: 执行计划
result = orchestrator.execute_plan("plan_abc123")

# 执行过程中：
# - 步骤 3 需要写入文件 → 权限请求 → 用户批准
# - 步骤 4 需要修改代码 → 权限请求 → 用户批准
# - 实时推送步骤进度

# Step 4: 查看结果
print(result)
# ExecutionResult:
#   success: true
#   artifacts:
#     - src/auth/oauth2.py
#     - config/oauth2.yaml
#     - tests/test_oauth2.py
#   audit_summary:
#     total_steps: 6
#     permissions_requested: 2
#     permissions_granted: 2
#     files_modified: 5
```

### 8.2 权限管理示例

```python
# 设置授权文件夹
permission_engine.set_authorized_roots([
    "/path/to/project/src",
    "/path/to/project/tests"
])

# 添加权限规则
permission_engine.add_rule(PermissionRule(
    permission_type=PermissionType.FILE_WRITE,
    pattern="**/*.py",
    decision=PermissionDecision.ALLOW_SESSION
))

# 处理权限请求
request = permission_engine.request_permission(
    permission_type=PermissionType.FILE_DELETE,
    resource="/path/to/project/src/old_auth.py",
    reason="清理旧的认证文件"
)
# → 用户收到权限请求弹窗
# → 用户选择 DENY
# → 步骤被跳过
```

---

## 九、风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 权限请求过多影响体验 | 用户疲劳 | 智能合并相似请求；提供"全部批准"选项 |
| 计划生成不准确 | 执行失败 | 支持用户编辑计划；失败自动重试 |
| 事件流丢失 | 状态不同步 | 事件持久化；支持重新订阅 |
| 审计日志膨胀 | 存储压力 | 自动清理；压缩存储 |

---

## 十、总结

本方案在 Trae 现有架构基础上，新增 6 个核心模块：

1. **PlanEngine** - 计划生成与审批
2. **PermissionEngine** - 权限请求与响应
3. **EventBus** - 实时事件推送
4. **AuditLogger** - 审计日志记录
5. **ArtifactManager** - 产出物管理
6. **WorkflowOrchestrator** - 统一编排

通过这些模块，Trae 将具备与 OpenWork 类似的自动化工作流能力：

- ✅ 计划生成 → 用户审批 → 执行 → 产出物
- ✅ 双层权限授权
- ✅ 实时事件流
- ✅ 完整审计日志
- ✅ 产出物管理

预计总工作量：**7-10 天**
