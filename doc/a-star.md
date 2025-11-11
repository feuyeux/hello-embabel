# AStarGoapPlanner.planToGoalFrom 方法详细分析

## 概述

`AStarGoapPlanner` 实现了基于 A* 算法的目标导向行动规划（GOAP - Goal-Oriented Action Planning）系统。该系统通过寻找最优的行动序列，将初始世界状态转换为满足目标的状态，同时最小化总成本。

## 核心概念

### 领域模型

```mermaid
classDiagram
    class GoapWorldState {
        +Map~String, ConditionDetermination~ state
        +isAchievable(action) boolean
    }
    
    class GoapAction {
        +String name
        +Map~String, ConditionDetermination~ preconditions
        +Map~String, ConditionDetermination~ effects
        +Double cost
        +Double value
        +isAchievable(state) boolean
    }
    
    class GoapGoal {
        +String name
        +Map~String, ConditionDetermination~ preconditions
        +Double value
        +isAchievable(state) boolean
    }
    
    class GoapPlan {
        +List~Action~ actions
        +Goal goal
        +GoapWorldState worldState
    }
    
    class ConditionDetermination {
        <<enumeration>>
        TRUE
        FALSE
        UNKNOWN
    }
    
    GoapWorldState --> ConditionDetermination
    GoapAction --> ConditionDetermination
    GoapGoal --> ConditionDetermination
    GoapPlan --> GoapAction
    GoapPlan --> GoapGoal
```

### A* 算法关键要素

- **g-score**: 从起始状态到当前状态的实际累积成本
- **h-score**: 从当前状态到目标状态的启发式估计成本
- **f-score**: g-score + h-score，表示通过当前状态到达目标的总估计成本

## planToGoalFrom 方法执行流程

### 主流程图

```mermaid
flowchart TD
    Start([开始: planToGoalFrom]) --> CheckGoal{目标已满足?}
    CheckGoal -->|是| ReturnEmpty[返回空计划]
    CheckGoal -->|否| CheckReachable{目标可达?}
    
    CheckReachable -->|否| ReturnNull[返回 null]
    CheckReachable -->|是| InitStructures[初始化数据结构]
    
    InitStructures --> InitDetails[初始化:<br/>- openList 优先队列<br/>- gScores 成本映射<br/>- cameFrom 路径记录<br/>- closedSet 已处理集合]
    
    InitDetails --> AddStart[将起始状态加入 openList<br/>gScore = 0.0]
    
    AddStart --> LoopCheck{openList 非空<br/>且未超过最大迭代?}
    
    LoopCheck -->|否| CheckBest{找到目标状态?}
    LoopCheck -->|是| PollNode[从 openList 取出<br/>f-score 最小的节点]
    
    PollNode --> CheckBetter{当前节点成本<br/>≥ 最佳目标成本?}
    CheckBetter -->|是| LoopCheck
    CheckBetter -->|否| CheckClosed{节点已处理?}
    
    CheckClosed -->|是| LoopCheck
    CheckClosed -->|否| MarkClosed[标记为已处理]
    
    MarkClosed --> IsGoal{是目标状态?}
    
    IsGoal -->|是| UpdateBest[更新最佳目标节点]
    UpdateBest --> LoopCheck
    
    IsGoal -->|否| SortActions[按前置条件数量<br/>降序排序动作]
    
    SortActions --> ForEachAction[遍历每个动作]
    
    ForEachAction --> CheckAchievable{动作可执行?}
    CheckAchievable -->|否| NextAction[下一个动作]
    CheckAchievable -->|是| ApplyAction[应用动作<br/>计算新状态]
    
    ApplyAction --> CheckSame{新状态 = 当前状态?}
    CheckSame -->|是| NextAction
    CheckSame -->|否| CalcTentative[计算 tentativeGScore<br/>= gScore + action.cost]
    
    CalcTentative --> CheckExpensive{成本 ≥ 最佳目标成本?}
    CheckExpensive -->|是| NextAction
    CheckExpensive -->|否| CheckBetterPath{找到更好路径?}
    
    CheckBetterPath -->|否| NextAction
    CheckBetterPath -->|是| RecordPath[记录路径:<br/>- 更新 cameFrom<br/>- 更新 gScores]
    
    RecordPath --> CheckInClosed{在 closedSet 中?}
    CheckInClosed -->|否| AddToOpen[加入 openList]
    CheckInClosed -->|是| ReopenNode[从 closedSet 移除<br/>重新加入 openList]
    
    AddToOpen --> NextAction
    ReopenNode --> NextAction
    NextAction --> MoreActions{还有动作?}
    
    MoreActions -->|是| ForEachAction
    MoreActions -->|否| LoopCheck
    
    CheckBest -->|否| ReturnNull
    CheckBest -->|是| Reconstruct[重建路径]
    
    Reconstruct --> BackwardOpt[反向规划优化]
    BackwardOpt --> ForwardOpt[正向规划优化]
    ForwardOpt --> ReturnPlan[返回优化后的计划]
    
    ReturnEmpty --> End([结束])
    ReturnNull --> End
    ReturnPlan --> End
```

