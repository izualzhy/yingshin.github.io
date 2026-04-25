---
title: "从 ADK 看 Google 眼里的 MCP&A2A&Skill"
date: 2026-04-25 10:18:16
tags: AI
---

我眼里的：
1. 许多平台原来开放了 OpenAPI，**MCP** 能够对接到现在的 AI 生态，调用门槛大幅降低。  
2. 当 Agent 多且杂(现实如此)，就会衍生出 Agent 之间的通信需求，**A2A** 像是在设想未来，姑且当做一个标准了解。    
3. **Skill** 通过渐进式披露解决了模型上下文的问题，本质上还是计算机常用的分而治之思想；同时 Skill 支持描述顺序执行/条件分支，避免了拖拉拽搭建工作流。  

现在从 ADK 的实现和使用方式，来看看其怎么看待这三个概念/技术点的。

## 1. MCP

分别是 Client Server 两个角度：
1. Client: 如何将 ADK tool(s) 转为 MCP Server
2. Server: 如何在 ADK agent 中接入其他 MCP Server

ADK tool(s) 暴露为 MCP Server，第一步使用`FunctionTool`，例如`FunctionTool(load_web_page)`，之后就都可以借助 MCP/FastMCP 完成了。

Agent 接入 MCP Server，支持 STDIO StreamableHTTP 。在实现上，通过传入`tools=[MCPSet(..), ]`完成。

`FunctionTool`是个很有用的封装，能够自动提取 name doc args 等信息。这些信息，MCP 和 Agent 都需要用到，因为**两者的相同点是都需要拿到这些信息，区别则是 MCP 给到 Agent，而 Agent 给到大模型**。

之前笔记定义 Agent 的例子里，传入的普通函数，实际是转为了`FunctionTool`
```python
async def _convert_tool_union_to_tools(
    tool_union: ToolUnion,
    ctx: ReadonlyContext,
    model: Union[str, BaseLlm],
    multiple_tools: bool = False,
) -> list[BaseTool]:
  ...

  if isinstance(tool_union, BaseTool):
    return [tool_union]
  if callable(tool_union):
    return [FunctionTool(func=tool_union)]
```

结合上面的`tools=[MCPSet(..), ]`方法，我们可以看到 ADK 里一个非常好的设计，那就是用 **ToolUnion 抽象了工具这一层**。

```python
ToolUnion: TypeAlias = Union[Callable, BaseTool, BaseToolset]
```

在 ADK 里，这些都是工具：
1. 普通函数 (Callable): 如 get_weather
2. 工具实例 (BaseTool): 如 FunctionTool 实例
3. 工具集 (BaseToolset): 如 MCPToolset 实例

区别是 MCPServer 自身的“工具”列表，需要连接后获取。

上述详细代码参见<sup>1<sup>

## 2. A2A

MCP 协议里，统一了 Tools Resources Prompts 等的调用方式，那如果想要在 Agent 里，跟另一个 Agent 通信呢？

A2A 协议想要先发占领的，就是这个领域。

回归到问题本身，多个 Agent 之间，为何不能本地通信？ADK 里是这么说的：

- Local Sub-Agents: These are agents that run within the same application process as your main agent. They are like internal modules or libraries, used to organize your code into logical, reusable components. Communication between a main agent and its local sub-agents is very fast because it happens directly in memory, without network overhead.
- Remote Agents (A2A): These are independent agents that run as separate services, communicating over a network. A2A defines the standard protocol for this communication.

简言之:
- 什么时候用本地模式：代码内部、延迟要求低的内部操作、共享内存、简单的 helper function    
- 什么时候用 A2A：想要做成微服务架构、第三方提供的 agent 服务、跨语言agent通信、控制格式  

注：有点像是古老的 本地函数还是远程调用 的区分方式😅

怎么做的？我一直没搞懂的两点在这里明确了：
1. 给每一个 agent 一个名片，包含了 name description 以及都有哪些能力  
2. Agent 通信实现用的 http 协议

具体的，基于 ADK 里普通 agent，定义`a2a_app = to_a2a(some_agent, ...)`，然后可以使用`uvicorn xxx:a2a_app --host --port `启动 HTTP 服务，作为 A2A Server；然后定义 `root_agent: RemoteA2aAgent`，在`agent-card`里指定 server AGENT_CARD_WELL_KNOWN_PATH(e.g. /.well-known/agent-card.json) 地址。

在`adk web`里使用`root_agent`，就可以达到调用其他 Agent 的目的了。

其中 agent card 形如：

