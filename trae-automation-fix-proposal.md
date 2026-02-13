# Trae 自动化工作流优化方案

## 问题诊断

**现状**：Trae 已有 `autonomous-agent` 技能，但存在以下问题：

| 问题 | 原因 | 影响 |
|------|------|------|
| 执行中停下来问用户 | 遇到未处理的错误/选择点 | 打断自动化流程 |
| 完成后没验证就交付 | 验证规则不够严格 | 交付有问题的代码 |
| 验证失败后放弃 | `max_heal_attempts: 3` 太小 | 没修好就停止 |

**用户期望**：
- ✅ 执行过程不停止询问
- ✅ 完成后必须验证
- ✅ 验证失败自动修复，直到成功

---

## 解决方案

### 方案一：修改配置（最简单）

修改 `.trae/skills/autonomous-agent/skill.yaml`：

```yaml
# 修改前
auto_execute:
  on_trigger: true
  auto_heal: true
  max_heal_attempts: 3

# 修改后
auto_execute:
  on_trigger: true
  auto_heal: true
  max_heal_attempts: 999  # 改为极大值，几乎不停止
  never_stop: true        # 新增：永不停止标志
```

### 方案二：修改代码逻辑

修改 `.trae/workflows/autonomous_agent.py`：

```python
# 找到 execute_task 方法中的重试逻辑
# 原代码（约 796 行）：
if task.retry_count >= self.max_retries:
    task.status = TaskStatus.FAILED
    break

# 改为：
if task.retry_count >= self.max_retries:
    # 不直接失败，而是尝试其他修复策略
    if self.auto_mode and task.retry_count < 999:
        self._try_alternative_fix(task, step, result.error)
        continue
    else:
        task.status = TaskStatus.FAILED
        break
```

### 方案三：增强验证强制执行

修改 `.trae/workflows/smart_router.py`：

```python
# 在 process 方法末尾添加强制验证循环
def process(self, task: str, files: List[str]) -> dict:
    # ... 现有执行逻辑 ...
    
    # 强制验证循环
    max_attempts = 999
    for attempt in range(max_attempts):
        validation_result = self._validate(files)
        
        if validation_result["all_passed"]:
            return {"success": True, "validation": validation_result}
        
        # 验证失败，自动修复
        if self.auto_heal:
            fix_result = self._auto_fix(validation_result["failures"])
            if not fix_result["success"]:
                # 修复失败，尝试其他策略
                self._try_alternative_strategy(validation_result["failures"])
        else:
            break
    
    return {"success": False, "validation": validation_result}
```

---

## 具体修改清单

### 1. 修改 skill.yaml（1 处）

```yaml
# .trae/skills/autonomous-agent/skill.yaml
# 第 145-151 行

auto_execute:
  on_trigger: true
  auto_heal: true
  max_heal_attempts: 999  # 改为 999
  never_stop: true        # 新增
  silent_mode: true       # 新增：静默模式，不询问用户
```

### 2. 修改 autonomous_agent.py（2 处）

```python
# .trae/workflows/autonomous_agent.py

# 修改点 1：类初始化（约 50 行）
class AutonomousAgent:
    def __init__(self, ...):
        # ...
        self.auto_mode = True      # 新增：默认开启自动模式
        self.silent_mode = True    # 新增：静默模式
        self.max_retries = 999     # 新增：极大重试次数

# 修改点 2：错误处理（约 850 行）
def _handle_error(self, step, error):
    """错误处理"""
    if self.silent_mode:
        # 静默模式：不询问用户，自动尝试修复
        return self._auto_fix_error(step, error)
    else:
        # 正常模式：询问用户
        return self._ask_user_for_decision(step, error)
```

### 3. 修改 smart_router.py（1 处）

```python
# .trae/workflows/smart_router.py

# 在 process 方法中添加（约 120 行）
def process(self, task: str, files: List[str] = None) -> dict:
    """处理任务"""
    
    # ... 现有代码 ...
    
    # 新增：强制验证循环
    validation_result = self._run_validation(files)
    
    while not validation_result["all_passed"]:
        if self.auto_heal:
            self._auto_fix(validation_result["failures"])
            validation_result = self._run_validation(files)
        else:
            break
    
    return {
        "success": validation_result["all_passed"],
        "validation": validation_result
    }
```

---

## 使用方式

修改后，用户只需说：

```
开始：帮我优化网页性能
```

或

```
自主执行：重构用户认证模块
```

系统会自动：
1. 分析任务类型
2. 生成执行计划
3. 执行任务
4. 验证结果
5. 失败自动修复
6. 循环直到成功

**不会停下来询问用户**，除非遇到无法自动处理的严重错误。

---

## 风险控制

| 风险 | 缓解措施 |
|------|----------|
| 无限循环 | 设置最大尝试次数 999，超过后停止 |
| 修复方向错误 | 每次修复后重新验证，验证不通过继续尝试 |
| 破坏性操作 | 在 `project_rules.md` 中定义禁止操作 |

---

## 实施步骤

1. **修改 skill.yaml** - 1 分钟
2. **修改 autonomous_agent.py** - 5 分钟
3. **修改 smart_router.py** - 5 分钟
4. **测试验证** - 10 分钟

**总计：约 20 分钟**

---

## 验证方法

修改完成后，测试：

```bash
# 测试自主执行
python .trae/workflows/autonomous_agent.py task "创建一个简单的HTML页面"

# 测试智能路由
python .trae/workflows/smart_router.py "优化代码" --files src/app.js
```

预期结果：
- ✅ 执行过程不停止
- ✅ 完成后自动验证
- ✅ 验证失败自动修复
- ✅ 最终输出成功/失败报告