### 详细步骤说明

#### 1. 快速检查阶段

```mermaid
flowchart LR
    A[输入状态] --> B{目标已满足?}
    B -->|是| C[返回空计划]
    B -->|否| D{目标可达性检查}
    D -->|不可达| E[返回 null]
    D -->|可能可达| F[继续 A* 搜索]
```

**代码实现**:
```kotlin
// 快速检查：如果目标已满足，返回空计划
if (goal.isAchievable(startState)) {
    return GoapPlan(emptyList(), goal, worldState = startState)
}

// 早期可达性检查，避免对不可达目标进行昂贵的 A* 搜索
if (!isGoalReachable(startState, actions, goal)) {
    return null
}
```

**目标可达性检查原理**:
- 构建所有动作可产生的效果集合
- 检查目标的每个前置条件
- 如果某个前置条件在起始状态中未满足，且没有任何动作能产生该效果，则目标不可达

#### 2. 数据结构初始化

```mermaid
graph TD
    A[初始化阶段] --> B[openList: PriorityQueue<br/>按 f-score 排序]
    A --> C[gScores: Map<br/>状态 → 最佳已知成本]
    A --> D[cameFrom: Map<br/>状态 → 前驱状态和动作]
    A --> E[closedSet: Set<br/>已完全评估的状态]
    A --> F[bestGoalNode: SearchNode?<br/>最佳目标节点]
```

**代码实现**:
```kotlin
val openList = PriorityQueue<SearchNode>()
val gScores = mutableMapOf<GoapWorldState, Double>().withDefault { Double.MAX_VALUE }
val cameFrom = mutableMapOf<GoapWorldState, Pair<GoapWorldState, GoapAction?>>()
val closedSet = mutableSetOf<GoapWorldState>()

gScores[startState] = 0.0
openList.add(SearchNode(startState, 0.0, heuristic(startState, goal)))
```

#### 3. A* 主循环

```mermaid
sequenceDiagram
    participant OL as openList
    participant CS as closedSet
    participant GS as gScores
    participant CF as cameFrom
    
    loop 直到 openList 为空或达到最大迭代次数
        OL->>OL: poll() 取出 f-score 最小节点
        
        alt 节点已在 closedSet
            OL->>OL: 跳过此节点
        else 节点未处理
            OL->>CS: 标记为已处理
            
            alt 是目标状态
                OL->>OL: 更新最佳目标节点
            else 非目标状态
                loop 遍历所有可执行动作
                    OL->>OL: 应用动作，计算新状态
                    OL->>GS: 计算 tentativeGScore
                    
                    alt 找到更好路径
                        GS->>GS: 更新 gScores
                        CF->>CF: 记录路径
                        
                        alt 新状态在 closedSet
                            CS->>CS: 移除（重新打开）
                            CS->>OL: 加入 openList
                        else 新状态不在 closedSet
                            OL->>OL: 加入 openList
                        end
                    end
                end
            end
        end
    end
```

**关键优化点**:

1. **动作排序**: 按前置条件数量降序排序，优先考虑更具体的动作
```kotlin
val sortedActions = actions.sortedByDescending { it.preconditions.size }
```

2. **状态变化检查**: 跳过不改变状态的动作，防止循环
```kotlin
if (nextState == current.state) continue
```

3. **成本剪枝**: 如果路径成本已超过最佳目标成本，跳过
```kotlin
if (bestGoalNode != null && tentativeGScore >= bestGoalScore) {
    continue
}
```

4. **重新打开节点**: 如果找到到达已关闭节点的更好路径，重新打开它
```kotlin
if (nextState !in closedSet) {
    openList.add(SearchNode(nextState, tentativeGScore, heuristic(nextState, goal)))
} else {
    closedSet.remove(nextState)
    openList.add(SearchNode(nextState, tentativeGScore, heuristic(nextState, goal)))
}
```

