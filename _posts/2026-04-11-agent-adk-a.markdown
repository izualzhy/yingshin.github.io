---
title: "Google ADK: 又一款 Agent 框架？"
date: 2026-04-11 11:09:51
tags: AI
---

## 1. Google ADK：轻量级智能体框架介绍

Agent Development Kit (ADK) 是 Google 去年推出的一款智能体框架，从我的测试情况看<sup>1</sup>，还是比较好用的。

相比 LangChain，主要的优点是**轻量级的设计及简洁代码、对 Agent 效果迭代过程的实用设计**。前者方便快速原型开发，后者则关注到了 Agent 效果而不是仅仅搭建出来。

LangChain 初始印象是功能强大，比如支持了不同的 LLM、不同的向量库等。但是实际使用下来，发现这种支持的代价就是深层次的抽象、反射。而对于简单的业务场景，比如代码定位、个性化的功能开发等，反而会更加复杂耗时。

之前也简单的看过[CAMEL 和 AutoGEN](https://izualzhy.cn/llm-paper-read-camel-autogen)，方便性上不如 ADK.

更合理的智能体框架形式，或许可以参考 Apache Flink 的做法：由框架实现主体逻辑，而非主体部分比如对接各类存储的工作则放在 connector 中实现，使用时按需加载。Flink 之所以能做到这一点，是因为它基于足够丰富的应用场景，抽象出了流处理的各个阶段及其定义。

这一点上，ADK 与其他 Agent 框架一样，也面临着类似的问题：框架提供了很多佐料和食材，但真正好吃的菜却少之又少。这就像当前好用的 Agent 寥寥无几，也因此难以支撑框架迈向新的阶段。

## 2. ADK核心功能与组件

### 2.1. 例子

ADK 支持通过 `adk run` (Terminal) 或 `adk web` (可视化页面) 快速运行项目。

```python
def get_weather(city: str) -> dict:
    """Returns the weather in a specified city."""
    return {"status": "success", "city": city, "weather": "晴，31 度"}

weather_agent = Agent(
    model=ark_ds_model,
    name='root_agent',
    instruction="你是一个穿衣助手，如果需要知道城市的天气，可以调用 get_weather",
    tools=[get_weather],
    before_model_callback=test_before_call_llm,
    after_model_callback=test_after_call_llm
)
```

执行 `adk run` 后的关键交互逻辑如下：

```bash
[user]: 北京今天穿什么衣服
--- 发送给模型的请求 (Callback 截取) ---
llm_request：... prompt: "你是一个穿衣助手... 北京今天穿什么衣服"
--- 模型返回的响应 ---
llm_response：function_call(name='get_weather', args={'city': '北京'})

[root_agent]: 根据北京今天的天气情况（晴，31度），我为您提供以下穿衣建议：
1. 上衣：轻薄透气的短袖T恤...
2. 下装：短裤或薄款休闲裤...
```

`adk web` 提供了更直观的可视化能力，**极大降低了原型验证的门槛**。

![adk_sample_weather_agent_web](/assets/images/adk/adk_sample_weather_agent_web.png)

### 2.2. Components

ADK核心组件架构图：

![adk-components](/assets/images/adk/adk-components.png)

ADK 通过 `Runner` 驱动 `Event` 循环。组件分三类：

- 执行核心：
  - Agent : 单个和多个，多个有 `SequentialAgent ParallelAgent LoopAgent` 三种形式。  
  - Planning : 分解任务，分步执行的能力
  - Models : 借助了 Litellm<sup>2</sup>打平各厂商 LLM 的调用  
  - Event : 描述用户、模型、工具行为的基础单元  
  - Runner : 基础引擎  
- 状态：
  - Session Management : 包括 Session State Events ，例子使用的`adk web`驱动执行，如果要自定义执行过程，就需要引入`SessionService`来管理 Session.
  - Memory : 长期记忆
- 扩展能力：
  - Tools : 提供 API 执行能力, 例如代码里的`get_weather`. 
  - Artifact Management (`Artifact`) : 该组件提供了读写文件/二进制的能力  
  - Callbacks : 在可观测、安全审查、prompt 注入等场景都是比较有用的，ADK 里特殊的一点是还可以用来读写状态。
  - Code Execution : generate and execute code ，后续笔记细说    

## 3. ADK的设计优势与技术特点

### 3.1. 存储服务的轻量化

Session 对应一轮聊天里的多次会话，生命周期为：

![event-loop](/assets/images/adk/event-loop.png)

**Session 通过 SessionService 管理**，支持 `InMemory`、`Database` (MySQL 5 表模式) 及 `VertexAi` 后端
- `InMemorySessionService`: 内存型
- `VertexAiSessionService`: 使用 Google Cloud Vertex AI infrastructure  
- `DatabaseSessionService`: 使用关系数据库存储，后端使用 mysql 的话，会用到 5 张表：
    - adk_internal_metadata: ['key', 'value']，系统内置
    - app_states: ['app_name', 'state', 'update_time']，记录 app 级别的 state
    - user_states: ['app_name', 'user_id', 'state', 'update_time']，记录用户的 state
    - events: ['id', 'app_name', 'user_id', 'session_id', 'invocation_id', 'timestamp', 'event_data']，记录每个 session 里的 event
    - sessions: ['app_name', 'user_id', 'id', 'state', 'create_time', 'update_time']，记录每个 session   

**MemoryService and Artifact:** `MemoryService` 处理长期记忆，`ArtifactService` 处理二进制流。这种解耦设计确保了存储介质的可插拔性，也避免了上下文污染。轻量级的设计确保了留白以让用户充分自定义，当然功能也对应弱一些。

Event 用来驱动整个过程，例如提问、回答、记忆压缩、function_call，都是不同的 Event，当我们不使用`adk web`而是自行管理 SessionService 时，就需要处理 Event 了。抽象出来 Event，使得我们只用关注如何处理 Event，而不用关心 EventLoop 的实现。比如 A2A 需要将 agent expose 为 RestfulAPI 服务，关注的也是 A2A协议格式 与 Event 之间如何相互转换。

### 3.2. 状态作用域 (State) 的设计

State 本质上是一个支持多级作用域的键值对。ADK 通过前缀设计，巧妙地解决了不同维度的状态共享问题：

1.  无前缀 (Session State): 仅限当前会话。
2.  user: (User State): 绑定用户 ID，跨会话共享。
3.  app: (App State): 全局共享。
4.  temp: (Temporary State): 仅限单次调用（Invocation）。

主要有三处可以读写 state：  
1. `Agent`的`output_key`，整个 Agent 的输出会存储到`state[$output_key]`   
2. `Agent`的`instruction`, 通过`{state_key}`的形式读取    
3. 各类 Callback 里，通过`Context.state`读写    
这三类方法，读写的都是一个对象，使用起来非常方便。

**实测效果对比：**

当 `user1` 沟通后记录状态，随后 `user2` 介入：

**表：app_states (全局状态，被 user2 覆盖)**  

`user1`触发更新：

| app_name | state (JSON) | update_time |
| :--- | :--- | :--- |
| state_app | `{"color": "红色", "food": "火锅"}` | 2026-04-11 18:12 |

`user2`触发更新：

| app_name | state (JSON) | update_time |
| :--- | :--- | :--- |
| state_app | `{"color": "蓝色", "food": "烧烤"}` | 2026-04-11 18:13 |

**表：user_states (用户状态，互相隔离)**  

| app_name | user_id | state (JSON) | update_time |
| :--- | :--- | :--- | :--- |
| state_app | user1 | `{"color": "红色", "food": "火锅"}` | 2026-04-11 18:12 |
| state_app | user2 | `{"color": "蓝色", "food": "烧烤"}` | 2026-04-11 18:13 |

**技术局限：**
目前 `instruction` 暂不支持直接通过 `{user:state_key}` 引用带前缀的状态，开发者需在 Callback 中手动提取。针对这个问题提交了 Issue：[google/adk-python #5298](https://github.com/google/adk-python/issues/5298)，看看是 bug 还是设计如此。

## 4. 一些收获

1. **AgentTeam VS Workflow:** 在 Agent Team 与 Workflow 之间，边界在于“确定性”：Workflow 提供路径约束，但是内部单个 Agent 依然保持生成式的“不确定性”, 当前有三类简洁的 flow 形式
  - 顺序执行：![sequential-agent](/assets/images/adk/sequential-agent.png){:width="300"}
  - 并发执行：![parallel-agent](/assets/images/adk/parallel-agent.png){:width="300"}
  - 循环执行：![loop-agent.png](/assets/images/adk/loop-agent.png){:width="300"}
  - 2.0 将引入更复杂的编排逻辑: ![workflow-design](/assets/images/adk/workflow-design.svg){:width="300"}
2. **Human-In-The-Loop (HITL):** ADK 的简洁性使得 HITL 可以通过 Tool 机制轻松实现，无需引入过重的框架特质。不足的是在`adk web`页面上无法嵌入这个能力。
3. **格式控制:** 可通过设置 `config.response_schema` 配合强校验，确保 Agent 输出符合业务定义的 JSON 结构。
4. **数据量:** 按照`user_states`表结构的设计，假定用户规模10w, app 100个, 每个用户记录10个state，那么该表的行数为: 100000 * 100 * 10 = 1亿，放到数据库里就需要考虑分库分表或者换其他存储，可见好用的基础设施仍然是必须的。  
5. **rewind**: 支持了可以回溯到某个问答阶段重新开始，比如 agent 可能给了你两个选项，最开始你选择了 A，结果几轮交互后发现不符合预期，就可以通过 rewind 回到最开始选择的阶段，重新从 B 开始。基于存储了全部 Event 的前提，实现这个不算很难，但是也是一个很有意思的功能尝试，像是我们自己在决策过程中不断地推演和回退，直到最优解。

## 5. 参考资料

1. [my adk sample code](https://github.com/izualzhy/AI-Systems/tree/main/google_adk)  
2. [litellm](https://github.com/BerriAI/litellm.git) - ADK 底层打平各厂商模型调用的基石
