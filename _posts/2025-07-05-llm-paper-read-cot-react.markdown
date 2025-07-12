---
title: "LLM Paper&Practice：从 CoT 到 ReAct"
date: 2025-07-05 11:42:17
tags: read ai
---

本文是我在阅读《大模型应用开发 动手做AI Agent》时，对书中推荐的几篇核心论文的理解与总结，主要聚焦于**思维链（Chain-of-Thought, CoT）**和 **ReAct** 这两种范式。

学术论文往往较为抽象，本文尽量做到理论与实践相结合，通过阅读论文和代码实践，来验证和剖析这些技术背后的思想，并分享我在此过程中的一些思考与困惑。

# 1. 思维链（Chain-of-Thought, CoT）

教会模型“一步一步想”。

## 1.1. 什么是 CoT？

思维链（Chain-of-Thought）是一种提示（Prompting）技术，它通过在提示中加入一些“逐步思考”的范例（Few-shot），来引导 LLM 在回答问题时，模仿范例，输出一个详细的、循序渐进的推理过程，并最终给出答案。

这个过程就像我们人类解决复杂问题时，会把问题分解成若干个小步骤，然后逐一解决。CoT 的核心思想就是将这种“思考过程”显式地展示出来。

详细论文参考：[Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)

## 1.2. 为什么需要 CoT？

标准的 LLM 在处理复杂问题，尤其是数学应用题、逻辑推理题时，很容易因为“一步到位”的思考模式而得出错误结论。它们往往只关注表面的关联，而缺乏深度的逻辑演绎。

CoT 的出现，恰好弥补了这一点。它强迫模型放慢脚步，关注过程而非仅仅是结果，从而显著提升了在推理任务上的准确性。

注意论文是在 2022年1月 发表的，之所以笔记标题叫做炒冷饭，原因也在于此。虽然只隔了三年时间，但是大模型的发展迅速，有些例子已经不适用。之前说“一日不见如隔三秋”，放在 AI 领域真是太合适不过，三年已过，如同上个世纪。

## 1.3. 实践

论文中这个被广泛引用的图例，直观地展示了 CoT 的作用：

![cot_figure_1](/assets/images/ai-paper/cot_figure_1.png)

我使用代码进行了实际测试：

```python
def testCOT(llm: ChatOpenAI):
    # Standard Prompting
    question_standard = """Q: Roger has 5 tennis balls. He buys 2 more cans of
tennis balls. Each can has 3 tennis balls. How many
tennis balls does he have now?
A: The answer is 11.
Q: The cafeteria had 23 apples. If they used 20 to
make lunch and bought 6 more, how many apples
do they have?"""
    print('-'*32 + ' Standard Prompting ' + '-'*32)
    print(invokeLLM(llm, question_standard))

    # Chain-of-Thought Prompting
    question_cot = """"Q: Roger has 5 tennis balls. He buys 2 more cans of
tennis balls. Each can has 3 tennis balls. How many
tennis balls does he have now?
A: Roger started with 5 balls. 2 cans of 3 tennis balls
each is 6 tennis balls. 5 + 6 = 11. The answer is 11.
Q: The cafeteria had 23 apples. If they used 20 to
make lunch and bought 6 more, how many apples
do they have?"""
    print('-'*32 + ' CoT Prompting ' + '-'*32)
    print(invokeLLM(llm, question_cot))
```

这是用 doubaoseed1.6 某次输出的效果：

```
-------------------------------- Standard Prompting --------------------------------
The cafeteria starts with 23 apples. They use 20, so subtract 20: 23 - 20 = 3. Then they buy 6 more, so add 6: 3 + 6 = 9. The answer is 9.
-------------------------------- CoT Prompting --------------------------------
A: The cafeteria started with 23 apples. They used 20 apples, so 23 - 20 = 3 apples left. Then they bought 6 more, so 3 + 6 = 9. The answer is 9.
```

doubao1.5visionlite 某次输出的效果：

```
-------------------------------- Standard Prompting --------------------------------
First, find out how many apples are left after using some:
The cafeteria had 23 apples and used 20, so the number of apples left is \(23 - 20= 3\).
Then, they bought 6 more apples. So the total number of apples now is \(3 + 6 = 9\).
The answer is 9.
-------------------------------- CoT Prompting --------------------------------
The cafeteria started with 23 apples. They used 20 apples, so they had 23 - 20 = 3 apples left. Then they bought 6 more apples. 3 + 6 = 9. The answer is 9.
```