#### 4. 启发式函数

```mermaid
graph LR
    A[当前状态] --> B[计算未满足的<br/>目标条件数量]
    B --> C[返回 h-score]
    
    style B fill:#e1f5ff
```

**代码实现**:
```kotlin
private fun heuristic(state: GoapWorldState, goal: GoapGoal): Double {
    return goal.preconditions.count { (key, value) -> 
        state.state[key] != value 
    }.toDouble()
}
```

**特性**:
- **可采纳性**: 永不高估实际成本，确保 A* 找到最优路径
- **简单高效**: 仅计数未满足条件，计算复杂度 O(n)

#### 5. 路径重建

```mermaid
flowchart LR
    A[目标状态] --> B[查找 cameFrom]
    B --> C[获取前驱状态和动作]
    C --> D[将动作加入列表]
    D --> E{到达起始状态?}
    E -->|否| B
    E -->|是| F[反转动作列表]
    F --> G[返回路径]
```

**代码实现**:
```kotlin
private fun reconstructPath(
    cameFrom: Map<GoapWorldState, Pair<GoapWorldState, GoapAction?>>,
    goalState: GoapWorldState
): List<GoapAction> {
    val actions = mutableListOf<GoapAction>()
    var currentState = goalState
    
    while (currentState in cameFrom) {
        val (previousState, action) = cameFrom[currentState]!!
        if (action != null) {
            actions.add(action)
        }
        currentState = previousState
    }
    
    return actions.reversed()
}
```

#### 6. 计划优化

```mermaid
flowchart TD
    A[原始计划] --> B[反向规划优化]
    B --> C[正向规划优化]
    C --> D[最终优化计划]
    
    subgraph 反向优化
        B1[从目标开始] --> B2[识别必要动作]
        B2 --> B3[移除不贡献的动作]
    end
    
    subgraph 正向优化
        C1[从起始状态模拟] --> C2[验证每个动作]
        C2 --> C3[移除无进展动作]
    end
```

##### 反向规划优化

**原理**: 从目标反向工作，只保留对实现目标有贡献的动作

```mermaid
flowchart TD
    Start[开始: 目标条件] --> Init[targetConditions = goal.preconditions]
    Init --> Loop[从计划末尾向前遍历]
    
    Loop --> CheckAction{动作是否建立<br/>所需条件?}
    CheckAction -->|是| Keep[保留动作]
    Keep --> RemoveCond[从 targetConditions<br/>移除已满足条件]
    RemoveCond --> AddPrec[添加动作的<br/>前置条件到 targetConditions]
    AddPrec --> Next[下一个动作]
    
    CheckAction -->|否| Skip[跳过动作]
    Skip --> Next
    
    Next --> MoreActions{还有动作?}
    MoreActions -->|是| Loop
    MoreActions -->|否| Reverse[反转保留的动作列表]
    Reverse --> End[返回优化计划]
```

**代码实现**:
```kotlin
private fun backwardPlanningOptimization(
    plan: List<GoapAction>,
    startState: GoapWorldState,
    goal: GoapGoal
): List<GoapAction> {
    if (plan.isEmpty()) return plan
    
    val targetConditions = goal.preconditions.toMutableMap()
    val keptActions = mutableListOf<GoapAction>()
    
    for (action in plan.reversed()) {
        var isNecessary = false
        
        for ((key, value) in action.effects) {
            if (targetConditions[key] == value) {
                isNecessary = true
                targetConditions.remove(key)
                action.preconditions.forEach { (precKey, precValue) ->
                    targetConditions[precKey] = precValue
                }
            }
        }
        
        if (isNecessary) {
            keptActions.add(action)
        }
    }
    
    return keptActions.reversed()
}
```

##### 正向规划优化

**原理**: 模拟计划执行，移除不产生目标进展的动作

```mermaid
flowchart TD
    Start[开始: 起始状态] --> Init[currentState = startState]
    Init --> Loop[遍历计划中的动作]
    
    Loop --> CheckAchievable{动作可执行?}
    CheckAchievable -->|否| Skip[跳过动作]
    CheckAchievable -->|是| Apply[应用动作]
    
    Apply --> CheckProgress{产生目标进展?}
    CheckProgress -->|是| Keep[保留动作<br/>更新 currentState]
    CheckProgress -->|否| Skip
    
    Keep --> Next[下一个动作]
    Skip --> Next
    
    Next --> MoreActions{还有动作?}
    MoreActions -->|是| Loop
    MoreActions -->|否| Verify[验证最终状态]
    
    Verify --> GoalAchieved{目标达成?}
    GoalAchieved -->|是| Return[返回优化计划]
    GoalAchieved -->|否| Fallback[返回原始计划]
```

