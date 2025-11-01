# ReCode 项目详细技术文档

> 本文档旨在帮助初学者深入理解 ReCode 项目的设计思想、技术架构和实现细节。

## 目录

1. [项目概述](#项目概述)
2. [核心思想](#核心思想)
3. [技术架构](#技术架构)
4. [核心模块详解](#核心模块详解)
5. [工作流程](#工作流程)
6. [代码深度解析](#代码深度解析)
7. [环境配置与使用](#环境配置与使用)
8. [扩展开发指南](#扩展开发指南)

---

## 项目概述

### 什么是 ReCode？

ReCode 是一个**基于大语言模型（LLM）的智能代理（Agent）框架**，它通过**递归代码生成**的方式来解决复杂任务。

### 解决了什么问题？

传统的 LLM Agent 面临两个核心挑战：

1. **规划与执行分离**：传统方法中，高层规划（Planning）和底层行动（Action）是分开的，导致规划无法根据执行结果动态调整。

2. **粒度控制困难**：难以在宏观战略思考和微观具体行动之间灵活切换。

### ReCode 的创新点

ReCode 将**规划和行动统一为代码表示**：
- 高层规划 → 占位符函数（Placeholder Function）
- 底层行动 → 具体的可执行代码
- 通过**递归展开**的方式，将复杂任务逐层分解为可执行的原子操作

---

## 核心思想

### 1. 树形代码结构（Tree-structured Code）

ReCode 将任务组织成一棵**代码树**：

```
solve(instruction, observation)           # 根节点：总任务
├── find_target(instruction)              # 子任务1：找到目标
│   ├── run("look")                       # 原子操作
│   └── run("go to cabinet 1")            # 原子操作
├── pick_up_target(target)                # 子任务2：拾取目标
│   └── run("take mug 1 from cabinet 1")  # 原子操作
└── complete_task(target, location)       # 子任务3：完成任务
    ├── run("go to desk 1")               # 原子操作
    └── run("put mug 1 in/on desk 1")     # 原子操作
```

**关键特点**：
- 每个节点代表一个子任务
- 叶子节点是具体的可执行代码
- 非叶子节点是占位符函数，需要进一步展开

### 2. 递归展开（Recursive Expansion）

当遇到占位符函数时，LLM 会决定：

**情况A：直接执行**（任务简单）
```python
# 占位符函数
find_target(instruction)

# LLM 展开为具体代码
obs = run("look")
target = run("examine desk 1")
```

**情况B：继续分解**（任务复杂）
```python
# 占位符函数
solve(instruction, observation)

# LLM 分解为子任务
target = find_target(instruction)
pick_up_target(target)
complete_task(target, destination)
```

### 3. 动态执行循环（Dynamic Execution Loop）

ReCode 采用**边展开边执行**的策略：

```
1. 遇到占位符 → 调用 LLM 展开
2. 执行展开后的代码 → 获取环境反馈
3. 根据反馈决定下一步：
   - 成功 → 继续下一个节点
   - 失败 → 标记错误并尝试重试
   - 需要更多信息 → 继续展开
```

### 4. 共享执行器状态（Shared Executor State）

所有代码在同一个 **Python 执行器**中运行：
- 维护**全局变量**（如 `target`, `observation` 等）
- 提供**约束环境**（只能调用预定义的 `run()` 函数）
- 捕获**执行结果**和**错误信息**

---

## 技术架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                      run.py (主入口)                      │
│  - 解析命令行参数                                         │
│  - 管理并发执行                                           │
│  - 汇总结果并生成报告                                     │
└────────────┬────────────────────────────────────────────┘
             │
     ┌───────┴───────┐
     │               │
┌────▼────┐    ┌────▼─────┐
│  Agent  │    │   Env    │
│ (智能体) │◄───┤ (环境)   │
└────┬────┘    └──────────┘
     │
┌────▼──────────────────────────────────────────────────┐
│             ReCodeAgent (ReCode 智能体)                │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ AsyncLLM    │  │  Executor    │  │  CodeNode    │ │
│  │ (LLM接口)   │  │  (代码执行器) │  │  (代码树节点)│ │
│  └─────────────┘  └──────────────┘  └──────────────┘ │
└───────────────────────────────────────────────────────┘
```

### 核心组件关系

```
ReCodeAgent
    ├── 使用 AsyncLLM 调用大语言模型
    ├── 使用 Executor 执行生成的代码
    ├── 使用 CodeNode 构建代码树
    └── 与 Env 交互获取环境反馈
```

---

## 核心模块详解

### 1. ReCodeAgent（智能体核心）

**位置**: `agents/recode/agent.py`

#### 核心属性

```python
class ReCodeAgent(Agent):
    # LLM 接口，用于生成代码
    llm: AsyncLLM

    # 代码执行器，用于执行生成的代码
    executor: Executor

    # 代码树的根节点
    root: Optional[CodeNode]

    # 当前正在处理的节点
    current_node: Optional[CodeNode]

    # 环境名称（alfworld, webshop, sciworld）
    env_name: str

    # 任务类型（如 put, clean, heat 等）
    task_type: str
```

#### 核心方法

##### `reset()` - 重置智能体状态

```python
def reset(self, running_config: dict, init_info: dict=None) -> None:
    # 1. 清空代码树
    self.root = None
    self.current_node = None

    # 2. 加载配置参数
    self.max_depth = running_config.get('max_depth', 10)  # 最大递归深度
    self.max_retry = running_config.get('max_retry', 5)    # 最大重试次数

    # 3. 初始化 LLM
    if "profile" in running_config:
        self.llm = AsyncLLM(running_config['profile'])

    # 4. 设置环境
    self.env_name = init_info['env_name']
    self.executor.set_env(init_info['env'])

    # 5. 加载资源（提示词、示例）
    self._load_resources()
```

##### `act()` - 智能体的核心行动循环

```python
async def act(self, observations: List[str]) -> List[str]:
    # 第一次调用：初始化代码树
    if not self.is_start:
        self._init_code_tree(observations[0])
        self.is_start = True

    # 如果当前节点是占位符，需要展开
    if self.current_node.status == NodeStatus.STUB:
        await self._handle_stub()

    # 如果遇到错误，结束任务
    elif self.current_node.status == NodeStatus.ERROR:
        return ["[FINISH]"]

    # 执行当前节点的代码
    result = self._execute(self.current_node.code)

    # 根据执行结果更新节点状态
    if result["success"]:
        self.current_node.status = NodeStatus.COMPLETED
        self.current_node = self.current_node.next()  # 移动到下一个节点
    else:
        if "NeedExpansion" in result["error"]:
            # 需要展开（遇到未定义的函数）
            self.current_node.status = NodeStatus.STUB
        else:
            # 执行错误
            self.current_node.status = NodeStatus.ERROR
```

**工作流程解析**：

1. **初始化阶段**：从环境观察中提取任务描述，创建根节点 `solve(instruction, observation)`

2. **展开阶段**：如果节点是占位符（STUB），调用 `_handle_stub()` 让 LLM 展开

3. **执行阶段**：执行节点代码，获取环境反馈

4. **状态转移**：
   - 成功 → COMPLETED，移动到下一个节点
   - 需要展开 → STUB，在下一轮中展开
   - 错误 → ERROR，结束任务

##### `_handle_stub()` - 处理占位符节点

```python
async def _handle_stub(self) -> None:
    # 1. 检查是否达到最大深度
    if self.current_node.depth >= self.max_depth:
        self.logger.warning("Max depth reached - terminating.")
        self.current_node = None
        return

    # 2. 调用 LLM 展开当前节点
    new_blocks = await self._expand()

    # 3. 为展开的代码块创建子节点
    if new_blocks:
        for block in new_blocks:
            child_node = CodeNode(code=block, parent=self.current_node)
            self.current_node.children.append(child_node)

    # 4. 移动到第一个子节点
    self.current_node = self.current_node.next()
```

##### `_expand()` - 调用 LLM 展开占位符

```python
async def _expand(self) -> Optional[List[str]]:
    attempt = 0
    while True:
        # 1. 构建提示词
        user_prompt = self._build_expand_prompt()

        # 2. 调用 LLM
        response, _cost = await self.llm(user_prompt)

        # 3. 解析 LLM 输出
        thought = parse_xml_tag(response, "think")      # 思考过程
        expanded_code = parse_xml_tag(response, "execute")  # 生成的代码

        # 4. 验证代码语法
        try:
            blocks = split_blocks(expanded_code)  # 拆分为多个代码块
            validate_blocks(blocks)               # 验证语法
            return blocks
        except (SyntaxError, ValueError) as e:
            attempt += 1
            if attempt >= self.max_rewrite:
                return None  # 达到最大重试次数，放弃
            # 重试...
```

**关键点**：
- LLM 需要在 `<think>` 标签中解释思路
- 在 `<execute>` 标签中生成代码
- 代码会被验证语法，不合格会重新生成

##### `_build_expand_prompt()` - 构建提示词

```python
def _build_expand_prompt(self) -> str:
    return EXPAND_PROMPT.format(
        available_actions=self.available_actions,  # 可用的环境动作
        examples=self.fewshots,                     # 示例
        task=self.current_node.code,                # 当前任务
        variables=get_variables(self.executor, self.current_node.code)  # 可用变量
    )
```

**提示词包含**：
1. **可用动作列表**：环境提供的所有原子操作（从 `actions.txt` 加载）
2. **示例**：少样本学习的例子（从 `fewshots/` 加载）
3. **当前任务**：需要展开的占位符函数
4. **可用变量**：当前执行器中的所有变量

---

### 2. Executor（代码执行器）

**位置**: `utils/executor.py`

#### 核心功能

Executor 是一个**受约束的 Python 代码执行器**，负责：

1. 在隔离环境中执行代码
2. 捕获输出和错误
3. 管理变量状态
4. 提供与环境交互的接口

#### 核心属性

```python
class Executor:
    # 环境实例
    env: Env

    # 已执行的动作列表
    actions: List[str]

    # 执行器中的变量
    _variables: Dict[str, Any]

    # 基础全局命名空间（包含 run 函数等）
    _base_globals: Dict[str, Any]
```

#### 核心方法

##### `execute()` - 执行代码块

```python
def execute(self, code: str) -> Dict[str, Any]:
    # 调用 _run_block 执行代码
    success, stdout_lines, error_msg = self._run_block(code)

    return {
        "code": code,
        "stdout": stdout_lines,    # 标准输出
        "error": error_msg,         # 错误信息
        "success": success          # 是否成功
    }
```

##### `_run_block()` - 实际执行代码

```python
def _run_block(self, block: str) -> tuple[bool, List[str], str]:
    # 1. 捕获标准输出
    capture = OutputCapture()
    old_stdout = sys.stdout
    sys.stdout = capture

    # 2. 构建执行环境
    exec_globals = {**self._base_globals, **self._variables}

    try:
        # 3. 执行代码
        exec(block, exec_globals)

        # 4. 保存新变量
        for key, value in exec_globals.items():
            if self._is_preserved_variable(key, value):
                self._variables[key] = value

        return True, capture.lines, ""

    except NameError as e:
        # 检查是否是未定义的函数调用（需要展开）
        match = re.search(r"name '(.+?)' is not defined", str(e))
        if match and f"{match.group(1)}(" in block:
            return False, capture.lines, f"NeedExpansion: `{match.group(1)}` needs to be expanded."
        return False, capture.lines, f"NameError: {e}"

    except Exception as e:
        return False, capture.lines, f"{e.__class__.__name__}: {e}"

    finally:
        sys.stdout = old_stdout
```

**关键机制**：

1. **输出捕获**：通过替换 `sys.stdout` 捕获所有 `print` 输出

2. **变量保存**：执行后会保存所有新创建的变量到 `_variables`

3. **错误检测**：
   - 普通错误 → 返回错误信息
   - `NameError` 且是函数调用 → 返回 `NeedExpansion`，触发展开

##### `run()` - 与环境交互

```python
def run(self, action: str) -> str:
    # 1. 异步调用环境的 run 方法
    result = self._submit_coro(self.env.run(action))

    # 2. 记录动作
    self.actions.append(action)

    # 3. 返回观察结果
    if isinstance(result, list):
        result = "\n".join(result)
    return result
```

**这是唯一允许代码与环境交互的方法**！

##### `_submit_coro()` - 在独立线程中运行异步代码

```python
def _submit_coro(self, coro):
    # 在后台事件循环中运行协程
    future = asyncio.run_coroutine_threadsafe(coro, self._loop)
    return future.result()  # 阻塞等待结果
```

**为什么需要这个**？

因为代码是用 `exec()` 同步执行的，但环境的 `run()` 是异步的，所以需要一个桥接机制。

---

### 3. CodeNode（代码树节点）

**位置**: `agents/recode/utils.py`

#### 数据结构

```python
@dataclass
class CodeNode:
    # 节点的思考过程（LLM 生成）
    thought: str = ""

    # 节点的代码
    code: str = ""

    # 唯一标识符
    id: str = field(default_factory=lambda: str(uuid.uuid4()))

    # 父节点
    parent: Optional['CodeNode'] = None

    # 子节点列表
    children: List['CodeNode'] = field(default_factory=list)

    # 节点状态
    status: NodeStatus = NodeStatus.PENDING

    # 节点深度
    depth: int = 0

    # 错误信息
    error: str = None

    # 执行过程中的观察
    observations: List[str] = field(default_factory=list)
```

#### 节点状态（NodeStatus）

```python
class NodeStatus(str, Enum):
    PENDING = "PENDING"      # 待处理
    COMPLETED = "COMPLETED"  # 已完成
    STUB = "STUB"            # 占位符（需要展开）
    ERROR = "ERROR"          # 错误
    SKIP = "SKIP"            # 跳过
```

#### 核心方法

##### `next()` - 获取下一个待处理节点

```python
def next(self) -> Optional['CodeNode']:
    # 1. 优先处理子节点
    for child in self.children:
        if child.status == NodeStatus.PENDING:
            return child

    # 2. 如果没有待处理的子节点，找兄弟节点
    if self.parent:
        siblings = self.parent.children
        current_index = siblings.index(self)
        for i in range(current_index + 1, len(siblings)):
            if siblings[i].status == NodeStatus.PENDING:
                return siblings[i]

    # 3. 如果兄弟节点也处理完了，回到父节点继续
    if self.parent:
        return self.parent.next()

    # 4. 所有节点都处理完了
    return None
```

**遍历顺序**：深度优先（DFS）

**示例**：

```
solve()                        # 1. 当前节点
├── find_target()              # 2. 下一个
│   ├── run("look")            # 3. 继续深入
│   └── run("go to cabinet")   # 4. 兄弟节点
├── pick_up()                  # 5. 回到上层的兄弟
└── complete()                 # 6. 最后一个兄弟
```

---

### 4. 环境接口（Env）

**位置**: `base/environment.py`

#### 抽象接口

```python
class Env(ABC):
    @abstractmethod
    async def _run(self, action: str) -> Any:
        """执行单个动作，返回观察"""
        pass

    async def run(self, action: Union[str, List[str]]) -> List[str]:
        """执行一个或多个动作"""
        observations = []
        for single_action in action:
            observations.append(await self._run(single_action))
            if self.is_success():
                self._done = True
            if self.is_done():
                break
        return observations

    def is_done(self) -> bool:
        """是否结束"""
        return self._done

    def is_success(self) -> bool:
        """是否成功"""
        return self._success

    @abstractmethod
    def reset(self, running_config: dict, id: Optional[str] = None) -> dict:
        """重置环境，返回初始观察"""
        pass
```

#### 环境实现示例

##### ALFWorld 环境

**任务类型**：家庭任务模拟（如"把杯子放到桌子上"）

**可用动作**：
- `go to <location>`：移动到某个位置
- `take <object> from <location>`：从某处拿取物品
- `put <object> in/on <location>`：放置物品
- `open <container>`：打开容器
- `close <container>`：关闭容器
- `toggle <object>`：切换开关
- `heat <object> with <appliance>`：加热物品
- `cool <object> with <appliance>`：冷却物品
- `clean <object> with <appliance>`：清洁物品
- `examine <object>`：检查物品
- `inventory`：查看背包
- `look`：观察周围

##### WebShop 环境

**任务类型**：在线购物（如"买一个红色的杯子，价格低于30美元"）

**可用动作**：
- `search[<query>]`：搜索商品
- `click[<button>]`：点击按钮
- `choose[<option>]`：选择选项

##### SciWorld 环境

**任务类型**：科学实验模拟（如"测量物体的温度"）

**可用动作**：与科学仪器和对象交互

---

## 工作流程

### 完整执行流程

让我们通过一个完整的例子来理解 ReCode 的工作流程：

#### 任务示例：将杯子放到桌子上

##### 1. 初始化阶段

```python
# 环境返回初始观察
observation = """
You are in the middle of a room. Looking quickly around you, you see a cabinet 1,
a desk 1, a shelf 1, and a drawer 1.
Your task is to: put a mug in/on desk 1
"""

# ReCodeAgent 初始化代码树
instruction = "put a mug in/on desk 1"
root = CodeNode(code="solve(instruction, observation)")
```

**代码树状态**：
```
solve(instruction, observation)  [PENDING]
```

##### 2. 第一次展开

**当前节点**: `solve(instruction, observation)`

**LLM 输入**：
```
可用动作: go to, take, put, look, examine, ...
当前任务: solve(instruction, observation)
可用变量:
- instruction (str): put a mug in/on desk 1
- observation (str): You are in the middle of a room...
```

**LLM 输出**：
```xml
<think>
任务需要三个步骤：
1. 找到杯子
2. 拾取杯子
3. 将杯子放到桌子上
</think>

<execute>
target = find_mug(observation)
pick_up(target)
put_on_desk(target)
</execute>
```

**代码树状态**：
```
solve(instruction, observation)  [COMPLETED]
├── target = find_mug(observation)  [PENDING]
├── pick_up(target)                 [PENDING]
└── put_on_desk(target)             [PENDING]
```

##### 3. 展开 find_mug

**当前节点**: `target = find_mug(observation)`

**LLM 输入**：
```
可用动作: go to, take, put, look, examine, ...
当前任务: target = find_mug(observation)
可用变量:
- instruction (str): put a mug in/on desk 1
- observation (str): You are in the middle of a room...
```

**LLM 输出**：
```xml
<think>
需要在不同的位置查找杯子。根据观察，可能的位置有 cabinet 1, shelf 1, drawer 1。
我将依次检查这些位置。
</think>

<execute>
obs1 = run("go to cabinet 1")
obs2 = run("open cabinet 1")
obs3 = run("examine cabinet 1")
</execute>
```

**代码树状态**：
```
solve(instruction, observation)  [COMPLETED]
├── target = find_mug(observation)  [COMPLETED]
│   ├── obs1 = run("go to cabinet 1")      [PENDING]
│   ├── obs2 = run("open cabinet 1")       [PENDING]
│   └── obs3 = run("examine cabinet 1")    [PENDING]
├── pick_up(target)                        [PENDING]
└── put_on_desk(target)                    [PENDING]
```

##### 4. 执行 run("go to cabinet 1")

**执行结果**：
```
You arrive at loc 1. On the cabinet 1, you see a mug 1, and a plate 1.
```

**变量更新**：
```python
obs1 = "You arrive at loc 1. On the cabinet 1, you see a mug 1, and a plate 1."
```

**代码树状态**：
```
solve(instruction, observation)  [COMPLETED]
├── target = find_mug(observation)  [COMPLETED]
│   ├── obs1 = run("go to cabinet 1")      [COMPLETED]  ✓
│   ├── obs2 = run("open cabinet 1")       [PENDING]
│   └── obs3 = run("examine cabinet 1")    [PENDING]
├── pick_up(target)                        [PENDING]
└── put_on_desk(target)                    [PENDING]
```

##### 5. 继续执行后续动作...

（省略详细过程，类似上述步骤）

##### 6. 最终完成任务

**代码树最终状态**：
```
solve(instruction, observation)  [COMPLETED]
├── target = find_mug(observation)  [COMPLETED]
│   ├── obs1 = run("go to cabinet 1")      [COMPLETED]
│   ├── obs2 = run("open cabinet 1")       [COMPLETED]
│   └── obs3 = run("examine cabinet 1")    [COMPLETED]
├── pick_up(target)                        [COMPLETED]
│   └── run("take mug 1 from cabinet 1")   [COMPLETED]
└── put_on_desk(target)                    [COMPLETED]
    ├── run("go to desk 1")                [COMPLETED]
    └── run("put mug 1 in/on desk 1")      [COMPLETED]
```

**最终变量状态**：
```python
instruction = "put a mug in/on desk 1"
observation = "You are in the middle of a room..."
obs1 = "You arrive at loc 1. On the cabinet 1, you see a mug 1..."
obs2 = "You open the cabinet 1."
obs3 = "On the cabinet 1, you see a mug 1, and a plate 1."
target = "mug 1"
# ... 其他变量
```

---

### 状态机图

```
       ┌─────────────┐
       │   PENDING   │ ← 节点创建时
       └──────┬──────┘
              │
              ├──→ 尝试执行
              │
      ┌───────┴────────┐
      │                │
      ▼                ▼
┌──────────┐    ┌─────────────┐
│   STUB   │    │  COMPLETED  │
│(需要展开)│    │  (执行成功) │
└────┬─────┘    └─────────────┘
     │
     ├──→ 调用 LLM 展开
     │
     ├──→ 创建子节点 → PENDING
     │
     └──→ 展开失败 → ERROR
```

---

## 代码深度解析

### 1. 提示词工程（Prompt Engineering）

ReCode 的核心提示词位于 `agents/recode/resources/prompts/default_new.py`：

```python
EXPAND_PROMPT = """
You are the EXPAND step in the LLM Agent loop. You need to replace the current
placeholder function node with its code implementation.

Decide how to implement the placeholder:
- If the subtask can be done in 1-2 primitive actions, write them directly using `run(action)`.
- If it will take more than 2 primitive actions, break it into smaller placeholder functions.

All legal primitive actions are:
{available_actions}

All placeholder functions should be used in the format:
var_out1, var_out2, ... = snake_style_function_name(var_in1, var_in2="value", ...)

In your response:
1. Start with <think>...</think> to explain your reasoning
2. Output Python code in <execute>...</execute>

---
Examples:
{examples}
---

Current function to expand: {task}
Available variables: {variables}
"""
```

#### 提示词设计要点

1. **明确角色**：你是 EXPAND 步骤，负责展开占位符

2. **决策规则**：
   - 简单任务（1-2步）→ 直接写 `run()` 调用
   - 复杂任务（>2步）→ 分解为子任务

3. **格式约束**：
   - 使用 `<think>` 标签解释思路
   - 使用 `<execute>` 标签输出代码
   - 占位符函数必须使用 `snake_case` 命名
   - 可以通过关键字参数显式声明变量

4. **上下文信息**：
   - 可用的原子操作列表
   - 少样本示例
   - 当前任务
   - 可用变量及其类型和值

### 2. 少样本学习（Few-shot Learning）

以 ALFWorld 的 `put` 任务为例（`agents/recode/resources/fewshots/alfworld/put.txt`）：

```python
# 示例 1：展开根任务
Task: solve(instruction, observation)
Variables:
- instruction (str): put a clean lettuce in countertop
- observation (str): You are in a kitchen. You see a fridge 1, a countertop 1, ...

<think>
这个任务需要：
1. 找到生菜
2. 清洁生菜
3. 将生菜放到台面上
我将任务分解为这三个子任务。
</think>

<execute>
lettuce = find_lettuce(observation)
clean_lettuce(lettuce)
put_on_countertop(lettuce)
</execute>
```

```python
# 示例 2：展开 find_lettuce
Task: lettuce = find_lettuce(observation)
Variables:
- observation (str): You are in a kitchen. You see a fridge 1, a countertop 1, ...

<think>
需要在各个位置查找生菜。我先检查冰箱，因为生菜通常在冰箱里。
</think>

<execute>
obs1 = run("go to fridge 1")
obs2 = run("open fridge 1")
obs3 = run("look")
lettuce = "lettuce 1"
</execute>
```

**少样本示例的作用**：

1. **格式示范**：展示如何使用 `<think>` 和 `<execute>` 标签
2. **决策示范**：什么时候分解，什么时候直接执行
3. **领域知识**：生菜在冰箱里，杯子在柜子里
4. **变量命名**：如何命名子任务函数和变量

### 3. 代码拆分与验证

#### split_blocks() - 拆分代码块

```python
def split_blocks(source: str) -> List[str]:
    """将多行代码拆分为独立的代码块"""

    # 1. 尝试使用 AST 解析
    try:
        tree = ast.parse(source)

        # 2. 检查是否包含函数定义（不允许）
        for node in ast.walk(tree):
            if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
                raise ValueError("Function definitions not allowed")

        # 3. 按语句拆分
        lines = source.splitlines(True)
        return [
            "".join(lines[node.lineno - 1 : node.end_lineno])
            for node in tree.body
        ]
    except SyntaxError:
        # AST 解析失败，使用备用方法
        pass

    # 4. 备用方法：使用 codeop.CommandCompiler
    blocks = []
    buf = []
    compiler = codeop.CommandCompiler()

    for line in source.splitlines(True):
        buf.append(line)
        try:
            compiled = compiler("".join(buf), symbol="exec")
        except (SyntaxError, ValueError):
            # 当前行导致语法错误，尝试分离
            # ... (复杂的分离逻辑)
            pass

        if compiled is not None:
            # 成功编译，保存为一个块
            blocks.append("".join(buf))
            buf.clear()

    return blocks
```

**拆分示例**：

输入：
```python
obs1 = run("go to cabinet 1")
obs2 = run("open cabinet 1")
target = "mug 1"
```

输出：
```python
[
    'obs1 = run("go to cabinet 1")\n',
    'obs2 = run("open cabinet 1")\n',
    'target = "mug 1"\n'
]
```

#### validate_blocks() - 验证代码块

```python
def validate_blocks(blocks: List[str]) -> None:
    """验证所有代码块的语法正确性"""
    compiler = codeop.CommandCompiler()

    for block in blocks:
        # 1. 检查是否可编译
        compiled = compiler(block, symbol="exec")
        if compiled is None:
            raise SyntaxError("Incomplete Python block")

        # 2. 解析 AST
        tree = ast.parse(block)

        # 3. 检查是否包含函数定义
        for node in ast.walk(tree):
            if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
                raise ValueError("Function definitions not allowed")
```

**为什么不允许函数定义？**

因为 ReCode 的设计理念是：
- 高层抽象 → 占位符函数调用（会在运行时展开）
- 底层实现 → 直接的可执行代码（`run()` 调用）

如果允许定义函数，会破坏这个抽象层次。

### 4. 变量提取

#### get_variables() - 提取函数调用中的变量

```python
def get_variables(executor: Executor, code: str) -> str:
    """
    从占位符函数调用中提取参数变量

    例如：
    input:  find_target(instruction, location, max_depth=5)
    output:
    - instruction (str): put a mug in desk
    - location (str): kitchen
    - max_depth (int): 5
    """

    tree = ast.parse(code)
    discovered_var_names = []

    def collect_from_call(call: ast.Call):
        # 收集位置参数
        for arg in call.args:
            if isinstance(arg, ast.Name):
                discovered_var_names.append(arg.id)

        # 收集关键字参数
        for kw in call.keywords:
            if kw.arg is None:
                continue

            # 如果是字面量，直接设置变量
            literal_value = try_literal_eval(kw.value)
            if literal_value is not None:
                executor.set_var(kw.arg, literal_value)
                discovered_var_names.append(kw.arg)

            # 如果是变量引用
            elif isinstance(kw.value, ast.Name):
                discovered_var_names.append(kw.value.id)

    # 查找函数调用并收集变量
    for node in ast.walk(tree):
        if isinstance(node, ast.Call):
            collect_from_call(node)

    # 格式化输出
    lines = []
    for name in discovered_var_names:
        value = executor.get_var(name)
        value_type = executor._infer_type_string(value)
        lines.append(f"- {name} ({value_type}): {value}")

    return "\n".join(lines)
```

**示例**：

代码：
```python
result = find_and_clean_object(
    observation,
    target_type="mug",
    max_attempts=3
)
```

提取的变量：
```
- observation (str): You are in the middle of a room...
- target_type (str): mug
- max_attempts (int): 3
```

### 5. 错误处理机制

#### NeedExpansion 检测

在 `Executor._run_block()` 中：

```python
try:
    exec(block, exec_globals)
    return True, capture.lines, ""

except NameError as e:
    # 提取未定义的名称
    match = re.search(r"name '(.+?)' is not defined", str(e))

    # 检查是否是函数调用
    if match and f"{match.group(1)}(" in block:
        # 这是一个占位符函数，需要展开
        return False, [], f"NeedExpansion: `{match.group(1)}` needs to be expanded."

    # 普通的 NameError
    return False, [], f"NameError: {e}"
```

**工作原理**：

1. 代码：`target = find_mug(observation)`
2. 执行时抛出：`NameError: name 'find_mug' is not defined`
3. 检测到 `find_mug(` 存在 → 判定为占位符
4. 返回 `NeedExpansion` → 触发 `_expand()`

#### 重试机制

```python
async def _expand(self) -> Optional[List[str]]:
    attempt = 0
    retry_hint_added = False

    while True:
        # 构建提示词
        user_prompt = self._build_expand_prompt()

        # 如果之前失败过，添加重试提示
        if retry_hint_added:
            user_prompt += (
                "\n\n[Important] Your previous expansion produced invalid code. "
                "Please follow the rules strictly."
            )

        # 调用 LLM
        response, _cost = await self.llm(user_prompt)

        # 验证代码
        try:
            blocks = split_blocks(expanded_code)
            validate_blocks(blocks)
            return blocks
        except (SyntaxError, ValueError) as e:
            attempt += 1
            retry_hint_added = True

            # 达到最大重试次数
            if attempt >= self.max_rewrite:
                self.logger.info(f"Max retries reached. Giving up.")
                return None

            # 继续重试
            self.logger.info(f"Retry {attempt}/{self.max_rewrite} due to: {e}")
```

---

## 环境配置与使用

### 1. 安装依赖

```bash
# 克隆仓库
git clone https://github.com/your-username/ReCode.git
cd ReCode

# 安装 Python 依赖
pip install -r requirements.txt

# 安装特定环境
# ALFWorld
pip install alfworld

# SciWorld
pip install scienceworld

# WebShop（需要额外设置）
bash envs/webshop/setup.sh
```

### 2. 配置 LLM

编辑 `configs/profiles.yaml`：

```yaml
models:
  default:
    api_key: "sk-your_api_key_here"
    base_url: "https://api.openai.com/v1"
    model: "gpt-4o-mini"
    temperature: 0.0
    track_costs: true

  gpt-4o:
    api_key: "sk-your_api_key_here"
    base_url: "https://api.openai.com/v1"
    model: "gpt-4o"
    temperature: 0.7
    max_tokens: 512

  claude:
    api_key: "sk-ant-your_key"
    base_url: "https://api.anthropic.com/v1"
    model: "claude-sonnet-4-5"
    temperature: 0.0
```

### 3. 设置环境数据

#### ALFWorld

```bash
# 下载数据集
export ALFWORLD_DATA=/path/to/alfworld

# 或编辑配置文件
vim envs/alfworld/base_config.yaml
```

#### WebShop

```bash
# 自动下载和设置
bash envs/webshop/setup.sh
```

### 4. 运行示例

#### 基础运行

```bash
# ALFWorld，单个实例
python run.py -a recode -e alfworld -n 1 --split test --profile default
```

#### 批量测试

```bash
# WebShop，10个实例，并发度为3
python run.py -a recode -e webshop -n 10 -c 3 --profile gpt-4o
```

#### 使用配置文件

创建 `configs/my_experiment.yaml`：

```yaml
agent: recode
env: alfworld
instances: 50
concurrent: 5
profile: gpt-4o
split: test
task_types: ["put", "clean", "heat"]
max_depth: 12
max_retry: 5
```

运行：

```bash
python run.py -C configs/my_experiment.yaml
```

### 5. 查看结果

运行后会生成：

```
logs/
└── 20250101_120000_ReCodeAgent_AlfworldEnv/
    ├── running_logs/
    │   ├── run.log              # 总日志
    │   ├── instance_0.log       # 实例0的详细日志
    │   ├── instance_1.log       # 实例1的详细日志
    │   └── ...
    └── results.json             # 结果汇总
```

#### results.json 格式

```json
{
  "summary": {
    "total_instances": 10,
    "successful_instances": 8,
    "success_rate": 0.8,
    "metrics_total": {
      "time": 450.5,
      "steps": 125,
      "cost": 2.34
    },
    "metrics_avg": {
      "time": 45.05,
      "steps": 12.5,
      "cost": 0.234
    }
  },
  "by_task_type": {
    "put": {
      "total_instances": 5,
      "successful_instances": 4,
      "success_rate": 0.8,
      "avg_time_per_instance": 40.2,
      "avg_steps_per_instance": 11.0
    },
    "clean": {
      "total_instances": 5,
      "successful_instances": 4,
      "success_rate": 0.8,
      "avg_time_per_instance": 50.8,
      "avg_steps_per_instance": 14.0
    }
  },
  "instances": [
    {
      "instance_id": 0,
      "success": true,
      "time": 45.2,
      "steps": 12,
      "cost": 0.25,
      "task_type": "put"
    },
    ...
  ]
}
```

---

## 扩展开发指南

### 1. 添加新环境

#### 步骤 1：实现环境类

在 `envs/my_env/env.py` 中：

```python
from base.environment import Env
from typing import Optional

class MyEnv(Env):
    def __init__(self, logger=None):
        self.logger = logger
        self.id = "my_env"
        self._step_count = 0
        self._done = False
        self._success = False

    def reset(self, running_config: dict, id: Optional[str] = None) -> dict:
        """重置环境"""
        self._step_count = 0
        self._done = False
        self._success = False

        # 加载任务
        task = self._load_task(running_config)

        # 返回初始观察
        return {
            "observations": [task.description],
            "env_name": "my_env",
            "env": self,
            "task_type": task.type
        }

    async def _run(self, action: str) -> str:
        """执行动作"""
        self._step_count += 1

        # 执行动作，获取观察
        observation = self._execute_action(action)

        # 检查是否成功
        if self._check_success():
            self._success = True
            self._done = True

        # 检查是否达到步数上限
        if self._step_count >= self.max_steps:
            self._done = True

        return observation

    def is_done(self) -> bool:
        return self._done

    def is_success(self) -> bool:
        return self._success

    def report(self) -> dict:
        """返回统计信息"""
        return {
            "success": self._success,
            "steps": self._step_count,
            "task_type": self.current_task_type
        }
```

#### 步骤 2：定义可用动作

在 `agents/recode/resources/prompts/my_env/actions.txt` 中：

```
Available actions:
- action1[param]: Description of action1
- action2[param1, param2]: Description of action2
- action3: Description of action3
```

#### 步骤 3：编写少样本示例

在 `agents/recode/resources/fewshots/my_env/base.txt` 中：

```
Example 1:
Task: solve(instruction, observation)
Variables:
- instruction (str): Your task description
- observation (str): Initial observation

<think>
Explanation of how to approach this task...
</think>

<execute>
step1 = subtask1(observation)
step2 = subtask2(step1)
finalize(step2)
</execute>

---

Example 2:
Task: subtask1(observation)
Variables:
- observation (str): Current state

<think>
This subtask can be done directly with primitive actions.
</think>

<execute>
result1 = run("action1[params]")
result2 = run("action2[params]")
</execute>
```

#### 步骤 4：注册环境

在 `run.py` 中：

```python
ENV_ALIASES = {
    "alfworld": "envs.alfworld.env.AlfworldEnv",
    "webshop": "envs.webshop.env.WebShopEnv",
    "sciworld": "envs.sciworld.env.SciWorldEnv",
    "my_env": "envs.my_env.env.MyEnv",  # 添加这一行
}
```

#### 步骤 5：更新 ReCodeAgent

在 `agents/recode/agent.py` 的 `_load_resources()` 中：

```python
def _load_resources(self):
    resources_path = Path("agents/recode/resources/prompts") / self.env_name
    self.available_actions = open(resources_path / "actions.txt", "r").read()

    fewshots_path = Path("agents/recode/resources/fewshots") / self.env_name

    if self.env_name == "my_env":
        self.fewshots = open(fewshots_path / "base.txt", "r").read()
    # ... 其他环境
```

在 `agents/recode/utils.py` 的 `parse_raw_observation()` 中：

```python
def parse_raw_observation(raw_observation: str, env_name: str) -> tuple[str, str]:
    if env_name == "my_env":
        # 解析你的环境的观察格式
        # 返回 (observation, instruction)
        return raw_observation, extract_instruction(raw_observation)
    # ... 其他环境
```

#### 步骤 6：测试

```bash
python run.py -a recode -e my_env -n 1 --profile default
```

### 2. 自定义 Agent

#### 实现 Agent 接口

```python
from base.agent import Agent
from typing import List

class MyAgent(Agent):
    def __init__(self, logger=None):
        self.logger = logger
        # 初始化你的组件

    async def act(self, observations: List[str]) -> List[str]:
        """
        根据观察返回动作列表

        Args:
            observations: 环境返回的观察列表

        Returns:
            actions: 要执行的动作列表
        """
        # 你的决策逻辑
        actions = self._make_decision(observations)
        return actions

    def reset(self, running_config: dict, init_info: dict = None) -> None:
        """重置 agent 状态"""
        # 重置逻辑
        pass

    def report(self) -> dict:
        """返回统计信息"""
        return {
            "cost": self.total_cost,
            "steps": self.step_count,
            # 其他指标
        }
```

#### 注册 Agent

在 `run.py` 中：

```python
AGENT_ALIASES = {
    "recode": "agents.recode.agent.ReCodeAgent",
    "my_agent": "agents.my_agent.agent.MyAgent",  # 添加这一行
}
```

### 3. 扩展 Executor 功能

#### 注册自定义函数

```python
from utils.executor import Executor

executor = Executor(env=env)

# 注册普通函数
def my_helper_function(x, y):
    return x + y

executor.register_function("my_func", my_helper_function)

# 注册动作函数（自动调用 run）
def custom_action(param):
    return f"custom_action[{param}]"

executor.register_action_function("custom_act", custom_action)
```

然后在生成的代码中可以使用：

```python
# 普通函数
result = my_func(10, 20)  # 返回 30

# 动作函数（会自动调用 run）
obs = custom_act("param")  # 等价于 run("custom_action[param]")
```

#### 注册 LLM 查询功能

```python
from utils.llm import AsyncLLM

llm = AsyncLLM(profile="default")
executor.register_ask_llm(llm)
```

然后在代码中可以：

```python
# 在生成的代码中调用 LLM
response = ask_llm("What is the capital of France?")
print(response)  # "Paris"
```

---

## 常见问题（FAQ）

### Q1: 为什么使用代码而不是自然语言表示计划？

**A**: 代码提供了几个关键优势：

1. **结构化**：代码有明确的语法和语义，易于解析和执行
2. **可执行**：可以直接运行，无需额外的解释层
3. **状态管理**：可以使用变量保存中间结果
4. **控制流**：支持条件、循环等复杂逻辑
5. **类型系统**：可以推断变量类型，提供更好的上下文

### Q2: 为什么不允许定义函数（def）？

**A**: ReCode 的设计哲学是：

- **占位符函数** = 高层抽象（运行时展开）
- **run() 调用** = 底层实现（直接执行）

如果允许定义函数，就模糊了这两个层次：
- 无法区分哪些函数需要展开，哪些不需要
- 破坏了"运行时展开"的机制
- 增加了复杂性（需要管理函数定义的作用域）

### Q3: 如何控制展开的粒度？

**A**: 通过提示词中的规则：

```
- If the subtask can be done in 1-2 primitive actions, write them directly.
- If it will take more than 2 primitive actions, break it into smaller functions.
```

这个规则告诉 LLM：
- 简单任务 → 直接写 `run()` 调用
- 复杂任务 → 分解为子任务

你可以修改这个规则来控制粒度。

### Q4: 如何处理循环和条件？

**A**: 当前版本的 ReCode 主要通过**隐式循环**处理：

例如，查找物品：
```python
# LLM 生成
obs1 = run("go to cabinet 1")
obs2 = run("examine cabinet 1")
# 如果没找到，继续生成
obs3 = run("go to drawer 1")
obs4 = run("examine drawer 1")
```

如果需要显式循环，可以在 Executor 中注册辅助函数：

```python
def search_locations(locations, target):
    for loc in locations:
        obs = run(f"go to {loc}")
        if target in obs:
            return loc
    return None

executor.register_function("search_locations", search_locations)
```

### Q5: 如何优化性能？

**A**: 几个优化方向：

1. **减少 LLM 调用**：
   - 使用更小的模型（gpt-4o-mini）
   - 增加每次展开的步数（修改提示词规则）
   - 使用缓存（相同任务重用结果）

2. **并发执行**：
   ```bash
   python run.py -a recode -e alfworld -n 100 -c 10  # 10个并发
   ```

3. **优化提示词**：
   - 减少示例数量
   - 使用更精简的描述

### Q6: 如何调试？

**A**: 几个调试技巧：

1. **查看日志**：
   ```bash
   tail -f logs/<run_id>/running_logs/run.log
   ```

2. **查看 LLM 输入输出**：
   日志中包含 `[LLM_IN]` 和 `[LLM_OUT]` 标记

3. **单实例运行**：
   ```bash
   python run.py -a recode -e alfworld -n 1
   ```

4. **添加调试输出**：
   在 `ReCodeAgent.act()` 中添加 `print()` 或 `logger.info()`

5. **使用 Python 调试器**：
   ```python
   import pdb
   pdb.set_trace()
   ```

---

## 总结

ReCode 是一个创新的 LLM Agent 框架，通过以下核心机制实现了高效的任务规划和执行：

1. **统一表示**：将规划和行动统一为代码
2. **递归分解**：通过占位符函数逐层分解任务
3. **动态执行**：边展开边执行，根据反馈调整
4. **约束执行**：在受控环境中安全执行代码

**核心组件**：
- **ReCodeAgent**：智能体核心，管理代码树和执行流程
- **Executor**：代码执行器，提供受约束的执行环境
- **CodeNode**：代码树节点，组织任务结构
- **AsyncLLM**：LLM 接口，生成代码

**工作流程**：
1. 初始化代码树（根节点 = 总任务）
2. 深度优先遍历节点
3. 遇到占位符 → 调用 LLM 展开
4. 执行代码 → 获取反馈
5. 更新状态 → 继续下一个节点

**适用场景**：
- 需要多步规划的任务
- 需要动态调整的任务
- 需要层次化分解的复杂任务

希望这份详细文档能帮助你深入理解 ReCode 的设计和实现！

---

## 参考资源

- **论文**: [ReCode: Unify Plan and Action for Universal Granularity Control](https://arxiv.org/abs/2510.23564)
- **代码仓库**: https://github.com/your-username/ReCode
- **相关项目**:
  - [ALFWorld](https://github.com/alfworld/alfworld)
  - [ScienceWorld](https://github.com/allenai/ScienceWorld)
  - [WebShop](https://github.com/princeton-nlp/WebShop)

## 更新日志

- **2025-01-01**: 创建详细中文文档

---

**如有问题，欢迎联系**: zhaoyangyu713@gmail.com
