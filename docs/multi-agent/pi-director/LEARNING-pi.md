# PI Director 是什么？

PI Director 是一个基于 LLM Agent Loop（大模型智能体循环）的课堂调度机制。 它负责协调 AI 教师、AI 同学等角色，决定当前轮次调用哪个 Agent、使用什么工具，以及何时结束回答。OpenMAIC 当前版本的课堂对话采用 Pi 运行机制。


## 1. 核心工作流程

用户提问 / 课堂事件

PI Director Loop

分析上下文，决定下一步行动

调用 Agent

教师 / 同学

调用工具

读取课件 / 课堂操作

获取执行结果 → 继续循环或结束

## 2. 它使用什么技术？

| 技术                 | 作用                      |
| ------------------ | ----------------------- |
| Agent Loop         | 循环执行思考、调用工具、接收结果        |
| LLM + Tool Calling | 由模型根据上下文选择工具或行动         |
| 多 Agent 协作         | 让不同角色参与课堂对话             |
| 流式事件               | 将 Agent 输出、操作和状态实时发送给前端 |

OpenMAIC 的 `lib/chat/pi/` 实现服务端 Pi Director Loop，通过 `call_agent` 将课堂轮次交给相应 Agent。


## 3. 与 LangGraph 有什么区别？

| 对比      | PI Director        | LangGraph   |
| ------- | ------------------ | ----------- |
| 核心思路    | Agent 循环驱动任务       | 图结构驱动任务     |
| 流程控制    | 循环中根据结果继续行动        | 节点、边、条件分支   |
| 灵活性     | 适合动态决定下一步行动        | 适合显式定义复杂工作流 |
| 是否是模型   | 不是                 | 不是          |
| 是否是独立算法 | 更偏 Agent 运行模式与调度机制 | 工作流编排框架     |

注意：PI Director 不等于 Pi 模型，也不是某种专门的推理模型。

## 4. 学习时应该怎么理解？

把它理解成一个由大模型驱动的调度循环即可：

```
while (!finished) {
  const decision = await director(context);

  const result = await executeAgentOrTool(decision);

  context = updateContext(context, result);

  finished = shouldFinish(context);
}
```

这是用于理解原理的伪代码，不是 OpenMAIC 的实际源码。

学习 OpenMAIC 时，建议重点看：

- `lib/chat/pi/`：Pi Agent 循环和课堂调度。
- `lib/action/`：课堂动作的执行机制。
- `lib/playback/`：课堂播放和交互状态机。

一句话总结：PI Director 负责决定“下一步做什么”，Agent 负责执行具体任务，Tool 负责完成具体操作。