用 deepseek-r1 某次输出的效果：
```
-------------------------------- Standard Prompting --------------------------------

The cafeteria started with 23 apples.  
They used 20 apples to make lunch, which means they have 23 - 20 = 3 apples left.  
Then, they bought 6 more apples, so they have 3 + 6 = 9 apples now.  

Alternatively, the net change is -20 (used) + 6 (bought) = -14, so 23 + (-14) = 9.  
Regardless of the order of operations, the result is consistent: 23 - 20 + 6 = 9, or 23 + 6 - 20 = 9.  

Thus, the cafeteria now has 9 apples.

\boxed{9}
-------------------------------- CoT Prompting --------------------------------

The cafeteria started with 23 apples.  
They used 20 apples for lunch, so they had 23 - 20 = 3 apples left.  
Then they bought 6 more apples, so 3 + 6 = 9.  
The answer is 9.
```

可以看到用现在的大模型，回答其实都是准确的。
{:.warning}

我尝试多次调用，以及输入其他问题，例如：
1. Mary’s father has five daughters: Nana, Nene, Nini, Nono. What is the name of the fifth daughter?(为了引导模型逐步推理，加入这一句"Let's think step by step.")
2. Mike plays ping pong for 40 minutes. In the first 20 minutes, he scores 4 points. In the second 20 minutes, he scores 25% more points. How many total points did he score?

即使使用 Standard Prompting 而不是 Chain-of-Thought Prompting ，模型在输出时不仅展现了推理能力，而且也能够输出正确答案。

每次调用结果不同，如果在 Prompt 里尝试引导模型推理，符合预期的概率确实更大一些。

<p class="success">
    我的理解是大模型的效果就像是一个 V 字型，左边是训练生成模型，右边是查询模型，两边都会影响推理效果：<br/>
1. <strong>模型自身已进化</strong>: 现在的很多大模型，在左边就已经使用更精确的数据、调优解决了我测试的这些 query，因此即使不使用 COT ，效果也非常好了。<br/>
2. <strong>CoT 提升稳定性</strong>: 为了更大概率的拿到想要的输出，还是应当尽量使用 COT<br/>
3. <strong>CoT 的历史定位</strong>: 相比后续出现的策略，COT 最为基础和直接。因为 COT 只使用了 Prompt 来提升效果，因此也是 Prompt Engine 的一种体现。(注：我查了下似乎提出 PE 只比 COT 早了半年)<br/>
</p>

COT 的本质是要引导大模型的推理过程，现在看其实已经是比较普遍的思想了。

现在想想，这也非常符合直觉。就像我们自己计算出错后，会下意识地告诉自己“再仔细想想”。“多想想，按部就班地推演一遍，就会少犯错”——这个简单的道理，在大模型领域被赋予了一个专门的术语：CoT。

*相关代码放在 <https://github.com/izualzhy/AI-Systems/blob/main/misc/read_paper_cot.py>*

# 2. ReAct

从“思考”到“行动”的进化。

## 2.1. 什么是 ReAct？

**ReAct** 范式可以看作是 CoT 的一个强大演进。它不仅包含了 CoT 的“推理”（Reasoning）能力，更引入了“行动”（Acting）的概念。

ReAct 的核心机制是一个 `Thought -> Action -> Observation` 的循环：

1.  **Thought (思考):** 模型根据当前任务和已有信息，分析现状，并规划出下一步需要做什么。
2.  **Action (行动):** 模型决定执行一个具体的“行动”。这个行动通常是调用外部工具（API），例如进行网络搜索、查询数据库、执行代码等。
3.  **Observation (观察):** 模型获取“行动”返回的结果。这个结果会作为新的信息，融入到下一轮的“思考”中。

通过这个循环，LLM 能够与外部世界进行交互，获取实时、准确的信息，从而克服其自身知识库静态、可能过时的缺点。