```json
{
    "capabilities": {},
    "defaultInputModes": [
        "text/plain"
    ],
    "defaultOutputModes": [
        "text/plain"
    ],
    "description": "hello world agent that can roll a dice of 8 sides and check prime numbers.",
    "name": "hello_world_agent",
    "preferredTransport": "JSONRPC",
    "protocolVersion": "0.3.0",
    "skills": [
        {
            "description": "hello world agent that can roll a dice of 8 sides and check prime numbers. \n      I roll dice and answer questions about the outcome of the dice rolls.\n      I can roll dice of different sizes.\n      I can use multiple tools in parallel by calling functions in parallel(in one request and in one round).\n      It is ok to discuss previous dice roles, and comment on the dice rolls.\n      When I am asked to roll a die, I must call the roll_die tool with the number of sides. Be sure to pass in an integer. Do not pass in a string.\n      I should never roll a die on my own.\n      When checking prime numbers, call the check_prime tool with a list of integers. Be sure to pass in a list of integers. I should never pass in a string.\n      I should not check prime numbers before calling the tool.\n      When I am asked to roll a die and check prime numbers, I should always make the following two function calls:\n      1. I should first call the roll_die tool to get a roll. Wait for the function response before calling the check_prime tool.\n      2. After I get the function response from roll_die tool, I should call the check_prime tool with the roll_die result.\n        2.1 If user asks I to check primes based on previous rolls, make sure I include the previous rolls in the list.\n      3. When I respond, I must include the roll_die result from step 1.\n      I should always perform the previous 3 steps when asking for a roll and checking prime numbers.\n      I should not rely on the previous history on prime results.\n    ",
            "examples": [],
            "id": "hello_world_agent",
            "name": "model",
            "tags": [
                "llm"
            ]
        },
        {
            "description": "Roll a die and return the rolled result.\n\nArgs:\n  sides: The integer number of sides the die has.\n  tool_context: the tool context\nReturns:\n  An integer of the result of rolling the die.",
            "id": "hello_world_agent-roll_die",
            "name": "roll_die",
            "tags": [
                "llm",
                "tools"
            ]
        },
        {
            "description": "Check if a given list of numbers are prime.\n\nArgs:\n  nums: The list of numbers to check.\n\nReturns:\n  A str indicating which number is prime.",
            "id": "hello_world_agent-check_prime",
            "name": "check_prime",
            "tags": [
                "llm",
                "tools"
            ]
        }
    ],
    "supportsAuthenticatedExtendedCard": false,
    "url": "http://localhost:8001",
    "version": "0.0.1"
}
```

详细代码可以参考：adk-python/contributing/samples a2a_root a2a_basic  

## 3. Skill

Skill 的介绍里，是允许它逐步加载，以尽量减少对代理操作上下文窗口的影响。但其实 MCP 的 list_tool，Agent 里的 card，都是这个效果，不是吗？我觉得 Skill 更被人容易接受主要是三点：

1. 效果好：大模型的指令遵从性越来越强  
2. 门槛低：不需要懂代码，只要你逻辑性强、懂金字塔原理，就可以很好的将自身业务经验转化为 Skill
3. 易传播：一个目录，copy 过来就能用  

所以我主要研究了 ADK 的实现里是如何支持 Skill 的。

在 ADK 里使用 skill，需要在 tools 里指定：`tools=[SkillToolSet(...), ], `(注：`SkillToolSet MCPToolSet`都属于`BaseToolSet`，即提供了工具集的工具)

`SkillToolSet`里包含多个`Skill`，`Skill`定义为：

```python
class Skill(BaseModel):
  """Complete skill representation including frontmatter, instructions, and resources.

  A skill combines:
  - L1: Frontmatter for discovery (name, description).
  - L2: Instructions from SKILL.md body, loaded when skill is triggered.
  - L3: Resources including additional instructions, assets, and scripts,
  loaded as needed.

  Attributes:
      frontmatter: Parsed skill frontmatter from SKILL.md.
      instructions: L2 skill content: markdown instruction from SKILL.md body.
      resources: L3 skill content: additional instructions, assets, and scripts.
  """
  ...
```

即：

![](/assets/images/adk/part1-progressive-disclosure_1.original.png){:width="600"}

**L1 -> L2 -> L3 从抽象到具体，从全局到细节，构建了整体的 Skill.**  

当 Agent 里使用了 Skill，会读取 L1 的内容(`format_skills_as_xml`)，添加到给 LLM 的 prompt 里：