**进展判断条件**:
```kotlin
val progressMade = nextState != currentState &&
    action.effects.any { (key, value) ->
        goal.preconditions.containsKey(key) &&
        currentState.state[key] != goal.preconditions[key] &&
        (value == goal.preconditions[key] || key !in nextState.state)
    }
```

## Sample 示例分析

### WriteAndReviewAgent 工作流程

```mermaid
flowchart TD
    Start[用户输入] --> Goal[目标: reviewStory<br/>故事已创作并被书评人评审]
    
    Goal --> Check1{Story 存在?}
    Check1 -->|否| Action1[执行 craftStory 动作]
    Check1 -->|是| Check2{Review 存在?}
    
    Action1 --> Effect1[效果: Story = 已创作]
    Effect1 --> Check2
    
    Check2 -->|否| Action2[执行 reviewStory 动作]
    Check2 -->|是| Complete[目标完成]
    
    Action2 --> Effect2[效果: ReviewedStory = 已评审]
    Effect2 --> Complete
    
    subgraph GOAP 规划
        P1[初始状态:<br/>Story = 不存在<br/>Review = 不存在]
        P2[目标状态:<br/>ReviewedStory = 已完成]
        P3[A* 搜索最优路径]
        
        P1 --> P3
        P2 --> P3
    end
```

### 状态转换示例

假设用户输入: "Write a story about a brave knight"

#### 初始世界状态
```kotlin
GoapWorldState(
    state = hashMapOf(
        "userInput.exists" to TRUE,
        "story.exists" to FALSE,
        "review.exists" to FALSE
    )
)
```

#### 可用动作

**动作 1: craftStory**
```kotlin
GoapAction(
    name = "craftStory",
    preconditions = mapOf(
        "userInput.exists" to TRUE
    ),
    effects = mapOf(
        "story.exists" to TRUE
    ),
    cost = 1.0
)
```

**动作 2: reviewStory**
```kotlin
GoapAction(
    name = "reviewStory",
    preconditions = mapOf(
        "userInput.exists" to TRUE,
        "story.exists" to TRUE
    ),
    effects = mapOf(
        "review.exists" to TRUE,
        "reviewedStory.complete" to TRUE
    ),
    cost = 1.0
)
```

#### 目标
```kotlin
GoapGoal(
    name = "writeAndReviewStory",
    preconditions = mapOf(
        "reviewedStory.complete" to TRUE
    )
)
```

#### A* 搜索过程

```mermaid
graph TD
    S0["状态 0 (起始)<br/>userInput: TRUE<br/>story: FALSE<br/>review: FALSE<br/>g=0, h=2, f=2"] 
    
    S1["状态 1<br/>userInput: TRUE<br/>story: TRUE<br/>review: FALSE<br/>g=1, h=1, f=2"]
    
    S2["状态 2 (目标)<br/>userInput: TRUE<br/>story: TRUE<br/>review: TRUE<br/>reviewedStory: TRUE<br/>g=2, h=0, f=2"]
    
    S0 -->|craftStory<br/>cost=1| S1
    S1 -->|reviewStory<br/>cost=1| S2
    
    style S0 fill:#e1f5ff
    style S2 fill:#c8e6c9
```

**搜索步骤**:

1. **迭代 1**: 
   - 从 openList 取出状态 0 (f=2)
   - 尝试 craftStory 动作
   - 生成状态 1，加入 openList

2. **迭代 2**:
   - 从 openList 取出状态 1 (f=2)
   - 尝试 reviewStory 动作
   - 生成状态 2（目标状态）
   - 更新 bestGoalNode

3. **路径重建**:
   - 从状态 2 回溯到状态 0
   - 路径: [craftStory, reviewStory]

4. **优化**:
   - 反向优化: 两个动作都对目标有贡献，保留
   - 正向优化: 两个动作都产生进展，保留
   - 最终计划: [craftStory, reviewStory]

### 执行结果

