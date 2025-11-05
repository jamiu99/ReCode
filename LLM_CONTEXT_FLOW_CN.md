# ReCode 中 LLM 的上下文变化全流程

> 本文档详细展示 LLM 在 ReCode 运行过程中，每次调用时的输入是什么，上下文如何变化

---

## 📋 目录

1. [提示词结构](#提示词结构)
2. [完整执行流程示例](#完整执行流程示例)
3. [上下文变化追踪](#上下文变化追踪)
4. [关键机制解析](#关键机制解析)

---

## 提示词结构

### LLM 输入的四个部分

每次调用 LLM 时，输入都包含以下四个部分：

```
┌─────────────────────────────────────────────────────────┐
│  1️⃣ 系统指令（固定）                                    │
│     - 你是 EXPAND 步骤                                  │
│     - 决策规则（何时分解，何时直接执行）                 │
│     - 格式要求（<think> 和 <execute> 标签）             │
├─────────────────────────────────────────────────────────┤
│  2️⃣ 可用动作列表（环境相关，固定）                      │
│     - go to {loc_ID}                                    │
│     - take {obj_ID} from {loc_ID}                       │
│     - open {loc_ID}                                     │
│     - ...                                               │
├─────────────────────────────────────────────────────────┤
│  3️⃣ 示例（Few-shot Examples）（任务类型相关，固定）     │
│     - 类似任务的完整示例                                │
│     - 展示如何思考和生成代码                            │
├─────────────────────────────────────────────────────────┤
│  4️⃣ 当前上下文（动态变化）⬅️ 这是关键！                │
│     - 当前需要展开的任务                                │
│     - 可用的变量及其值                                  │
└─────────────────────────────────────────────────────────┘
```

**关键点**：
- 前三部分在同一个环境、同一个任务类型中是**固定的**
- 只有第四部分（当前上下文）是**动态变化**的
- **上下文变化**主要体现在：
  1. 当前任务从高层到底层
  2. 可用变量越来越多

---

## 完整执行流程示例

### 任务：清洗苹果并放到桌子上

让我们完整追踪一次任务执行中，LLM 的每次调用。

---

### 🔄 第 1 次 LLM 调用

#### 背景

```
环境初始化完成
任务描述：clean some apple and put it in sidetable
环境观察：You are in the middle of a room. Looking quickly around you,
         you see a cabinet 4, a cabinet 3, ..., a sinkbasin 1, ...

创建根节点：solve(instruction, observation)
当前节点状态：PENDING
执行器变量：instruction = "clean some apple and put it in sidetable"
          observation = "You are in the middle of a room..."
```

#### LLM 输入

```markdown
═══════════════════════════════════════════════════════════
📥 LLM 输入 - 第 1 次调用
═══════════════════════════════════════════════════════════

【系统指令】
You are the EXPAND step in the LLM Agent loop. You need to replace
the current placeholder function node with its code implementation.

Decide how to implement the placeholder:
- If the subtask can be done in 1-2 primitive actions, write them
  directly using `run(action: str)`.
- If it will take more than 2 primitive actions, break it into
  smaller placeholder functions.

All legal primitive actions are:
【可用动作列表】
- go to {loc_ID}
- open {loc_ID}
- close {loc_ID}
- take {obj_ID} from {loc_ID}
- move {obj_ID} to {loc_ID}
- clean {obj_ID} with {sinkbasin_ID}
- examine {loc_ID}
- inventory

【示例 - 共 4 个类似任务的完整示例】
[Input]
The current function to expand is:
solve(instruction, observation)
The variables you can use is:
- instruction (str): Your task is to: clean some apple and put it in sidetable.
- observation (str): You are in the middle of a room...

[Output]
<think>
I first characterize the task: the object is an apple that must be
cleaned and finally placed on the sidetable...
</think>

<execute>
cleaning_location_ID = 'sinkbasin 1'
target_location_ID = 'sidetable 1'
obj = 'apple'
all_location_IDs = re.findall(r'[a-z]+(?:[a-z]+)? \d+', observation)

obj_ID = find_and_take(obj, all_location_IDs)
clean_with(obj_ID, cleaning_location_ID)
put_in(obj_ID, target_location_ID)
</execute>

... (其他 3 个示例)

(End of Examples)
---

【当前上下文】⬅️ 动态部分
The current function to expand is:
solve(instruction, observation)

The variables you can use is:
- instruction (str): clean some apple and put it in sidetable
- observation (str): You are in the middle of a room. Looking quickly
  around you, you see a cabinet 4, a cabinet 3, a cabinet 2, a cabinet 1,
  a coffeemachine 1, a countertop 1, a diningtable 3, a diningtable 2,
  a diningtable 1, a drawer 1, a fridge 1, a garbagecan 1, a microwave 1,
  a sidetable 1, a sinkbasin 1, a stoveburner 4, a stoveburner 3,
  a stoveburner 2, a stoveburner 1, and a toaster 1.

═══════════════════════════════════════════════════════════
```

#### LLM 输出

```markdown
═══════════════════════════════════════════════════════════
📤 LLM 输出 - 第 1 次调用
═══════════════════════════════════════════════════════════

<think>
这个任务需要：
1. 找到苹果并拿取
2. 清洗苹果（必须在 sinkbasin）
3. 将苹果放到 sidetable

我将把任务分解为三个子任务，并提取所有位置 ID 以便搜索。
</think>

<execute>
cleaning_location_ID = 'sinkbasin 1'
target_location_ID = 'sidetable 1'
obj = 'apple'
all_location_IDs = re.findall(r'[a-z]+(?:[a-z]+)? \d+', observation)

obj_ID = find_and_take(obj, all_location_IDs)
clean_with(obj_ID, cleaning_location_ID)
put_in(obj_ID, target_location_ID)
</execute>

═══════════════════════════════════════════════════════════
```

#### 代码解析和执行

```python
# ReCode 解析 LLM 输出，拆分为代码块：

块 1: cleaning_location_ID = 'sinkbasin 1'
块 2: target_location_ID = 'sidetable 1'
块 3: obj = 'apple'
块 4: all_location_IDs = re.findall(r'[a-z]+(?:[a-z]+)? \d+', observation)
块 5: obj_ID = find_and_take(obj, all_location_IDs)
块 6: clean_with(obj_ID, cleaning_location_ID)
块 7: put_in(obj_ID, target_location_ID)

# 创建代码树节点
根节点 solve() → COMPLETED（已展开）
  ├─ 子节点 1: cleaning_location_ID = 'sinkbasin 1' → PENDING
  ├─ 子节点 2: target_location_ID = 'sidetable 1' → PENDING
  ├─ 子节点 3: obj = 'apple' → PENDING
  ├─ 子节点 4: all_location_IDs = re.findall(...) → PENDING
  ├─ 子节点 5: obj_ID = find_and_take(...) → PENDING
  ├─ 子节点 6: clean_with(...) → PENDING
  └─ 子节点 7: put_in(...) → PENDING
```

#### 执行块 1-4（设置变量）

```python
# 执行块 1
cleaning_location_ID = 'sinkbasin 1'
✅ 成功，保存变量

# 执行块 2
target_location_ID = 'sidetable 1'
✅ 成功，保存变量

# 执行块 3
obj = 'apple'
✅ 成功，保存变量

# 执行块 4
all_location_IDs = re.findall(r'[a-z]+(?:[a-z]+)? \d+', observation)
✅ 成功，保存变量
   all_location_IDs = ['cabinet 4', 'cabinet 3', ..., 'toaster 1']
```

#### 当前执行器状态

```python
执行器变量更新：
{
    'instruction': 'clean some apple and put it in sidetable',
    'observation': 'You are in the middle of a room...',
    'cleaning_location_ID': 'sinkbasin 1',          # 新增
    'target_location_ID': 'sidetable 1',            # 新增
    'obj': 'apple',                                 # 新增
    'all_location_IDs': ['cabinet 4', 'cabinet 3', # 新增
                         'cabinet 2', 'cabinet 1',
                         'coffeemachine 1', ...]
}
```

#### 执行块 5（遇到占位符）

```python
# 尝试执行块 5
obj_ID = find_and_take(obj, all_location_IDs)

❌ 错误：NameError: name 'find_and_take' is not defined

# Executor 检测到这是一个函数调用
# 判定：这是占位符，需要展开
# 标记节点状态：STUB

当前节点：obj_ID = find_and_take(obj, all_location_IDs)
节点状态：STUB → 需要调用 LLM 展开
```

---

### 🔄 第 2 次 LLM 调用

#### 背景

```
当前节点：obj_ID = find_and_take(obj, all_location_IDs)
节点状态：STUB（需要展开）
执行器变量：instruction, observation, cleaning_location_ID,
          target_location_ID, obj, all_location_IDs
```

#### LLM 输入

```markdown
═══════════════════════════════════════════════════════════
📥 LLM 输入 - 第 2 次调用
═══════════════════════════════════════════════════════════

【系统指令】（相同，省略）
【可用动作列表】（相同，省略）
【示例】（相同，但现在示例中包含 find_and_take 的展开案例）

... (示例展示如何展开 find_and_take)

---

【当前上下文】⬅️ 注意变化！
The current function to expand is:
obj_ID = find_and_take(obj, all_location_IDs)

The variables you can use is:
- obj (str): apple
- all_location_IDs (list[str]): ['cabinet 4', 'cabinet 3', 'cabinet 2',
  'cabinet 1', 'coffeemachine 1', 'countertop 1', 'diningtable 3',
  'diningtable 2', 'diningtable 1', 'drawer 1', 'fridge 1', 'garbagecan 1',
  'microwave 1', 'sidetable 1', 'sinkbasin 1', 'stoveburner 4',
  'stoveburner 3', 'stoveburner 2', 'stoveburner 1', 'toaster 1']

═══════════════════════════════════════════════════════════
```

**关键变化**：
1. ✅ **当前任务**从 `solve(...)` 变成了 `find_and_take(...)`
2. ✅ **可用变量**只提取了与当前任务相关的：`obj` 和 `all_location_IDs`
3. ✅ 变量的**值**已经是具体的（不再是 observation 的原始文本）

#### LLM 输出

```markdown
═══════════════════════════════════════════════════════════
📤 LLM 输出 - 第 2 次调用
═══════════════════════════════════════════════════════════

<think>
目标是在房间中系统地搜索苹果并拾取它。
我将遍历所有位置：
1. 先导航到位置
2. 如果需要，检查位置
3. 如果是关闭的容器，先打开
4. 一旦发现苹果，提取具体的 ID（如 "apple 1"）并拾取
</think>

<execute>
for location_ID in all_location_IDs:
    obs = run(f'go to {location_ID}')
    if 'Nothing happens.' in obs:
        obs = run(f'examine {location_ID}')
    location_closed = 'closed' in obs
    if location_closed:
        obs = run(f'open {location_ID}')
    if f" {obj} " in obs:
        obj_ID = re.search(rf'\b{obj} \d+\b', obs).group()
        run(f'take {obj_ID} from {location_ID}')
        break
</execute>

═══════════════════════════════════════════════════════════
```

#### 代码解析和执行

```python
# ReCode 解析输出，拆分为代码块

块 1: for location_ID in all_location_IDs: ...（整个循环是一个块）

# 创建子节点
节点 obj_ID = find_and_take(...) → COMPLETED（已展开）
  └─ 子节点 1: for location_ID in all_location_IDs: ... → PENDING
```

#### 执行循环（与环境交互）

```python
# 开始执行循环

迭代 1: location_ID = 'cabinet 4'
  执行: run('go to cabinet 4')
  环境返回: "You arrive at loc 1. On the cabinet 4, you see nothing."
  保存: obs = "You arrive at loc 1..."
  检查: 'Nothing happens.' in obs → False
  检查: 'closed' in obs → False
  检查: ' apple ' in obs → False
  继续下一个位置

迭代 2: location_ID = 'cabinet 3'
  执行: run('go to cabinet 3')
  环境返回: "You arrive at loc 2. The cabinet 3 is closed."
  保存: obs = "You arrive at loc 2. The cabinet 3 is closed."
  检查: 'Nothing happens.' in obs → False
  检查: 'closed' in obs → True ✅
  执行: run('open cabinet 3')
  环境返回: "You open the cabinet 3. The cabinet 3 is open. In it, you see nothing."
  保存: obs = "You open the cabinet 3..."
  检查: ' apple ' in obs → False
  继续下一个位置

迭代 3: location_ID = 'cabinet 2'
  执行: run('go to cabinet 2')
  环境返回: "You arrive at loc 3. The cabinet 2 is closed."
  检查: 'closed' in obs → True
  执行: run('open cabinet 2')
  环境返回: "You open the cabinet 2. In it, you see an apple 1."
  保存: obs = "You open the cabinet 2. In it, you see an apple 1."
  检查: ' apple ' in obs → True ✅ 找到了！

  提取 ID:
    re.search(r'\bapple \d+\b', obs).group()
    → 'apple 1'
  保存: obj_ID = 'apple 1'

  执行: run('take apple 1 from cabinet 2')
  环境返回: "You pick up the apple 1 from the cabinet 2."

  break  # 退出循环
```

#### 当前执行器状态

```python
执行器变量更新：
{
    'instruction': 'clean some apple and put it in sidetable',
    'observation': 'You are in the middle of a room...',
    'cleaning_location_ID': 'sinkbasin 1',
    'target_location_ID': 'sidetable 1',
    'obj': 'apple',
    'all_location_IDs': ['cabinet 4', ...],
    'location_ID': 'cabinet 2',                    # 新增（循环变量）
    'obs': 'You open the cabinet 2. In it, ...',   # 新增
    'location_closed': True,                        # 新增
    'obj_ID': 'apple 1'                            # 新增 ⬅️ 关键！
}
```

#### 移动到下一个节点

```python
当前节点：for location_ID in all_location_IDs: ... → COMPLETED ✅
移动到：clean_with(obj_ID, cleaning_location_ID) → PENDING
节点状态：PENDING → 尝试执行
```

---

### 🔄 第 3 次 LLM 调用

#### 背景

```
当前节点：clean_with(obj_ID, cleaning_location_ID)
节点状态：PENDING → 尝试执行 → 发现是占位符 → STUB
执行器变量：包含所有之前的变量，特别是 obj_ID = 'apple 1'
```

#### LLM 输入

```markdown
═══════════════════════════════════════════════════════════
📥 LLM 输入 - 第 3 次调用
═══════════════════════════════════════════════════════════

【系统指令】（相同，省略）
【可用动作列表】（相同，省略）
【示例】（包含 clean_with 的展开示例）

---

【当前上下文】⬅️ 再次变化！
The current function to expand is:
clean_with(obj_ID, cleaning_location_ID)

The variables you can use is:
- obj_ID (str): apple 1
- cleaning_location_ID (str): sinkbasin 1

═══════════════════════════════════════════════════════════
```

**关键变化**：
1. ✅ **当前任务**变成了 `clean_with(...)`
2. ✅ **可用变量**只包含相关的两个：`obj_ID` 和 `cleaning_location_ID`
3. ✅ `obj_ID` 的值已经是具体的 `'apple 1'`（来自上一步的执行结果）

#### LLM 输出

```markdown
═══════════════════════════════════════════════════════════
📤 LLM 输出 - 第 3 次调用
═══════════════════════════════════════════════════════════

<think>
清洗必须在 sinkbasin 进行。
我需要：
1. 导航到清洗位置
2. 使用清洗动作
</think>

<execute>
run(f'go to {cleaning_location_ID}')
run(f'clean {obj_ID} with {cleaning_location_ID}')
</execute>

═══════════════════════════════════════════════════════════
```

#### 执行

```python
# 执行块 1
run(f'go to {cleaning_location_ID}')
→ run('go to sinkbasin 1')
环境返回: "You arrive at loc 5. On the sinkbasin 1, you see nothing."
✅ 成功

# 执行块 2
run(f'clean {obj_ID} with {cleaning_location_ID}')
→ run('clean apple 1 with sinkbasin 1')
环境返回: "You clean the apple 1 using the sinkbasin 1."
✅ 成功

当前节点：clean_with(...) → COMPLETED ✅
移动到：put_in(obj_ID, target_location_ID) → PENDING
```

---

### 🔄 第 4 次 LLM 调用

#### 背景

```
当前节点：put_in(obj_ID, target_location_ID)
节点状态：STUB（需要展开）
执行器变量：所有之前的变量
```

#### LLM 输入

```markdown
═══════════════════════════════════════════════════════════
📥 LLM 输入 - 第 4 次调用
═══════════════════════════════════════════════════════════

【系统指令】（相同）
【可用动作列表】（相同）
【示例】（包含 put_in 的展开示例）

---

【当前上下文】
The current function to expand is:
put_in(obj_ID, target_location_ID)

The variables you can use is:
- obj_ID (str): apple 1
- target_location_ID (str): sidetable 1

═══════════════════════════════════════════════════════════
```

#### LLM 输出

```markdown
═══════════════════════════════════════════════════════════
📤 LLM 输出 - 第 4 次调用
═══════════════════════════════════════════════════════════

<think>
我需要：
1. 导航到目标位置
2. 检查是否需要打开（虽然 sidetable 是表面，不需要打开）
3. 将物品放置上去
</think>

<execute>
obs = run(f'go to {target_location_ID}')
if 'Nothing happens.' in obs:
    obs = run(f'examine {target_location_ID}')
location_closed = 'closed' in obs
if location_closed:
    obs = run(f'open {target_location_ID}')
run(f'move {obj_ID} to {target_location_ID}')
</execute>

═══════════════════════════════════════════════════════════
```

#### 执行

```python
# 执行
obs = run('go to sidetable 1')
环境返回: "You arrive at loc 8. On the sidetable 1, you see a cd 1."
✅ 成功

检查: 'Nothing happens.' in obs → False
检查: 'closed' in obs → False

run('move apple 1 to sidetable 1')
环境返回: "You put the apple 1 in/on the sidetable 1."
✅ 成功！

🎉 任务完成！
```

---

## 上下文变化追踪

### 可视化上下文演变

```
第 1 次调用
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📍 任务：solve(instruction, observation)
📦 变量：
   - instruction (str): "clean some apple..."
   - observation (str): "You are in the middle..."
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        ↓
    LLM 生成代码
        ↓
    执行并保存变量
        ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第 2 次调用
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📍 任务：obj_ID = find_and_take(obj, all_location_IDs)
📦 变量：⬅️ 只提取相关的
   - obj (str): "apple"
   - all_location_IDs (list[str]): ['cabinet 4', ...]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        ↓
    LLM 生成循环代码
        ↓
    执行循环，与环境交互
        ↓
    保存 obj_ID = 'apple 1'
        ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第 3 次调用
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📍 任务：clean_with(obj_ID, cleaning_location_ID)
📦 变量：⬅️ 使用上一步的结果
   - obj_ID (str): "apple 1"  ⬅️ 来自第 2 次调用
   - cleaning_location_ID (str): "sinkbasin 1"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        ↓
    LLM 生成清洗代码
        ↓
    执行清洗动作
        ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第 4 次调用
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📍 任务：put_in(obj_ID, target_location_ID)
📦 变量：
   - obj_ID (str): "apple 1"
   - target_location_ID (str): "sidetable 1"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        ↓
    LLM 生成放置代码
        ↓
    执行放置动作
        ↓
    🎉 任务完成！
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 变量的生命周期

```
变量累积过程：

第 1 次调用后：
{
    instruction: "clean some apple...",
    observation: "You are in the middle...",
    cleaning_location_ID: "sinkbasin 1",      ← 新增
    target_location_ID: "sidetable 1",        ← 新增
    obj: "apple",                             ← 新增
    all_location_IDs: ["cabinet 4", ...]      ← 新增
}

第 2 次调用后：
{
    ... (所有上面的变量)
    location_ID: "cabinet 2",                 ← 新增
    obs: "You open the cabinet 2...",         ← 新增
    location_closed: True,                    ← 新增
    obj_ID: "apple 1"                         ← 新增 ⬅️ 关键！
}

第 3 次调用后：
{
    ... (所有上面的变量)
    obs: "You clean the apple 1..."           ← 更新
}

第 4 次调用后：
{
    ... (所有上面的变量)
    obs: "You put the apple 1..."             ← 更新
}
```

---

## 关键机制解析

### 1. 上下文提取机制

**问题**：为什么 LLM 每次只看到相关的变量，而不是所有变量？

**答案**：`get_variables()` 函数的智能提取

```python
def get_variables(executor: Executor, code: str) -> str:
    """
    从函数调用中提取参数变量

    例如：
    输入代码：find_and_take(obj, all_location_IDs)

    1. 解析 AST，找到函数调用
    2. 提取参数名：obj, all_location_IDs
    3. 从执行器中获取这些变量的值
    4. 格式化输出
    """

    # 示例输出：
    """
    - obj (str): apple
    - all_location_IDs (list[str]): ['cabinet 4', 'cabinet 3', ...]
    """
```

**作用**：
1. ✅ **减少上下文长度**：只传递相关信息
2. ✅ **提高准确性**：避免无关信息干扰
3. ✅ **降低成本**：减少 token 使用

### 2. 占位符识别机制

**问题**：ReCode 如何知道一个函数是占位符？

**答案**：通过 `NameError` 检测

```python
try:
    exec("obj_ID = find_and_take(obj, all_location_IDs)")
except NameError as e:
    # 错误信息：name 'find_and_take' is not defined

    # 检查是否是函数调用
    if "find_and_take(" in code:
        # 这是占位符！标记为 STUB
        return "NeedExpansion: `find_and_take` needs to be expanded."
```

**流程**：
1. 尝试执行代码
2. 捕获 `NameError`
3. 检查是否包含函数调用语法 `name(`
4. 判定为占位符，返回 `NeedExpansion`
5. 触发新的 LLM 调用

### 3. 提示词组装机制

**完整的提示词构建**：

```python
def _build_expand_prompt(self) -> str:
    # 1. 加载可用动作（环境特定，固定）
    actions = open(f"prompts/{env_name}/actions.txt").read()

    # 2. 加载示例（任务类型特定，固定）
    examples = open(f"fewshots/{env_name}/{task_type}.txt").read()

    # 3. 获取当前任务（动态）
    task = self.current_node.code

    # 4. 提取相关变量（动态）
    variables = get_variables(self.executor, task)

    # 5. 组装
    return EXPAND_PROMPT.format(
        available_actions=actions,
        examples=examples,
        task=task,
        variables=variables
    )
```

### 4. 示例的作用（Few-shot Learning）

**为什么需要示例？**

示例告诉 LLM：
1. **输入格式**：如何理解当前任务和变量
2. **输出格式**：如何使用 `<think>` 和 `<execute>` 标签
3. **决策模式**：什么时候分解，什么时候直接执行
4. **编码风格**：如何命名变量，如何使用 `run()`
5. **领域知识**：例如，清洗必须在 sinkbasin，打开必须在 take 之前

**示例的层次结构**：

```
高层任务示例：
  solve(instruction, observation)
  → 展开为 3-4 个子任务

中层任务示例：
  find_and_take(obj, all_location_IDs)
  → 展开为循环+具体动作

底层任务示例：
  clean_with(obj_ID, location_ID)
  → 直接使用 run() 执行
```

---

## 总结

### LLM 上下文变化的核心规律

```
1️⃣ 固定部分（不变）
   - 系统指令
   - 可用动作列表
   - 示例（同一任务类型）

2️⃣ 动态部分（每次都变）
   - 当前任务：从高层到底层
   - 可用变量：越来越具体

3️⃣ 变化趋势
   任务：solve → find_and_take → (循环+run) → clean_with → (run) → put_in → (run)
          ↓           ↓                           ↓                    ↓
   抽象度： 高  →      中            →           低      →           底层

4️⃣ 变量演化
   observation (原始文本)
     → all_location_IDs (提取的列表)
       → obj_ID (搜索得到的具体值)
         → 在后续任务中重复使用
```

### 为什么这样设计有效？

1. **渐进式具体化**：从抽象到具体，逐步细化
2. **上下文聚焦**：每次只关注相关信息
3. **知识复用**：示例教会 LLM 通用模式
4. **状态维护**：执行器保存所有变量，形成"记忆"
5. **动态反馈**：每次执行后的结果影响下一次决策

### 与传统方法的对比

```
传统方法：
  一次性生成完整计划 → 执行
  ❌ 无法根据中间结果调整
  ❌ 上下文过长
  ❌ 容易出错

ReCode 方法：
  多次递归调用 LLM → 边生成边执行
  ✅ 根据环境反馈调整
  ✅ 每次上下文聚焦
  ✅ 逐步验证正确性
```

---

## 附录：完整的 LLM 输入模板

```markdown
═══════════════════════════════════════════════════════════
完整的 LLM 输入模板
═══════════════════════════════════════════════════════════

You are the EXPAND step in the LLM Agent loop. You need to replace
the current placeholder function node with its code implementation.

Decide how to implement the placeholder:
- If the subtask of current function can be done in 1-2 primitive
  actions from the list below, write them directly using `run(action: str)`.
- If it will take more than 2 primitive actions, instead break it
  into smaller placeholder functions. Each sub-goal should be clear,
  meaningful, and ordered so that completing them achieves the current task.

All legal primitive actions are:
{available_actions}

And all of them should be used in the function `run(action: str) -> str`,
which returns an observation in string format.

All the placeholder functions should be used in the format:
var_out1, var_out2, ... = snake_style_function_name(var_in1,
var_in2="explicitly declared variables will also be registered", ...),
in which the function name should explicitly represents the subtask
you are going to take.

Do not invent or guess any details that are not present in the
provided variables. If essential information is missing or uncertain,
write a descriptive placeholder function that explicitly represents
the missing decision, to be expanded later.

In your response:
1. Start with a brief natural language explanation of how you will
   complete or break down the task, enclosed with <think> and </think>.
2. Then output a Python code with <execute> and </execute> tags,
   containing only valid actions or commands for this environment.
   Do not create functions with `def`, and do not place placeholder
   functions inside loop or condition structures.

---
Here are some examples to guide the style and format:
{examples}
(End of Examples)
---

The current function to expand is:
{task}

The variables you can use is:
{variables}

═══════════════════════════════════════════════════════════
```

---

**🎉 现在你完全理解 LLM 在 ReCode 中的上下文变化了！**

关键要点：
- ✅ LLM 输入 = 固定指令 + 动态上下文
- ✅ 上下文 = 当前任务 + 相关变量
- ✅ 变量逐步累积，从抽象到具体
- ✅ 每次调用都聚焦于当前子任务