```python
@experimental(FeatureName.SKILL_TOOLSET)
class SkillToolset(BaseToolset):
  """A toolset for managing and interacting with agent skills."""
  async def process_llm_request(
      self, *, tool_context: ToolContext, llm_request: LlmRequest
  ) -> None:
    """Processes the outgoing LLM request to include available skills."""
    skills = self._list_skills()
    skills_xml = prompt.format_skills_as_xml(skills)
    instructions = []
    instructions.append(_DEFAULT_SKILL_SYSTEM_INSTRUCTION)
    instructions.append(skills_xml)
    llm_request.append_instructions(instructions)
```

`_DEFAULT_SKILL_SYSTEM_INSTRUCTION`这段提示词，告诉了模型：
1. `load_skill`用于读取 full instruction
2. `load_skill_resource`用读取 directory (e.g., `references/*`, `assets/*`, `scripts/*`)
3. `run_skill_script`用于执行脚本  

对应到`SkillToolset`本身提供的工具：
```python
    self._tools = [
        ListSkillsTool(self),       # 给出所有的 Skills L1 内容
        LoadSkillTool(self),        # 指定 Skill 名字，读取 SKILL.md instructions 
        LoadSkillResourceTool(self) # 指定 Skill 名字，读取 resource 
        RunSkillScriptTool(self),   # 指定 Skill 名字，执行 script 
    ]
```

注：`format_skills_as_xml`用的 XML 格式，之前看到有说法是 XML 的标签结构对 LLM 更友好，更容易被理解和解析，感觉不大可信。

接下来就是进入到 Agent Loop:
1. LLM prompt 会包含包含两部分 instruction：agent 自身的，以及上述加入的系统的  
2. 工具列表会包含两部分：固定传递`SkillToolSet`自身的四个工具，以及随着 LLM 回复里动态加入的工具  
3. Skill.md 也是 instruction，但是实际是作为工具调用结果返回   
4. 如果 Skill 里用到了 Agent 自身的工具，ADK 里使用 state 上下文来记录了该信息  

其他跟普通 Agent 实现区别不大，该调用嘛调用嘛。

详细代码可以参考：adk-python/contributing/samples skills_agent

## 4. 总结

如果基于当前的现状看，我觉得三者没什么太大联系，相同点是都是 AI 进程里非常火的概念。

1. MCP: 无论是 HTTP 还是本地调用，预期都能够产生明确的调用结果、返回
2. A2A: 当你需要远程调用一个 Agent 时
3. Skill: 用文字作为入口，在需要确定性的时候，则执行代码

但是随着发展，你很难说当 MCP 成为事实标准后，会不会又在 resource prompt tool 之上，提出 agent system module 等各种概念，那尽管 A2A 有先发优势，实际遵循的可能也是 MCP 协议。

而 Agent 之间确实有通信需求吗？我想 A2A 如果有足够动力，对于框架使用者来说，完全可以像 aka 那样，使用者无需感知是本地还是远程，通过协议本身`local://weather_agent..` `cluster:svc_name://weather_agent..`来区分，将复杂度留给框架而不是用户自己区分。

Agent 开发平台，试图通过拖拉拽搭建出来确定逻辑的工作流(比如 Dify Coze 试图做的那样)，但是拖拉拽这项低代码技术一直都不温不火（高不成低不就，会写代码的觉得不灵活，不会写代码的觉得太费劲，纯个人见解），而 Skill 则支持了普通用户用文字描述这个流程。

但是 Skill 装多了，一上来也会用掉很多 token ，所以也会有说法是自己根据情况加载，这种时候，使用能通信的 agent 集群，就是一个更好的选择了吧？而 Agent 之间的通信、何时调用 Skill、MCP、其他 Agent，就是 Agent 框架里要做的事情了。比如你不能在 Skill 里写，在 xxx 情况下连接这个 MCP，连接这个 Agent.

整体上，他们都有的特点：：
1. 多方受益，符合经济学
2. 分而治之，符合技术思想
3. 使用简单，符合用户习惯

## 5. 参考资料

1. [agent_using_fs_mcp_server](https://github.com/izualzhy/AI-Systems/tree/main/google_adk/agent_using_fs_mcp_server)
2. [developers-guide-to-building-adk-agents-with-skills](https://developers.googleblog.com/developers-guide-to-building-adk-agents-with-skills/)
3. [5 Agent Skill Design Patterns Every ADK Developer Should Know](https://lavinigam.com/posts/adk-skill-design-patterns/?utm_source=lavinigam&utm_medium=reddit&utm_campaign=adk-skill-design-patterns)