```mermaid
sequenceDiagram
    participant U as 用户
    participant P as 规划器
    participant A as Agent
    participant LLM as LLM
    
    U->>P: 输入: "Write a story about a brave knight"
    P->>P: 执行 A* 搜索
    P->>A: 计划: [craftStory, reviewStory]
    
    A->>LLM: craftStory(userInput)
    LLM->>A: Story 对象
    
    A->>LLM: reviewStory(userInput, story)
    LLM->>A: ReviewedStory 对象
    
    A->>U: 返回: 故事 + 评审
```

## 算法复杂度分析

### 时间复杂度

- **最坏情况**: O(b^d)
  - b: 分支因子（每个状态的平均可执行动作数）
  - d: 解决方案深度
  
- **实际性能**: 由于启发式函数和剪枝优化，通常远好于最坏情况

### 空间复杂度

- **O(|S|)**: 其中 |S| 是探索的状态数
- 主要存储:
  - openList: O(|S|)
  - gScores: O(|S|)
  - cameFrom: O(|S|)
  - closedSet: O(|S|)

## 关键优化技术总结

1. **早期可达性检查**: 避免对明显不可达目标的搜索
2. **动作排序**: 优先考虑更具体的动作
3. **成本剪枝**: 跳过成本已超过最佳解的路径
4. **状态重新打开**: 允许找到更好路径时重新评估已关闭状态
5. **双重计划优化**: 反向和正向优化相结合
6. **迭代限制**: 防止无限循环

## 算法保证

1. **完备性**: 如果存在解决方案，A* 将找到它
2. **最优性**: 由于使用可采纳启发式函数，A* 保证找到最低成本路径
3. **效率**: 启发式函数引导搜索朝向目标，减少探索的状态数

## 总结

`AStarGoapPlanner.planToGoalFrom` 方法实现了一个高效、优化的 GOAP 规划系统：

- 使用 A* 算法确保找到最优行动序列
- 通过多层优化减少不必要的动作
- 提供早期失败检测避免无效搜索
- 在 Sample 示例中，能够自动规划从用户输入到完成故事评审的完整流程

该实现在理论正确性和实际性能之间取得了良好平衡，适用于复杂的 AI Agent 规划场景。

----

## A* search algorithm

A* (pronounced "A-star") is a graph traversal and pathfinding algorithm that is used in many fields of computer science due to its completeness, optimality, and optimal efficiency. Given a weighted graph, a source node and a goal node, the algorithm finds the shortest path (with respect to the given weights) from source to goal.

One major practical drawback is its ${\displaystyle O(b^{d})}$ space complexity where 
- $d$ is the depth of the shallowest solution (the length of the shortest path from the source node to any given goal node) 
- $b$ is the branching factor (the maximum number of successors for any given state), as it stores all generated nodes in memory. 

Thus, in practical travel-routing systems, it is generally outperformed by algorithms that can pre-process the graph to attain better performance, as well as by memory-bounded approaches; however, A* is still the best solution in many cases.

Peter Hart, Nils Nilsson and Bertram Raphael of Stanford Research Institute (now SRI International) first published the algorithm in 1968. It can be seen as an extension of Dijkstra's algorithm. A* achieves better performance by using heuristics to guide its search.

Compared to Dijkstra's algorithm, the A* algorithm only finds the shortest path from a specified source to a specified goal, and not the shortest-path tree from a specified source to all possible goals. This is a necessary trade-off for using a specific-goal-directed heuristic. For Dijkstra's algorithm, since the entire shortest-path tree is generated, every node is a goal, and there can be no specific-goal-directed heuristic.


A* is an informed search algorithm, or a best-first search, meaning that it is formulated in terms of weighted graphs: starting from a specific starting node of a graph, it aims to find a path to the given goal node having the smallest cost (least distance travelled, shortest time, etc.). It does this by maintaining a tree of paths originating at the start node and extending those paths one edge at a time until the goal node is reached.

At each iteration of its main loop, A* needs to determine which of its paths to extend. It does so based on the cost of the path and an estimate of the cost required to extend the path all the way to the goal. Specifically, A* selects the path that minimizes

$$
{\displaystyle f(n)=g(n)+h(n)}
$$

where 

- $n$ is the next node on the path
- $g(n)$ is the cost of the path from the start node to n
- $h(n)$ is a heuristic function that estimates the cost of the cheapest path from n to the goal. 

The heuristic function is problem-specific. If the heuristic function is admissible – meaning that it never overestimates the actual cost to get to the goal – A* is guaranteed to return a least-cost path from start to goal.


Dijkstra's algorithm 戴克斯特拉算法

https://en.wikipedia.org/wiki/A*_search_algorithm