详细论文参考：[REACT: SYNERGIZING REASONING AND ACTING IN LANGUAGE MODELS](https://arxiv.org/abs/2210.03629)

## 2.2. 为什么需要 ReAct？

CoT 极大地增强了模型的推理能力，但它的推理过程是“封闭”的，完全依赖于模型内部的知识。如果遇到以下场景，CoT 就会力不从心：

*   **知识过时：** LLM 的知识截止于其训练日期。
*   **需要精确计算：** LLM 在数学计算上并不可靠。
*   **需要与外部服务交互：** 例如订票、发邮件等。

ReAct 通过引入“行动”，将 LLM 从一个“封闭大脑”变成了一个能够主动探索和获取信息的“智能代理”（Agent），极大地拓展了其应用边界。

![react_figure_1](/assets/images/ai-paper/react_figure_1.png)

这套“思考-行动-观察”的循环，正是 ReAct Agent 的基本工作模式。

## 2.3. 实践

那么，这篇学术论文的思想在工程实践中是如何应用的呢？主要体现在两个方面：

1.  **精心设计的 Prompt**：构建一个能引导模型输出“Thought, Action, Action Input”等结构化文本的提示。
2.  **外部控制循环**：在代码中实现一个循环，用于解析模型输出、调用相应工具、并将工具结果返回给模型，直到任务完成。

我们用 LangChain 框架来直观感受一下：
```python
def get_current_weather(location: str):
    print(f"\n** [函数被调用] location = {location} **\n")
    return f"{location} 今天天气晴，25°C."

tools = [
    Tool.from_function(
        func=get_current_weather,
        name="get_current_weather",
        description="获取城市当前天气",
        handle_parsing_errors=True
    )
]

# 初始化 Agent
# llm = getDoubaoSeed16()
llm = getDoubao15VisionLite()
# llm = getDeepSeekR1()

def v1():
    # llm = getDeepSeekR1()
    agent = initialize_agent(tools,
                             llm,
                             agent_type=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
                             # agent_type="openai-tools",
                             verbose=True,
                             temperature=0,
                             handle_parsing_errors=True)

    def run():
        print(agent.agent.llm_chain.prompt.template)
        # 用户提问
        result = agent.invoke("请问上海的天气怎么样？")
        print(result)
    
    run()
```

上述代码输出：
```
Answer the following questions as best you can. You have access to the following tools:

get_current_weather(location: str) - 获取城市当前天气

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [get_current_weather]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought:{agent_scratchpad}


> Entering new AgentExecutor chain...
Thought: 需要获取上海当前的天气信息，调用 get_current_weather 工具来实现
Action: get_current_weather
Action Input: 上海
** [函数被调用] location = 上海 **


Observation: 上海 今天天气晴，25°C.
Thought:I now know the final answer
Final Answer: 上海 今天天气晴，25°C.

> Finished chain.
```

可以看到函数被调用，且返回了预期的最终答案。

### 2.3.1. Prompt

`ZERO_SHOT_REACT_DESCRIPTION` 这个 Agent 内置的 Prompt 模板，完美诠释了 ReAct 的思想：

```
Answer the following questions as best you can. You have access to the following tools:

get_current_weather(location: str) - 获取城市当前天气

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [get_current_weather]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought:{agent_scratchpad}
```
这个模板通过清晰的指令和格式要求，引导模型遵循 `Thought -> Action -> Observation` 的路径进行思考和输出。

### 2.3.2. 循环

```python
class AgentExecutor(Chain):

    def _call(
        self,
        inputs: dict[str, str],
        run_manager: Optional[CallbackManagerForChainRun] = None,
    ) -> dict[str, Any]:
        """Run text through and get agent response."""
        # Construct a mapping of tool name to tool for easy lookup
        name_to_tool_map = {tool.name: tool for tool in self.tools}
        # We construct a mapping from each tool to a color, used for logging.
        color_mapping = get_color_mapping(
            [tool.name for tool in self.tools], excluded_colors=["green", "red"]
        )
        intermediate_steps: list[tuple[AgentAction, str]] = []
        # Let's start tracking the number of iterations and time elapsed
        iterations = 0
        time_elapsed = 0.0
        start_time = time.time()
        # We now enter the agent loop (until it returns something).
        while self._should_continue(iterations, time_elapsed):
            next_step_output = self._take_next_step(
                name_to_tool_map,
                color_mapping,
                inputs,
                intermediate_steps,
                run_manager=run_manager,
            )
            if isinstance(next_step_output, AgentFinish):
                return self._return(
                    next_step_output, intermediate_steps, run_manager=run_manager
                )

            intermediate_steps.extend(next_step_output)
            if len(next_step_output) == 1:
                next_step_action = next_step_output[0]
                # See if tool should return directly
                tool_return = self._get_tool_return(next_step_action)
                if tool_return is not None:
                    return self._return(
                        tool_return, intermediate_steps, run_manager=run_manager
                    )
            iterations += 1
            time_elapsed = time.time() - start_time
        output = self._action_agent.return_stopped_response(
            self.early_stopping_method, intermediate_steps, **inputs
        )
        return self._return(output, intermediate_steps, run_manager=run_manager)
```

循环的过程在上述代码里，遵循`调用大模型 -> 解析大模型返回结果(action/final answer) -> 执行 -> 根据执行结果继续调用...`.

`_should_continue`里传入了迭代次数、消耗时间，避免陷入循环一直不返回。

解析返回结果：

```python
class MRKLOutputParser(AgentOutputParser):

    def parse(self, text: str) -> Union[AgentAction, AgentFinish]:
        """Parse the output from the agent into
        an AgentAction or AgentFinish object.

        Args:
            text: The text to parse.

        Returns:
            An AgentAction or AgentFinish object.

        Raises:
            OutputParserException: If the output could not be parsed.
        """
        includes_answer = FINAL_ANSWER_ACTION in text
        regex = (
            r"Action\s*\d*\s*:[\s]*(.*?)[\s]*Action\s*\d*\s*Input\s*\d*\s*:[\s]*(.*)"
        )
        action_match = re.search(regex, text, re.DOTALL)
        if action_match and includes_answer:
            ...
```

执行工具的调用栈：
```
AgentExecutor._take_next_step -> 
AgentExecutor._iter_next_step ->
AgentExecutor._perform_agent_action ->
BaseTool.run ->
Tool._run ->
自定义函数
```

由于大模型的框架都在快速迭代(换句话说不成熟😄)，就不深入展开代码了。理解概念吧。-_-||

## 2.4. 实践中的困惑

理论很丰满，但现实很骨感。实际上，我第一次运行上述代码时，花费了数小时也未能成功。

我发现，代码能否跑通，**极度依赖所选择的 LLM**。当我使用 `Doubao-Seed-1.6` 或 `DeepSeek-R1` 时，遇到了各种问题：
1.  模型返回结果格式错乱，比如同时输出了 `Final Answer` 和 `Action`，导致框架无法判断是否应结束循环。
2.  模型“自作主张”，不调用我定义的 `get_current_weather` 工具，反而直接输出一段它自己“知道”的关于上海天气的陈旧信息。

直到我换成了 `Doubao-1.5-vision-lite` 模型，一切才顺利运行。这引人深思：难道模型越高级，反而越不听话，越不遵守 Prompt 的指令了？

<p class="success">
    这让我产生了更深层次的理解：<br/>
1. <strong>ReAct 的本质</strong>：大模型本质是概率性的，无法保证100%准确。而现实世界需要确定性。ReAct 的出现，就是为了解决这个��盾：通过引入“工具”这个确定性组件，来弥补大模型的短板。而 Prompt 就是我们让模型学会使用工具的“说明书”。<br/>
2. <strong>脆弱的控制链</strong>：目前的 Agent 实现对 `模型 + Prompt` 的组合高度敏感。更换一个模型，或者稍微修改一下 Prompt，都可能导致整个控制链条的崩溃。这暴露了当前 Agent 技术的脆弱性。<br/>
3. <strong>农场里的科学家</strong>：我们现在通过精心设计的 Prompt 来控制大模型的行为，其原理更像是通过大量实验观察到的经验规律，而非精确的理论指导。这让我感觉，我们就像一群“农场鸡舍里的科学家”，试图理解并引导一群我们并未完全掌握其内在机理的“鸡”。<br/>
4. <strong>工具的演进</strong>：ReAct 引入工具，最初可能只是为了获取更准确、更实时的信息。但这个思想一旦被打开，我们的想象力便不再局限于此，而是希望用 Agent 去完成更复杂、更主动的任务，这催生了当前百花齐放的 Agent ��术浪潮。
</p>
*注：相关代码在 <https://github.com/izualzhy/AI-Systems/blob/main/misc/read_paper_react.py>*

# 3. 总结：从思想到行动的演进

回顾这两篇里程碑式的论文，我们可以看到一条清晰的技术演进路径：

1.  **Standard Prompting**：直接问答，完全依赖模型自身知识。
2.  **Chain-of-Thought (CoT)**：通过引导模型思考中间步骤，提升复杂推理的可靠性。这是**内部思维**的优化。
3.  **ReAct**：将 CoT 的“思考”与“行动”（使用工具）相结合，让模型能与外部世界交互，从“思考者”进化为“行动者”。这是**思维与实践**的结合。

这个过程，是从一个封闭的“知识库”到一个能够进行“研究”和“实践”的初级 Agent 的演变。我在实践中遇到的问题，或许也是当前 Agent 开发领域正在努力解决的核心问题：
**如何在给予模型强大推理能力的同时，确保它能可靠、可控地遵循我们设计的流程和规则，在需要时放下“身段”，谦虚地使用工具**
