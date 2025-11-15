# #LangGraph流式处理** **  
下面我给你一份 ****工程级、可直接上手的《LangGraph 流式处理（Streaming）最佳实践指南》****。 重点是：****不是告诉你 LangGraph 能流式，而是让你会写、会调、会用在真实 RAG / Agent / 多节点工作流中。****  
LangGraph 的流式能力其实比 LangChain 强得多，因为它天然支持：  
* 节点级别的 streaming  
* 多节点连续 streaming  
* 人机交互 + streaming  
* 工具调用 + streaming  
* 中断 + resume + streaming  
* 完整的 token-by-token 输出  
下面我用最清晰的方式讲透。  
  
## 🚀** 一、LangGraph 流式处理的本质**  
一句话：  
**LangGraph 的流式输出不是只流 LLM token，而是流“事件（Events）”，每个事件代表节点执行过程中的一步。**  
**LangGraph 的流式输出不是只流 LLM token，而是流“事件（Events）”，每个事件代表节点执行过程中的一步。**  
事件包括：  
* LLM 的 token 输出  
* 某个节点开始执行  
* 某个节点结束执行  
* 工具返回结果  
* 中断（需要用户输入）  
* 状态更新（state mutation）  
* 整个工作流结束  
所以，LangGraph 的 streaming 是 ****真正的工作流 streaming****，不是“仅流模型输出”。  
  
## 🧱** 二、关键 API（必须掌握）**  
## ✔** app.stream(input, config)**  
****执行图 + 流式事件输出****  
## ✔** app.astream_events(input, config)**  
****更底层的事件级流式接口（建议用于复杂 agent）****  
## ✔** app.send(input, config)**  
****用于用户事件恢复（HITL）时，也支持 streaming****  
  
## 🧪** 三、最小可运行的流式 Demo（一个节点）**  
下面示例可以在线性输出 DeepSeek/Llama3 token：  
```
from langgraph.graph import StateGraph, END
from typing import List, TypedDict
from langchain_core.messages import HumanMessage, AIMessage
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="deepseek-chat", streaming=True)

class State(TypedDict):
    messages: List

def chat_node(state: State):
    response = llm.stream(state["messages"])
    for token in response:
        yield token
    return {"messages": state["messages"] + [token]}

graph = StateGraph(State)
graph.add_node("chat", chat_node)
graph.set_entry_point("chat")
graph.add_edge("chat", END)

app = graph.compile()

```
## ****流式运行：****  
```
for event in app.stream(
    {"messages": [HumanMessage(content="Hello!")]},
    config={"thread_id": "1"}
):
    print(event)

```
你会看到：  
```
{'node': 'chat', 'event': 'start'}
token: "你"
token: "好"
token: "，"
token: "有"
token: "什"
token: "么"
...
{'node': 'chat', 'event': 'end'}

```
这是 ****完整节点级 streaming****（很多框架做不到）。  
  
## 🧱** 四、推荐使用：astream_events()（更强、更专业）**  
这个接口提供 全事件 流：  
```
async for event in app.astream_events(
    input,
    config
):
    print(event)

```
它会给你：  

| 事件类型         | 用途        |
| ------------ | --------- |
| on_start     | 节点开始      |
| on_end       | 节点结束      |
| on_token     | LLM token |
| on_message   | 工具/LLM 消息 |
| on_interrupt | 等待用户输入    |
  
这对于：  
* 多节点 Agent  
* RAG + 评估  
* Planner / Tool / Executor  
* 调试复杂 workflow  
* 前端实时渲染  
非常关键。  
  
## 🧪** 五、复杂工作流流式示例（多节点）**  
以下是一个****多节点 RAG 流式 pipeline****：  
1. query_preprocess  
2. retrieve  
3. rerank  
4. generate_answer（流式输出 token）  
代码结构：  
```
async for event in app.astream_events(
    {"question": "介绍下 LangGraph"},
    config={"thread_id": "u_01"}
):
    if event["type"] == "on_token":
        print(event["token"], end="")
    else:
        print("\n[EVENT]", event)

```
输出类似：  
```
[EVENT] on_start: retrieve
[EVENT] retrieved_docs: 3 items
[EVENT] on_end: retrieve
[EVENT] on_start: rerank
[EVENT] rerank_scores: [...]
[EVENT] on_end: rerank

L  # token
a
n
g
G
r
a
p
h
...

```
这是 ****真正的 Workflow-level Streaming****。  
  
## 🔥** 六、人机交互 + 流式（最常用场景）**  
当 workflow 跑到一个节点：  
```
raise Interrupt("need_user_input")

```
前端会收到：  
```
[EVENT] interrupt

```
然后用户输入内容后：  
```
async for event in app.send(
    {"user_input": "..."},
    config={"thread_id": "u01"}
):
    ...

```
****继续流式输出 token****。  
  
## 🧠** 七、工程级最佳实践**  
## ✔** 1）所有 LLM 节点都打开 streaming**  
```
ChatOpenAI(model="deepseek-chat", streaming=True)

```
## ✔** 2）使用 astream_events 而不是 stream**  
特别是多节点、多 Agent、要前端渲染的应用。  
## ✔** 3）用事件类型区分 UI 操作**  
例如：  

| 事件        | 前端处理     |
| --------- | -------- |
| on_token  | 输出到聊天框   |
| on_start  | 显示“执行中…” |
| on_end    | 显示“完成”   |
| interrupt | 显示表单、按钮  |
  
✔**** 4）结合 Checkpointer 做“流式 + 可恢复”****  
支持：  
* 前端长链路任务  
* 中断后恢复  
* 审计回放  
## ✔** 5）结合 RAG Flow**  
retriever → rerank → generator → stream token 是最经典的 RAG 交互方式。  
  
## 🎯** 八、总结：LangGraph 的流式为什么强？**  
因为它不是 model streaming，而是：  
## ✔** Workflow Streaming**  
## ✔** Node Streaming**  
## ✔** Event Streaming**  
## ✔** Human-in-the-Loop Streaming**  
## ✔** Multi-node, multi-agent Streaming**  
## ✔** Recoverable Streaming**  
一句话：  
**LangChain 流的是 token，LangGraph 流的是“智能体的执行全过程”。**  
**LangChain 流的是 token，LangGraph 流的是“智能体的执行全过程”。**  
这就是它为什么能做“可控 Agent”。  
  
如果你想，我可以继续给你：  
## 🔹** LangGraph + 前端（Next.js）实时流式 Demo**  
## 🔹** LangGraph + DeepSeek R1 Streaming（含思维链过滤）**  
## 🔹** LangGraph 多 Agent Streaming 工作流**  
## 🔹** LangGraph 实现 GPTs 的 Streaming 等价能力**  
你想要哪一个？  
