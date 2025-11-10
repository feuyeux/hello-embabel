# Embabel 框架深度原理分析

> - 测试案例: "Tell me a story about Tom and Jerry"
> - 执行时间: 2025-11-10 19:50:50 - 19:51:01 (总计 11秒)

---

## 目录

1. [框架概述](#1-框架概述)
2. [核心架构](#2-核心架构)
3. [真实调用链路](#3-真实调用链路)
4. [GOAP 规划原理](#4-goap-规划原理)
5. [性能分析](#5-性能分析)

---

## 1. 框架概述

### 1.1 Embabel 是什么

Embabel 是基于 Spring Boot 的 AI Agent 开发框架，核心特点：

- **GOAP (Goal-Oriented Action Planning)** - 目标导向的动作规划引擎
- **注解驱动** - 通过 `@Agent` 和 `@Action` 注解声明智能体和动作
- **动态规划** - 根据当前状态和目标自动规划执行路径
- **多人格 LLM** - 支持不同任务使用不同的 AI 人格

### 1.2 核心概念

**Agent (智能体)**:
```java
@Agent(description = "Generate a story based on user input and review it")
public class WriteAndReviewAgent {
    
    @Action
    Story craftStory(UserInput userInput, OperationContext context) {
        // 创作故事
    }
    
    @AchievesGoal(description = "The story has been crafted and reviewed")
    @Action
    ReviewedStory reviewStory(UserInput userInput, Story story, OperationContext context) {
        // 评审故事
    }
}
```

**World State (世界状态)**:
- 表示当前系统中所有对象的存在状态
- 例如: `{UserInput=TRUE, Story=FALSE, ReviewedStory=FALSE}`

**Action (动作)**:
- 方法参数 = 前置条件 (需要哪些对象)
- 返回类型 = 效果 (产生什么对象)
- `@AchievesGoal` 标记目标动作

---

## 2. 核心架构

### 2.1 整体架构图

```
用户输入
   ↓
ShellCommands (Spring Shell)
   ↓
Autonomy (Agent 选择与执行协调)
   ↓
Agent Selector (LLM 驱动的 Agent 选择)
   ↓
SimpleAgentProcess (进程管理)
   ↓
GOAP Planner (A* 规划算法)
   ↓
ActionRunner (Action 执行器)
   ↓
DefaultActionMethodManager (方法调用管理)
   ↓
Kotlin Reflection (反射调用)
   ↓
WriteAndReviewAgent (业务逻辑)
   ↓
OperationContext (AI 服务)
   ↓
ChatClientLlmOperations (LLM 客户端)
   ↓
Ollama (qwen2.5:latest)
```

### 2.2 关键组件

| 组件 | 职责 | 实现类 |
|------|------|--------|
| Shell | 用户交互入口 | ShellCommands |
| Autonomy | Agent 选择与执行协调 | Autonomy |
| Process | 进程生命周期管理 | SimpleAgentProcess |
| Planner | GOAP 规划算法 | GOAP Planner |
| ActionRunner | Action 执行与计时 | ActionRunner |
| ActionManager | 方法调用管理 | DefaultActionMethodManager |
| AIContext | AI 服务访问 | OperationContext |
| LLM Client | LLM 调用 | ChatClientLlmOperations |

---

## 3. 真实调用链路

### 3.1 完整时间线

基于真实日志的完整执行时间线:

| 时间 | 事件 | 耗时 |
|------|------|------|
| 19:50:50.068 | 用户输入: "Tell me a story about Tom and Jerry" | - |
| 19:50:50.068 | 开始 Agent 选择 | - |
| 19:50:55.061 | Agent 选择完成: WriteAndReviewAgent (1.0) | 4.99秒 |
| 19:50:55.085 | GOAP 规划完成: [craftStory → reviewStory] | 0.024秒 |
| 19:50:55.086 | 开始执行 craftStory | - |
| 19:50:58.243 | craftStory 完成 | 3.16秒 |
| 19:50:58.254 | GOAP 重规划完成: [reviewStory] | 0.011秒 |
| 19:50:58.254 | 开始执行 reviewStory | - |
| 19:51:01.434 | reviewStory 完成 | 3.18秒 |
| 19:51:01.443 | 目标达成，流程结束 | 总计: 11.38秒 |                            


### 3.2 完整调用堆栈

基于 `doc/stack.txt` 的 craftStory 调用堆栈:

```
10. WriteAndReviewAgent.craftStory()           ← 业务逻辑 (java:139)
     ↑
 9. Method.invoke()                            ← Java 反射
     ↑
 8. KCallableImpl.call()                       ← Kotlin 反射 (kt:107)
     ↑
 7. DefaultActionMethodManager.invokeActionMethod()  ← 方法管理
     ↑
 6. ActionRunner.execute()                     ← 执行器 (kt:50)
     ↑
 5. RetryTemplate.doExecute()                  ← 重试机制
     ↑
 4. AbstractAgentProcess.executeAction()       ← 进程执行
     ↑
 3. Autonomy.runAgent()                        ← 协调层
     ↑
 2. ShellCommands.execute()                    ← Shell 入口 (kt:322)
     ↑
 1. HelloJavaApplication.main()                ← 应用入口 (java:27)
```

**关键观察**:
- 从用户输入到业务逻辑经过 **10 层调用**
- 使用 **Kotlin 反射** 而非 Java 反射
- 包含 **Spring Retry** 机制
- **ActionRunner** 负责性能计时


### 3.3 详细执行流程

#### 阶段 1: Agent 选择 (4.99秒)

```
19:50:50.068  ShellCommands.execute()
              └─ 创建 UserInput(content="Tell me a story about Tom and Jerry",
                                timestamp=2025-11-10T11:50:50.068805Z)

19:50:50.068  Autonomy.chooseAndRunAgent()
              └─ Agent Selector 调用 LLM 分析用户意图
                 ├─ 创建 DummyInstance for RankingsResponse
                 ├─ LLM 调用 (459 prompt + 19 completion tokens)
                 └─ 返回: {"rankings":[{"name":"WriteAndReviewAgent", "confidence":1.0}]}

19:50:55.061  选中 WriteAndReviewAgent
              └─ 扫描 Agent 元数据:
                 ├─ craftStory: UserInput → Story
                 ├─ reviewStory: UserInput,Story → ReviewedStory (Goal)
                 └─ 构建 Action 依赖图
```

**Agent 元数据** (基于日志):
```
WriteAndReviewAgent:
  description: "Generate a story based on user input and review it"
  provider: com.embabel.template.agent
  version: 0.1.0-SNAPSHOT
  
  goals:
    - reviewStory (目标 Action)
      preconditions:
        hasRun_reviewStory: FALSE
        it:UserInput: TRUE
        it:Story: TRUE
        it:ReviewedStory: FALSE
      postconditions:
        hasRun_reviewStory: TRUE
        it:ReviewedStory: TRUE
  
  actions:
    - craftStory
      preconditions:
        hasRun_craftStory: FALSE
        it:UserInput: TRUE
        it:Story: FALSE
      postconditions:
        hasRun_craftStory: TRUE
        it:Story: TRUE
    
    - reviewStory (同上)
```

#### 阶段 2: GOAP 规划 (0.024秒)

```
19:50:55.061  创建 Process: relaxed_moore
              └─ Blackboard ID: b986b939-16b3-4018-9176-b9cd4b1423a3

19:50:55.077  初始化世界状态:
              {
                it:UserInput: TRUE,
                it:Story: FALSE,
                it:ReviewedStory: FALSE,
                hasRun_craftStory: FALSE,
                hasRun_reviewStory: FALSE
              }

19:50:55.085  GOAP Planner 执行 A* 搜索:
              ├─ 目标: ReviewedStory=TRUE
              ├─ 分析可执行 Actions:
              │  ├─ craftStory: 前置条件满足 ✓ (UserInput=TRUE)
              │  └─ reviewStory: 前置条件不满足 ✗ (缺少 Story)
              └─ 生成计划: [craftStory → reviewStory]
                 cost: 0.0, netValue: 0.0
```

**日志输出**:
```
[relaxed_moore] Formulated plan:
  com.embabel.template.agent.WriteAndReviewAgent.craftStory ->
    com.embabel.template.agent.WriteAndReviewAgent.reviewStory
  goal: com.embabel.template.agent.WriteAndReviewAgent.reviewStory
  cost: 0.0
  netValue: 0.0
```

#### 阶段 3: 执行 craftStory (3.16秒)

```
19:50:55.085  SimpleAgentProcess.formulateAndExecutePlan()
              └─ 执行 Action: craftStory

19:50:55.086  ActionRunner.execute()
              ├─ 延迟调度: PT0S (立即执行)
              └─ 输出类型: JvmType(className=WriteAndReviewAgent$Story)

19:50:55.086  DefaultActionMethodManager.invokeActionMethod()
              ├─ 解析输入: [UserInput(...)]
              └─ 使用 Kotlin Reflection 调用

19:50:55.086  WriteAndReviewAgent.craftStory() 开始执行
              ├─ 构建 Prompt:
              │  "Craft a short story in 100 words or less.
              │   The story should be engaging and imaginative.
              │   Use the user's input as inspiration if possible.
              │   
              │   # User input
              │   Tell me a story about Tom and Jerry"
              │
              ├─ 配置 LLM:
              │  ├─ Model: qwen2.5:latest
              │  ├─ Temperature: 0.7 (高温度，增加创意)
              │  └─ Persona: WRITER ("Creative Storyteller")
              │
              └─ 调用 ChatClientLlmOperations.createObject()

[LLM 调用中... 耗时 3.16秒]

19:50:58.243  LLM 返回 Story 对象:
              Story[text="Tom and Jerry found themselves in a magical circus 
                          where the acts were orchestrated by fairy tales. In the 
                          moonlit ring, Tom played the brave knight, chasing shadows 
                          that turned into fearsome dragons. Jerry, clever as ever, 
                          rode tiny unicorns, leaving trails of glittering stardust. 
                          Together, they faced enchanted forests and solved riddles 
                          to find a hidden treasure chest filled with stories, each 
                          one more fantastical than the last."]

19:50:58.243  绑定输出到 Blackboard:
              ├─ 键: "it"
              └─ 值: Story 对象

19:50:58.243  craftStory 执行完成
              └─ 耗时: PT3.157S
```

**世界状态更新**:
```
执行前: {UserInput=TRUE, Story=FALSE, ReviewedStory=FALSE, hasRun_craftStory=FALSE}
执行后: {UserInput=TRUE, Story=TRUE,  ReviewedStory=FALSE, hasRun_craftStory=TRUE}
```

**Blackboard 更新**:
```
id: 27283519-de42-4807-9fa2-d506bd916876
map:
  it = Story[text="In a bustling circus..."]
entries:
  - UserInput(content="Tell me a story about Tom and Jerry", ...)
  - Story[text="In a bustling circus..."]
```

#### 阶段 4: 动态重规划 (0.011秒)

```
19:50:58.243  检查当前状态:
              {
                it:UserInput: TRUE,
                it:Story: TRUE,          ← 新增
                it:ReviewedStory: FALSE,
                hasRun_craftStory: TRUE, ← 已执行
                hasRun_reviewStory: FALSE
              }

19:50:58.254  GOAP Planner 重新规划:
              ├─ 分析 reviewStory 前置条件:
              │  ├─ UserInput: TRUE ✓
              │  ├─ Story: TRUE ✓
              │  └─ hasRun_reviewStory: FALSE ✓
              ├─ 前置条件满足，可以执行
              └─ 生成新计划: [reviewStory]
                 cost: 0.0, netValue: 0.0
```

**日志输出**:
```
[relaxed_moore] Formulated plan:
  com.embabel.template.agent.WriteAndReviewAgent.reviewStory
  goal: com.embabel.template.agent.WriteAndReviewAgent.reviewStory
  cost: 0.0
  netValue: 0.0
from:
  hasRun_reviewStory: FALSE
  it:ReviewedStory: FALSE
  hasRun_craftStory: TRUE
  it:UserInput: TRUE
  it:Story: TRUE
```

#### 阶段 5: 执行 reviewStory (3.18秒)

```
19:50:58.254  SimpleAgentProcess.formulateAndExecutePlan()
              └─ 执行 Action: reviewStory

19:50:58.254  DefaultActionMethodManager.invokeActionMethod()
              ├─ 解析输入: [UserInput(...), Story[...]]
              └─ 使用 Kotlin Reflection 调用

19:50:58.254  WriteAndReviewAgent.reviewStory() 开始执行
              ├─ 构建 Prompt:
              │  "You will be given a short story to review.
              │   Review it in 100 words or less.
              │   Consider whether or not the story is engaging, imaginative, 
              │   and well-written.
              │   
              │   # Story
              │   Tom and Jerry found themselves in a magical circus...
              │   
              │   # User input that inspired the story
              │   Tell me a story about Tom and Jerry"
              │
              ├─ 配置 LLM:
              │  ├─ Model: qwen2.5:latest
              │  ├─ Temperature: 默认 (0.5)
              │  └─ Persona: REVIEWER ("New York Times Book Reviewer")
              │
              └─ 调用 ChatClientLlmOperations.generateText()

[LLM 调用中... 耗时 3.18秒]

19:51:01.434  LLM 返回评审文本:
              "This whimsical tale of Tom and Jerry in a magical circus is 
               both engaging and imaginative. The dynamic between the two 
               characters remains intact—Tom as the intrepid hero and Jerry 
               as the clever sidekick—while introducing a new, fairy-tale-
               infused setting that keeps the spirit of adventure alive. The 
               story's fantastical elements are woven seamlessly into familiar 
               scenarios, making it delightful for all ages. A perfect read 
               for anyone who cherishes the classic duo in a fresh, enchanting 
               setting."

19:51:01.434  创建 ReviewedStory 对象:
              ReviewedStory[
                story=Story[text="Tom and Jerry found themselves..."],
                review="This whimsical tale of Tom and Jerry...",
                reviewer=Persona(name="Media Book Review",
                                persona="New York Times Book Reviewer",
                                voice="Professional and insightful",
                                objective="Help guide readers toward good stories")
              ]

19:51:01.434  绑定输出到 Blackboard:
              ├─ 键: "it"
              └─ 值: ReviewedStory 对象

19:51:01.434  reviewStory 执行完成
              └─ 耗时: PT3.18S
```

**世界状态更新**:
```
执行前: {UserInput=TRUE, Story=TRUE, ReviewedStory=FALSE, hasRun_reviewStory=FALSE}
执行后: {UserInput=TRUE, Story=TRUE, ReviewedStory=TRUE,  hasRun_reviewStory=TRUE}
```

#### 阶段 6: 目标验证与完成 (0.009秒)

```
19:51:01.434  GOAP Planner 最终验证:
              ├─ 当前状态:
              │  {
              │    it:UserInput: TRUE,
              │    it:Story: TRUE,
              │    it:ReviewedStory: TRUE,  ← 目标对象存在
              │    hasRun_craftStory: TRUE,
              │    hasRun_reviewStory: TRUE
              │  }
              ├─ 目标状态: ReviewedStory=TRUE
              └─ 目标达成 ✓

19:51:01.443  Process 完成:
              ├─ 总耗时: PT6.373S (6.37秒)
              ├─ Actions 执行: 2 (craftStory, reviewStory)
              ├─ 规划次数: 2 (初始 + 1次重规划)
              └─ LLM 调用: 3次 (Agent选择 + 2个Action)
```

**最终 Blackboard**:
```
id: b986b939-16b3-4018-9176-b9cd4b1423a3
map:
  it = ReviewedStory[
    story=Story[text="Tom and Jerry found themselves in a magical circus..."],
    review="This whimsical tale of Tom and Jerry in a magical circus...",
    reviewer=Persona(name="Media Book Review", ...)
  ]
entries:
  - UserInput(content="Tell me a story about Tom and Jerry", ...)
  - Story[text="Tom and Jerry found themselves in a magical circus..."]
  - ReviewedStory[...]
```

#### 阶段 7: 结果格式化与输出

基于 `doc/stack3.txt` 的调用堆栈:

```
ShellCommands.runProcess()
└─ FormatProcessOutputKt.formatProcessOutput() ← formatProcessOutput.kt:41
   └─ ReviewedStory.getContent() ← WriteAndReviewAgent.java:74
      └─ String.format() 生成 Markdown 格式
```

**输出内容**:

```
You asked: UserInput(content=Tell me a story about Tom and Jerry, ...)

# Story
Tom and Jerry found themselves in a magical circus where the acts were 
orchestrated by fairy tales. In the moonlit ring, Tom played the brave 
knight, chasing shadows that turned into fearsome dragons. Jerry, clever 
as ever, rode tiny unicorns, leaving trails of glittering stardust. 
Together, they faced enchanted forests and solved riddles to find a 
hidden treasure chest filled with stories, each one more fantastical 
than the last.

# Review
This whimsical tale of Tom and Jerry in a magical circus is both engaging 
and imaginative. The dynamic between the two characters remains intact—Tom 
as the intrepid hero and Jerry as the clever sidekick—while introducing a 
new, fairy-tale-infused setting that keeps the spirit of adventure alive. 
The story's fantastical elements are woven seamlessly into familiar 
scenarios, making it delightful for all ages. A perfect read for anyone 
who cherishes the classic duo in a fresh, enchanting setting.

# Reviewer
Media Book Review, Monday, November 10, 2025

LLMs used: [qwen2.5:latest] across 3 calls
Prompt tokens: 459 (Agent选择) + 估算 (craftStory + reviewStory)
Completion tokens: 19 (Agent选择) + 估算 (craftStory + reviewStory)
Cost: $0.0000

Tool usage: (none)
```

---

## 4. GOAP 规划原理

### 4.1 核心算法

GOAP 使用 **A* 搜索算法** 进行规划:

```java
public List<ActionDefinition> plan(
    WorldState initial,      // 初始状态
    WorldState goal,         // 目标状态
    List<ActionDefinition> actions  // 可用动作
) {
    PriorityQueue<PlanNode> openSet = new PriorityQueue<>(
        Comparator.comparingInt(n -> n.estimatedCost(goal))
    );
    
    Set<WorldState> closedSet = new HashSet<>();
    PlanNode start = new PlanNode(initial, new ArrayList<>(), 0, null);
    openSet.add(start);
    
    while (!openSet.isEmpty()) {
        PlanNode current = openSet.poll();
        
        if (goalAchieved(current.state, goal)) {
            return current.plan;  // 返回计划
        }
        
        closedSet.add(current.state);
        
        for (ActionDefinition action : actions) {
            if (!action.canExecute(current.state)) {
                continue;  // 前置条件不满足
            }
            
            WorldState newState = action.apply(current.state, simulateResult(action));
            
            if (closedSet.contains(newState)) {
                continue;  // 已访问过
            }
            
            List<ActionDefinition> newPlan = new ArrayList<>(current.plan);
            newPlan.add(action);
            
            PlanNode newNode = new PlanNode(
                newState,
                newPlan,
                current.totalCost + action.cost,
                current
            );
            
            openSet.add(newNode);
        }
    }
    
    return null;  // 无法找到计划
}
```

### 4.2 规划过程可视化

基于真实案例的规划过程:

```
初始状态: {UserInput=T, Story=F, ReviewedStory=F}
目标状态: {ReviewedStory=T}

步骤 1: 分析可执行 Actions
  ├─ craftStory:
  │  ├─ 前置条件: UserInput=T ✓
  │  └─ 可执行: YES
  └─ reviewStory:
     ├─ 前置条件: UserInput=T ✓, Story=T ✗
     └─ 可执行: NO

步骤 2: 模拟执行 craftStory
  ├─ 执行前: {UserInput=T, Story=F, ReviewedStory=F}
  └─ 执行后: {UserInput=T, Story=T, ReviewedStory=F}

步骤 3: 继续搜索
  ├─ reviewStory:
  │  ├─ 前置条件: UserInput=T ✓, Story=T ✓
  │  └─ 可执行: YES
  └─ 模拟执行后: {UserInput=T, Story=T, ReviewedStory=T}

步骤 4: 目标检查
  ├─ 当前状态: ReviewedStory=T
  ├─ 目标状态: ReviewedStory=T
  └─ 目标达成 ✓

生成计划: [craftStory → reviewStory]
耗时: 0.008秒
```

### 4.3 动态重规划

每执行完一个 Action 后，框架会重新评估:

```
执行 craftStory 后:
  ├─ 更新世界状态: Story=TRUE
  ├─ 检查剩余计划: [reviewStory]
  ├─ 验证前置条件:
  │  ├─ UserInput: TRUE ✓
  │  └─ Story: TRUE ✓
  ├─ 前置条件满足，继续执行
  └─ 更新计划: [reviewStory]

执行 reviewStory 后:
  ├─ 更新世界状态: ReviewedStory=TRUE
  ├─ 检查目标: ReviewedStory=TRUE ✓
  └─ 目标达成，流程结束
```

**重规划触发条件**:
1. 当前计划为空
2. 下一个 Action 的前置条件不满足
3. 发现更优的执行路径

---

## 5. 性能分析

### 5.1 时间分解

基于真实日志的性能分析:

```
总执行时间: 11.38秒

详细分解:
  1. Agent 选择:        4.99秒  (43.8%)  ← 主要瓶颈
     └─ LLM 调用:       4.99秒  (459 prompt + 19 completion tokens)
  
  2. GOAP 初始规划:     0.024秒 (0.2%)
  
  3. craftStory 执行:   3.16秒  (27.8%)
     ├─ Prompt 构建:    <0.01秒
     ├─ LLM 调用:       ~3.15秒
     └─ 对象创建:       <0.01秒
  
  4. 状态更新:          0.011秒 (0.1%)
  
  5. GOAP 重规划:       0.011秒 (0.1%)
  
  6. reviewStory 执行:  3.18秒  (27.9%)
     ├─ Prompt 构建:    <0.01秒
     ├─ LLM 调用:       ~3.17秒
     └─ 对象创建:       <0.01秒
  
  7. 目标验证:          0.009秒 (0.1%)

瓶颈分析:
  - LLM 调用总计:      ~11.32秒 (99.5%)
    ├─ Agent 选择:     4.99秒   (43.8%)
    ├─ craftStory:     3.16秒   (27.8%)
    └─ reviewStory:    3.18秒   (27.9%)
  - 框架开销:          ~0.06秒  (0.5%)
```

### 5.2 Token 使用统计

```
Agent 选择:
  - Prompt tokens:     459
  - Completion tokens: 19
  - Total:             478

craftStory:
  - Prompt tokens:     估算 ~250
  - Completion tokens: 估算 ~90
  - Total:             估算 ~340

reviewStory:
  - Prompt tokens:     估算 ~200
  - Completion tokens: 估算 ~80
  - Total:             估算 ~280

总计 (估算):
  - Prompt tokens:     ~909
  - Completion tokens: ~189
  - Total tokens:      ~1098
  - Cost:              $0.0000
```

### 5.3 性能瓶颈

**主要瓶颈**:
1. **Agent 选择 LLM 调用**: 4.99秒 - 占总时间的 43.8%
2. **craftStory LLM 调用**: 3.16秒 - 占总时间的 27.8%
3. **reviewStory LLM 调用**: 3.18秒 - 占总时间的 27.9%
4. **框架开销**: 0.06秒 - 占总时间的 0.5% (可忽略)

**关键发现**:
- LLM 调用占据了 99.5% 的执行时间
- GOAP 规划非常高效 (0.024秒 + 0.011秒)
- 框架本身的开销极小，反射调用、状态管理等都非常快
- Agent 选择耗时最长，占了近一半时间

**优化建议**:
1. **优化 Agent 选择**:
   - 如果只有一个 Agent，可以跳过 LLM 选择
   - 使用更快的小模型进行 Agent 选择
   - 考虑基于规则的 Agent 选择策略

2. **使用更快的 LLM 模型**:
   - 考虑使用 qwen2.5:1.5b 等小模型
   - 或使用 qwen3:0.6b 进行快速响应

3. **启用响应缓存**:
   - 对相同输入缓存 LLM 响应
   - 避免重复调用

4. **并行执行**:
   - 如果有多个独立的 Action，可以考虑并行执行

### 5.4 框架开销分析

```
框架总开销: ~0.06秒 (0.5%)

分解:
  - Agent 扫描与注册:  <0.01秒
  - GOAP 规划:         0.035秒 (2次规划)
  - Action 调用开销:   <0.01秒
  - 状态管理:          <0.02秒
  - 目标验证:          0.009秒

结论:
  - GOAP 规划非常高效 (0.024秒 + 0.011秒)
  - 框架本身的反射调用开销可忽略不计
  - 状态管理和 Blackboard 操作都很快
  - 框架设计优秀，几乎没有性能损耗
  - 性能完全取决于 LLM 调用速度
```

