# 什么是LangGraph？
LangGraph 是 LangChain 发布的一个多智能体框架。通过建立在LangChain 之上，LangGraph 使开发人员可以轻松创建强大的智能体。

![](./images/1766296177189-4dbb5b74-7ed5-4e50-8ade-35f64d4c3ae7.png)

### LangGraph 的核心功能
+ **支持循环流：** LangGraph 允许定义包含循环的流程，这对于大多数代理架构至关重要。这使得 LangGraph 更适合构建需要记忆和上下文推理的应用程序。
+ **状态管理：** LangGraph 提供了状态管理功能，允许代理在多个步骤之间存储和检索信息。这对于构建需要跟踪对话状态或游戏状态的应用程序至关重要。
+ **多参与者支持：** LangGraph 支持多个代理相互交互，以实现更复杂的工作流程。这使得 LangGraph 非常适合构建需要协作或竞争的代理应用程序。
+ **可扩展性：** LangGraph 可以扩展到生产环境，以支持大规模应用程序。

# LangGraph和LangChain的区别
LangGraph和LangChain是两个相关但不同的工具，都来自LangChain生态系统。

## LangChain
LangChain是一个用于构建大语言模型应用程序的框架

+ **线性工作流**：主要支持顺序执行的链式操作
+ **组件库**：提供丰富的预构建组件，如提示模板、向量存储、检索器等
+ **简单集成**：易于快速原型开发和简单的LLM应用
+ **抽象层**：为不同的LLM提供统一接口

## LangGraph
LangGraph是LangChain团队开发的更高级工具，专门用于构建复杂的多智能体系统：

+ **图状工作流**：支持复杂的分支、循环和条件逻辑
+ **状态管理**：内置强大的状态管理机制
+ **多智能体协作**：原生支持多个AI智能体之间的交互
+ **复杂决策流**：可以根据条件动态选择执行路径
+ **持久化**：支持长时间运行的工作流和状态持久化

## 主要区别
**复杂性处理**：

+ LangChain适合简单到中等复杂度的应用
+ LangGraph专为复杂的多步骤、多智能体场景设计

**工作流结构**：

+ LangChain主要是链式（Chain）结构
+ LangGraph是图状（Graph）结构，支持任意的节点连接

暂时无法在飞书文档外展示此内容

# LangGraph安装和使用
```plain
pip install -U "langgraph>=0.6.1"
```

简单Agent

```plain
from langgraph.prebuilt import create_react_agent
from langchain.chat_models import init_chat_model
from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from datetime import datetime
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-07-28')


# 定义工具函数
@tool
def get_data_tool():
    """获取目前日期的工具"""
    return datetime.now().date()


tools = [TavilySearch(), get_data_tool]

system_prompt = """你是一个智能助手。你有以下工具可以使用：

1. search_web: 用于搜索互联网获取最新信息，特别是产品价格、新闻、实时数据等
2. get_data_tool: 获取今天的日期

重要规则：
- 当用户询问产品价格、最新信息、新闻等需要实时数据的问题时，必须使用search_web工具
- 当用户询问时间或日期时，使用相应的时间工具
- 如果你的知识库中没有准确或最新的信息，应该使用搜索工具
- 优先使用工具获取准确信息，而不是依赖可能过时的训练数据

请根据用户问题选择合适的工具来获取准确答案。"""

agent = create_react_agent(model=llm,
                           tools=tools,
                           prompt=system_prompt,
                           debug=False
                           )

response = agent.invoke({"messages": [{"role": "user", "content": "小米yu7价格"}]})

print(response["messages"])
```

# LangGraph基础知识（核心概念）
| **<font style="color:#1f2329;">序号</font>** | **<font style="color:#1f2329;">核心概念</font>** |
| --- | --- |
| <font style="color:#1f2329;">① State（状态）</font> | <font style="color:#1f2329;">存储工作流中每个步骤的上下文信息（如问题、回答、变量等）</font> |
| <font style="color:#1f2329;">② Node（节点）</font> | <font style="color:#1f2329;">工作流中执行的一个步骤（如调用 LLM、调用工具、某个函数）</font> |
| <font style="color:#1f2329;">③ Edge（边/跳转）</font> | <font style="color:#1f2329;">控制流程走向的“路口”，决定从哪个节点跳到哪个节点（支持条件判断）</font> |
| <font style="color:#1f2329;">④ Graph（流程图）</font> | <font style="color:#1f2329;">将所有节点和边组织起来形成一张状态流程图，是 LangGraph 的执行主结构</font> |


## Graph（流程图）
LangGraph 的核心是将代理工作流程建模为图表。您可以使用三个关键组件来定义代理的行为：

1. `State`：表示应用程序当前快照的共享数据结构。它可以是任何 Python 类型，但通常是 `TypedDict`或 Pydantic `BaseModel`。
2. `Nodes`：用于编码代理逻辑的 Python 函数。它们接收当前值`State`作为输入，执行一些计算，并返回更新后的`State`。
3. `Edges`：根据当前条件确定下一步执行哪个操作的 Python 函数`State`。它们可以是条件分支或固定转换。

通过组合`Nodes`和`Edges`，您可以创建复杂的循环工作流，使其`State`随时间推移而演化。然而，真正的强大之处在于 LangGraph 对 的管理方式`State`。需要强调的是：`Nodes`和`Edges`只不过是 Python 函数而已——它们可以包含 LLM 代码，也可以只是经典的 Python 代码。

简而言之：节点负责工作，边负责告诉下一步做什么。

暂时无法在飞书文档外展示此内容

### 状态图
`StateGraph` 类是主要使用的图形类。这是由用户定义的 `State` 对象参数化的。

通俗来说，它是一张**流程图 + 状态管理系统**，定义了：

1. 哪些步骤（节点）要执行？
2. 每一步之间怎么跳转（边）？
3. 整个流程中数据状态如何流动和更新（状态）？

**为什么叫“状态图”而不是“流程图”？**

LangGraph 不只是流程控制，还强调：

+ 每个节点执行前、执行后都可以**访问和修改状态**（state）
+ 状态是图的“血液”，在节点之间流动
+ **节点的跳转可以依据状态来判断**（如条件跳转）

所以叫做 **State Graph（有状态的流程图）**，而不是“静态流程图”。

```plain
from typing import TypedDict
from langgraph.graph import StateGraph

# 定义状态结构
class MyState(TypedDict):
    question: str
    answer: str

# 定义节点函数
def search_node(state):
    return {"answer": "这是答案"}

# 创建状态图
builder = StateGraph(state_schema=MyState)
# 添加一个节点
builder.add_node("search", search_node)
# 第一个要调用的节点
builder.set_entry_point("search")
# 要构建图，首先要定义状态，然后添加节点和边，最后进行编译，会进行基本的检查
graph = builder.compile()

# 执行图
result = graph.invoke({"question": "什么是状态图？"})
print(result["answer"])  # 输出：这是答案
```

## State（状态）
在使用 LangGraph 构建流程图之前，**第一件事**就是定义图的状态 `State`。这是整个图运行中用于**共享和传递信息**的核心机制。

### **什么是 State？**
LangGraph 中的 **State** 是图中所有节点（Node）之间传递数据的**模式结构**，可以类比为一个共享的上下文字典，它包含输入、输出、中间变量等。

定义 State 时，需要包含两个部分：

1. **Schema（模式）**：指定 State 的字段结构（可以用 `TypedDict` 或 `Pydantic`）

```plain
# langgraph推荐使用TypedDict
"""
1. TypedDict 是标准库的一部分（来自 typing 模块），零依赖，零性能开销而 Pydantic 会在每一步创建模型实例，会增加运行时负担
2. LangGraph 中的 State 实质就是一个字典（dict），而 TypedDict 就是“有类型注解的 dict”，与 LangGraph 的执行机制无缝对接，而 Pydantic 是类结构，需要 .dict() 转换，略显多余
"""
from typing import TypedDict


class State1(TypedDict):
    user_input: str

# 使用 pydantic 可以进行参数校验和提供默认值
from pydantic import BaseModel


class State2(BaseModel):
    question: str
    result: str = ""
```

1. **多个模式（Multiple Schemas）：**在大多数情况下，LangGraph 使用一个统一的 State 模式。但你也可以设置“输入模式”和“输出模式”分开
    1. 输入模式：接收用户输入的字段（如 `question`）
    2. 输出模式：只保留最终输出的字段（如 `final_answer`）

```plain
from typing import TypedDict
from langgraph.graph import StateGraph


# 1. 定义输入、输出、图内部的状态结构

# 输入字段：用户的问题
class InputState(TypedDict):
    question: str


# 中间状态：包括中间结果
class InternalState(TypedDict):
    question: str
    search_result: str
    final_answer: str


# 输出字段：只想返回最终答案
class OutputState(TypedDict):
    final_answer: str


# 2. 定义节点函数（中间节点用中间字段）
def search_node(state: InternalState) -> dict:
    return {"search_result": f"搜索了：{state['question']}"}


def answer_node(state: InternalState) -> dict:
    return {"final_answer": f"根据搜索结果：{state['search_result']}，这是答案"}


# 3. 创建 StateGraph，显式指定输入/输出 Schema
builder = StateGraph(state_schema=InternalState, 
                     input_schema=InputState, 
                     output_schema=OutputState)

# 4. 添加节点
builder.add_node("search", search_node)
builder.add_node("answer", answer_node)

# 5. 配置流程
builder.set_entry_point("search")
builder.add_edge("search", "answer")

# 6. 编译并执行图
app = builder.compile()

result = app.invoke({"question": "什么是LangGraph？"})

print(result)  # {'final_answer': '根据搜索结果：搜索了：什么是LangGraph？，这是答案'}
```

1. **Reducer（归并函数）**：在 LangGraph 中，所有节点返回的都是“局部更新结果”，**Reducer 是用于合并多个节点输出更新的机制**。 **将每个节点返回的“局部状态更新”统一合并进全局的 State。**

暂时无法在飞书文档外展示此内容

```plain
from typing import Annotated 
from typing_extensions import TypedDict 
from operator import add 

class State(TypedDict):     
    foo: int     
    bar: Annotated[list[str], add] # 每条消息是 {role, content}，会自动追加到列表末尾
```

### 使用图形状态中的消息
**为什么要使用消息？**

大多数现代 LLM 提供商都提供聊天模型接口，接受消息列表作为输入。LangChain`ChatModel`尤其接受对象列表`Message`作为输入。这些消息有多种形式，例如`HumanMessage`（用户输入）或`AIMessage`（LLM 响应）。

**在图表中使用消息**

在许多情况下，将之前的对话历史记录以消息列表的形式存储在图状态中会很有帮助。为此，我们可以向图状态添加一个键（通道），该键存储`Message`对象列表，并使用 Reducer 函数对其进行注释。Reducer 函数对于指示图如何`Message`在每次状态更新（例如，当节点发送更新时）时更新状态中的对象列表至关重要。如果您未指定 Reducer，则每次状态更新都会用最新提供的值覆盖消息列表。如果您只想将消息附加到现有列表中，可以使用`operator.add`。

```plain
operator 是 Python 的一个内置模块，把常见的运算符（如 +、-、==、getitem 等）变成了函数，方便函数式编程和高阶函数使用。
```

```plain
from typing import TypedDict
from langgraph.graph import StateGraph
from typing import Annotated
import operator


# 定义状态结构   如果定义的是list[dict]，会覆盖之前的数据
class ChatState(TypedDict):
    messages: Annotated[list, operator.add]  # 每条消息是 {role, content}，会自动追加到列表末尾


# 节点函数：添加用户问题
def user_input_node(state: ChatState) -> dict:
    user_msg = {"role": "user", "content": "什么是LangGraph？"}
    return {"messages": [user_msg]}


# 节点函数：添加助手回复
def assistant_node(state: ChatState) -> dict:
    reply = {"role": "assistant", "content": "LangGraph 是一个有状态的图编排框架。"}
    return {"messages": [reply]}


# 构建状态图
builder = StateGraph(state_schema=ChatState)
builder.add_node("user_input", user_input_node)
builder.add_node("assistant_reply", assistant_node)
builder.set_entry_point("user_input")
builder.add_edge("user_input", "assistant_reply")

graph = builder.compile()

result = graph.invoke({"messages": []})
print(result["messages"])
```

有场景可能还需要手动更新图状态中的消息（例如，人机交互）。 如果您使用 `operator.add`，您发送到图的手动状态更新将被附加到现有消息列表中，而不是更新现有消息。 为了避免这种情况，您需要一个能够跟踪消息 ID 并在更新时覆盖现有消息的 Reducer。 为此，您可以使用预构建 `add_messages` 函数。 对于新消息，它只会附加到现有列表中，但它也会正确处理现有消息的更新。

```plain
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

### MessagesState
由于在状态中包含消息列表非常常见，因此存在一个名为`MessagesState`的预建状态，它使使用消息变得非常简单。该状态`MessagesState`使用单个键定义`messages`，该键是对象列表`AnyMessage`并使用`add_messages`。通常，需要跟踪的状态不仅仅是消息，因此我们看到人们将这个状态子类化并添加更多字段，例如：

```plain
from langgraph.graph import MessagesState

# 和上述代码不同会在State类中自动维护一个messages 字段，不需要显示创建
class State(MessagesState):
    documents: list[str]
```

## Node（节点）
**节点（Nodes）是图中执行逻辑的基本单位**。每个节点表示一个**函数步骤、处理阶段或子逻辑流程**，多个节点通过边连接成有向图，组成一个完整的有状态计算流程。

**LangGraph 中的节点就是你定义的一个函数（或 Runnable 对象）**，用于接收状态、执行逻辑，并返回更新后的状态

```plain
def my_node(state: dict) -> dict:
    # 处理输入状态，并返回更新字段
    return {"new_key": "new_value"}
    
# LangGraph 会自动用 reducer 把这些更新合并进全局状态。
```

### `START`节点
Node`START`是一个特殊节点，表示将用户输入发送到图的节点。引用此节点的主要目的是确定应首先调用哪些节点。

(API 参考：START)

```plain
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

### `END`节点
Node`END`是一个特殊节点，表示终端节点。当需要指示哪些边在完成后没有操作时，可以引用此节点。

```plain
from langgraph.graph import END

graph.add_edge("node_a", END)
```

### 并行运行节点
```plain
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # The operator.add reducer fn makes this append-only
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_node(d)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()

print(graph.invoke({"aggregate": ["start"]}))
```

## Edge（边/跳转）
**Edge（边）** 是连接节点的通道，表示图中**节点之间的执行跳转关系**。你可以把它理解为「节点执行完之后，下一步去哪，是构成 LangGraph 流程图的核心。

**Edge 是 LangGraph 中连接两个节点的“执行路径”**，控制流程的走向。

+ 普通边：直接从一个节点到下一个节点。

```plain
graph.add_edge("节点A", "节点B")
```

+ 条件边：调用一个函数来确定下一步要去哪个节点。

```plain
from typing import TypedDict
from langgraph.graph import StateGraph, END


class MyState(TypedDict):
    type: str
    result: str


def judge_node(state: MyState):
    """节点函数：可以做一些预处理"""
    return state  # 保持状态不变，只是路由


def route_condition(state: MyState):
    """条件函数：只负责路由决策"""
    if state["type"] == "a":
        return "a"
    elif state["type"] == "b":
        return "b"
    else:
        return "default"


def node_a(state):
    return {"result": "走了 A 分支"}


def node_b(state):
    return {"result": "走了 B 分支"}


def node_default(state):
    return {"result": "走了默认分支"}


# 构建图
graph = StateGraph(state_schema=MyState)
# 定义节点
graph.add_node("judge_node", judge_node)
graph.add_node("a", node_a)
graph.add_node("b", node_b)
graph.add_node("default", node_default)
# 定义开始节点
graph.set_entry_point("judge_node")

# 使用不同的函数作为条件函数
graph.add_conditional_edges("judge_node", route_condition, {
    "a": "a",
    "b": "b",
    "default": "default"
})

# 添加结束边
graph.add_edge("a", END)
graph.add_edge("b", END)
graph.add_edge("default", END)

app = graph.compile()

# 测试
print("测试 A:", app.invoke({"type": "a", "result": ""}))
```

+ 入口点：当图开始运行时首先运行的第一个（些）节点。

```plain
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

+ 条件入口点：调用一个函数来确定当用户输入到达时首先调用哪个节点。

```plain
from typing import TypedDict
from langgraph.graph import StateGraph, END


class MyState(TypedDict):
    user_type: str  # "vip", "normal", "guest"
    message: str
    result: str


# 定义不同的处理节点
def vip_service(state):
    """VIP 用户服务"""
    return {"result": f"VIP专享服务: {state['message']}"}


def normal_service(state):
    """普通用户服务"""
    return {"result": f"标准服务: {state['message']}"}


def guest_service(state):
    """游客服务"""
    return {"result": f"游客服务(功能受限): {state['message']}"}


# 条件入口点函数
def route_by_user_type(state):
    """根据用户类型路由到不同的服务"""
    user_type = state["user_type"]

    if user_type == "vip":
        return "vip_service"
    elif user_type == "normal":
        return "normal_service"
    else:
        return "guest_service"


# 构建图
workflow = StateGraph(state_schema=MyState)

# 添加节点
workflow.add_node("vip_service", vip_service)
workflow.add_node("normal_service", normal_service)
workflow.add_node("guest_service", guest_service)

# 设置条件入口点 - 关键部分！
workflow.set_conditional_entry_point(
    route_by_user_type,  # 条件函数
    {
        "vip_service": "vip_service",
        "normal_service": "normal_service",
        "guest_service": "guest_service"
    }
)

# 添加结束边
workflow.add_edge("vip_service", END)
workflow.add_edge("normal_service", END)
workflow.add_edge("guest_service", END)

# 编译图
app = workflow.compile()
# 测试 VIP 用户
result1 = app.invoke({
    "user_type": "vip",
    "message": "我要退款",
    "result": ""
})
print("VIP用户:", result1)
```

## Send发送
默认情况下，`Nodes`和`Edges`是提前定义的，并在相同的共享状态下运行。但是，在某些情况下，确切的边无法提前知道，并且可能希望同时存在`State`的不同版本。

`send("节点名", 更新数据)` 的作用是：告诉 LangGraph 把这部分更新数据发送给指定节点，让它继续执行。

### 主要用途
1. **条件路由**：根据某些条件将消息发送到不同的节点
2. **并行处理**：同时向多个节点发送消息 
3. **动态工作流**：根据运行时状态决定消息的发送目标

```plain
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from typing import TypedDict

class MyState(TypedDict):
    messages: list
    result: str
    info: str

def process_input(state: MyState) -> Send:
    """处理输入并决定发送到哪个节点"""
    if len(state["messages"]) > 5:
        return Send("complex_processor", state)
    else:
        return Send("simple_processor", state)

def simple_processor(state: MyState) -> MyState:
    """简单处理器"""
    return {"messages": state["messages"], "result": "simple"}

def complex_processor(state: MyState) -> MyState:
    """复杂处理器"""
    return {"messages": state["messages"], "result": "complex"}

def end_processor(state: MyState) -> MyState:
    """结束处理器"""
    return {"info": "我是结束节点"}

# 构建图
graph = StateGraph(state_schema=MyState)
graph.add_node("input", process_input)
graph.add_node("simple_processor", simple_processor)
graph.add_node("complex_processor", complex_processor)
graph.add_node("end_processor", end_processor)


graph.add_edge(START, "input")
graph.add_edge("input", "end_processor")
graph.add_edge("end_processor", END)
app = graph.compile()
a = {"messages": [i for i in range(3)]}
print(app.invoke({"messages": [123]}))
```

**和add_conditional_edges有什么区别呢？**

+ `**add_conditional_edges**`：**外部决策**。路由逻辑在一个单独的函数中，它在节点运行之后被调用，根据图的当前状态来决定下一步去哪里。
+ `**Send**`：**内部决策**。路由逻辑在节点自身的函数体内，节点在运行时直接、显式地决定将其结果发送到哪个特定的节点

两者通常在需要并发处理的时候配合使用

### Map-Reduce模式
**Map-Reduce **是一种经典的并行计算模式，特别适合处理大规模数据。

**Map-Reduce** 将复杂的数据处理任务分解为两个阶段：

1. **Map阶段**：将大任务分解为多个小任务，并行处理
2. **Reduce阶段**：将所有小任务的结果合并成最终结果

```plain
"""
LangGraph Map-Reduce 简单案例：数字求和
把一堆数字分给多个worker算平方，然后把结果加起来
"""
from typing import Annotated
import operator
from langgraph.graph import StateGraph, START, END
from langgraph.constants import Send
from typing import TypedDict, List

# 状态定义
class State(TypedDict):
    numbers: List[int]        # 输入的数字
    results: Annotated[list[int], operator.add]        # worker的结果
    final_sum: int           # 最终求和

# 1. Map阶段：分发数字
def split_numbers(state: State):
    """把数字分发给不同的worker"""
    numbers = state["numbers"]
    print(f"分发数字: {numbers}")

    # 每个数字发给一个worker
    return [Send("worker", {"number": num}) for num in numbers]

# 2. Worker阶段：计算平方
def calculate_square(state: State):
    """每个worker计算一个数字的平方"""
    number = state["number"]
    square = number * number
    print(f"Worker: {number}² = {square}")
    return {"results": [square]}

# 3. Reduce阶段：求和
def sum_results(state: State):
    """把所有结果加起来"""
    results = state.get("results", [])
    total = sum(results)
    print(f"求和: {results} = {total}")
    return {"final_sum": total}

# 构建图
def create_simple_graph():
    graph = StateGraph(State)

    # 添加节点
    graph.add_node("splitter", lambda s: s)  # 分发器
    graph.add_node("worker", calculate_square)  # 工作节点
    graph.add_node("summer", sum_results)      # 求和器

    # 连接节点
    graph.add_edge(START, "splitter")
    graph.add_conditional_edges("splitter", split_numbers, ["worker"])  # Map阶段
    graph.add_edge("worker", "summer")  # Worker完成后求和
    graph.add_edge("summer", END)

    return graph.compile()

# 运行例子
def run_example():
    app = create_simple_graph()

    # 测试数据
    initial_state = {
        "numbers": [1, 2, 3, 4, 5],
        "results": [],
        "final_sum": 0
    }

    print("开始计算...")
    print("任务：计算每个数字的平方，然后求和")
    print()

    # 运行
    result = app.invoke(initial_state)
    print(result)

if __name__ == "__main__":
    run_example()
```

## `<font style="color:rgb(216,57,49);">Command</font>`<font style="color:rgb(216,57,49);">命令</font>
将控制流（边）和状态更新（节点）结合在一起可能非常有用。例如，您可能希望在同一个节点中既执行状态更新，又决定接下来要去哪个节点。

在节点函数中返回时`Command`，必须添加返回类型注释，其中包含节点路由到的节点名称列表，例如`Command[Literal["my_other_node"]]`。这对于图形渲染是必需的，它告诉 LangGraph`my_node`可以导航到`my_other_node`。

```plain
from typing import TypedDict
from langgraph.graph import StateGraph, END
from langgraph.types import Command, Send
from typing import Literal


class MyState(TypedDict):
    type: str
    text: str
    result: str


def judge_node(state: MyState) -> Command[Literal["a", "b", "default"]]:
    """条件函数：使用Command进行路由和状态更新"""
    if state["type"] == "a":
        return Command(update={"text": "走了 A 分支"}, goto="a")
    elif state["type"] == "b":
        # Command只是更新当前节点结束的状态
        # return Command(update={"text": "走了 B 分支"}, goto="b")
        return Send("b", state)
    else:
        return Command(update={"text": "走了默认分支"}, goto="default")


def node_a(state):
    return {"result": f"A节点处理: {state['text']}"}


def node_b(state):
    return {"result": f"B节点处理: {state['text']}"}


def node_default(state):
    return {"result": f"默认节点处理: {state['text']}"}


# 构建图
graph = StateGraph(state_schema=MyState)
graph.add_node("judge_node", judge_node)
graph.add_node("a", node_a)
graph.add_node("b", node_b)
graph.add_node("default", node_default)

graph.set_entry_point("judge_node")

# 添加结束边
graph.add_edge("a", END)
graph.add_edge("b", END)
graph.add_edge("default", END)

app = graph.compile()

# 测试
print("测试 A:", app.invoke({"type": "a", "text": "", "result": ""}))
print("测试 B:", app.invoke({"type": "b", "text": "", "result": ""}))
print("测试其他:", app.invoke({"type": "default", "text": "", "result": ""}))
```

### 什么时候应该使用`Command`而不是条件边？
`Command`当需要**同时**更新图形状态**和**路由到其他节点时使用。例如，在实现多代理切换时，需要路由到其他代理并向该代理传递一些信息。在进行command更新状态的时候，更新的属性必须符合初始化状态的内容

使用条件边在节点之间有条件地路由而不更新状态。

## 配置`Runtime`
创建图时，还可以标记图的某些部分是可配置的。这样做通常是为了方便在模型或系统提示之间切换。这允许创建单个“认知架构”（图），但拥有多个不同的实例。

**在运行图时提供额外的“配置参数”而不是“状态参数”**，并且通过类型约束这些参数。

```plain
from langgraph.graph import StateGraph
from langgraph.runtime import Runtime
from typing import TypedDict

# 定义状态结构
class MyState(TypedDict):
    question: str
    answer: str

# 定义配置结构
class MyContext(TypedDict):
    language: str  # 配置中包含语言选项，比如 "en" 或 "zh"

# 节点函数可以访问 runtime 参数  runtime 可以访问上下文和内存存储
def step1(state: MyState, runtime: Runtime[MyContext]):
    if runtime.context["language"] == "zh":
        answer = "你好！"
    else:
        answer = "Hello!"
    return {"answer": answer}

# 构建图..
graph = StateGraph(state_schema=MyState, context_schema=MyContext)
graph.add_node("step1", step1)
graph.set_entry_point("step1")

# 编译
app = graph.compile()

# 执行时传入 config 参数（区分于 state）
result = app.invoke({"question": "Hi"}, context={"language": "zh"})
print(result)  # => {"question": "Hi", "answer": "你好！"}
```

在运行时指定llm

```plain
from langgraph.graph import MessagesState
from langgraph.runtime import Runtime
from langgraph.graph import END, StateGraph, START
from typing_extensions import TypedDict


class MyContext(TypedDict):
    model: str


MODELS = {
    "anthropic": "anthropic:claude-3-5-haiku-latest",
    "openai": "openai:gpt-4.1-mini",
}


def call_model(state: MessagesState, runtime: Runtime[MyContext]):
    model = ""
    if runtime.context:
        model = runtime.context["model"]
        model = MODELS[model]
    return {"messages": {"role": "assistant", "content": model}}


builder = StateGraph(MessagesState, context_schema=MyContext)
builder.add_node("model", call_model)
builder.add_edge(START, "model")
builder.add_edge("model", END)

graph = builder.compile()

# 问题
input_message = {"role": "user", "content": "hi"}
# 没有配置时，使用默认值（Anthropic）
response_1 = graph.invoke({"messages": [input_message]})
# 切换成 openai
context = {"model": "openai"}
response_2 = graph.invoke({"messages": [input_message]}, context=context)

print(response_1)
print(response_2)
```

### 递归限制
递归限制设置图在单次执行中可以执行的最大超步数。一旦达到限制，LangGraph 将出现`GraphRecursionError`。默认情况下，此值设置为 25 步。可以在运行时在任何图上设置递归限制，并将其传递给`.invoke`/`.stream`通过配置字典。

```plain
import operator
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.managed.is_last_step import RemainingSteps

class State(TypedDict):
    aggregate: Annotated[list, operator.add]
    remaining_steps: RemainingSteps

def a(state: State):
    print(f'Node A sees {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Node B sees {state["aggregate"]}')
    return {"aggregate": ["B"]}

# Define nodes
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

# Define edges
def route(state: State) -> Literal["b", END]:
    if state["remaining_steps"] <= 2:
        return END
    else:
        return "b"

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()

# Test it out
result = graph.invoke({"aggregate": []}, {"recursion_limit": 10})
print(result)
```

## 可视化图表
```plain
"""
LangGraph Map-Reduce 简单案例：数字求和
把一堆数字分给多个worker算平方，然后把结果加起来
"""
from typing import Annotated
import operator
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from typing import TypedDict, List

# 状态定义
class State(TypedDict):
    numbers: List[int]        # 输入的数字
    results: Annotated[list[int], operator.add]        # worker的结果
    final_sum: int           # 最终求和

# 1. Map阶段：分发数字
def split_numbers(state: State):
    """把数字分发给不同的worker"""
    numbers = state["numbers"]
    print(f"📦 分发数字: {numbers}")

    # 每个数字发给一个worker
    return [Send("worker", {"number": num}) for num in numbers]

# 2. Worker阶段：计算平方
def calculate_square(state: State):
    """每个worker计算一个数字的平方"""
    number = state["number"]
    square = number * number
    print(f"⚡ Worker: {number}² = {square}")
    return {"results": [square]}

# 3. Reduce阶段：求和
def sum_results(state: State):
    """把所有结果加起来"""
    results = state.get("results", [])
    total = sum(results)
    print(f"📊 求和: {results} = {total}")
    return {"final_sum": total}

# 构建图
def create_simple_graph():
    graph = StateGraph(State)

    # 添加节点
    graph.add_node("splitter", lambda s: s)  # 分发器
    graph.add_node("worker", calculate_square)  # 工作节点
    graph.add_node("summer", sum_results)      # 求和器

    # 连接节点
    graph.add_edge(START, "splitter")
    graph.add_conditional_edges("splitter", split_numbers, ["worker"])  # Map阶段
    graph.add_edge("worker", "summer")  # Worker完成后求和
    graph.add_edge("summer", END)

    return graph.compile()

# 运行例子
def run_example():
    app = create_simple_graph()

    # 测试数据
    initial_state = {
        "numbers": [1, 2, 3, 4, 5],
        "results": [],
        "final_sum": 0
    }

    print("🚀 开始计算...")
    print("任务：计算每个数字的平方，然后求和")
    print()

    # 运行
    app.invoke(initial_state)

    from IPython.display import Image, display
    from langchain_core.runnables.graph import MermaidDrawMethod
    display(
        Image(
            app.get_graph().draw_mermaid_png(
                draw_method=MermaidDrawMethod.API,
                output_file_path='./可视化图.png'
            )
        )
    )

if __name__ == "__main__":
    run_example()
```

# 子图
LangGraph子图（Subgraph）是一种模块化的图结构，允许您将复杂的工作流分解为更小的、可重用的组件。就像函数在编程中的作用一样，子图提供了封装和复用的能力。

![](./images/1766296155045-ed3a4f4b-086a-40c7-9042-c4da1e2cc328.png)

## 子图的优势
1. **代码复用**：避免重复编写相同的逻辑
2. **清晰的架构**：将复杂流程分解为清晰的模块
3. **易于维护**：修改子图只需在一个地方进行
4. **团队协作**：不同团队可以独立开发不同的子图
5. **测试友好**：可以单独测试子图的功能

## 两种状态通讯
### **1.共享状态键（Shared State Keys）**
父图和子图在其状态模式中有共享的状态键。在这种情况下，您可以将子图作为节点包含在父图中。

```plain
from langgraph.graph import StateGraph, MessagesState, START
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


# 创建子图
def subplot(state: MessagesState) -> MessagesState:
    # 获取大模型回答的内容进行摘要总结
    answer = state["messages"][-1].content
    summary_prompt = f"请用一句话总结下面这句话：\n\n答：{answer}"
    response = llm.invoke(summary_prompt)
    return {"messages": state["messages"] + [response]}


summary_subgraph = (
    StateGraph(state_schema=MessagesState)
    .add_node("subplot", subplot)
    .add_edge(START, "subplot")
    .compile()
)


# 创建父图

def llm_answer_node(state: MessagesState) -> MessagesState:
    # 使用大模型进行回答
    answer = llm.invoke(state["messages"])
    print("父图输出", answer)
    return {"messages": state["messages"] + [answer]}


parent_graph = (
    StateGraph(MessagesState)
    .add_node("llm_answer", llm_answer_node)
    .add_node("summarize_subgraph", summary_subgraph)
    .add_edge(START, "llm_answer")
    .add_edge("llm_answer", "summarize_subgraph")
    .compile()
)

# 测试
input_state = {
    "messages": [{"role": "user", "content": "langgraph是什么？"}],
}
result = parent_graph.invoke(input_state)
print(result)
```

### 2.不同状态模式（Different State Schemas）常用
父图和子图有不同的模式（状态模式中没有共享的状态键）。在这种情况下，您必须在父图的节点内部调用子图：这在父图和子图有不同状态模式且需要在调用子图前后转换状态时很有用。

```plain
from langgraph.graph import StateGraph, MessagesState, START
from typing_extensions import TypedDict, Annotated
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


# 创建子图
class SubgraphMessagesState(TypedDict):
    subgraph_messages: Annotated[list[AnyMessage], add_messages]


def subplot(state: SubgraphMessagesState) -> SubgraphMessagesState:
    # 获取大模型回答的内容进行摘要总结
    answer = state["subgraph_messages"][-1].content
    summary_prompt = f"请用一句话总结下面这句话：\n\n答：{answer}"
    response = llm.invoke(summary_prompt)
    print("\n\n")
    print("子图中问题和输出:", state["subgraph_messages"] + [response])
    return {"subgraph_messages": [response]}


summary_subgraph = (
    StateGraph(state_schema=SubgraphMessagesState)
    .add_node("subplot", subplot)
    .add_edge(START, "subplot")
    .compile()
)


# 创建父图

def llm_answer_node(state: MessagesState) -> MessagesState:
    # 使用大模型进行回答
    answer = llm.invoke(state["messages"])
    print("父图中问题和输出:", state["messages"] + [answer])
    # 转换状态格式
    summary_result = summary_subgraph.invoke({"subgraph_messages": state["messages"] + [answer]})
    return {"messages": state["messages"] + [answer]+ [summary_result["subgraph_messages"][2]]}


parent_graph = (
    StateGraph(state_schema=MessagesState)
    .add_node("llm_answer", llm_answer_node)
    .add_edge(START, "llm_answer")
    .compile()
)

# 测试输入
input_state = {
    "messages": [{"role": "user", "content": "langgraph是什么？"}],
}
result = parent_graph.invoke(input_state)
print("最终结果：", result)
```

# 流式输出
LangGraph 实施了流媒体系统来显示实时更新，从而实现响应迅速且透明的用户体验。

LangGraph 的流式传输系统可将图形运行的实时反馈显示到您的应用中。

**流式输出在LangGraph中的重要性：**

⚡ 用户立即看到反馈

🎯 减少等待时间

💾 节省内存使用

😊 提升用户体验

将以下一个或多个流模式作为列表传递给`stream()`或`astream()`方法：

| **<font style="color:rgb(0, 0, 0);">模式</font>** | **<font style="color:rgb(0, 0, 0);">描述</font>** | **<font style="color:rgb(0, 0, 0);">概述</font>** |
| :---: | --- | :---: |
| <font style="color:#1f2329;">values</font> | <font style="color:#1f2329;">在图的每个步骤之后流式传输状态的完整值。</font> | <font style="color:#1f2329;">看到完整状态</font> |
| <font style="color:#1f2329;">updates</font> | <font style="color:#1f2329;">将图的每个步骤之后的更新流式传输到状态。如果在同一步骤中进行了多个更新（例如，运行了多个节点），则这些更新将分别流式传输。</font> | <font style="color:#1f2329;">看到变化部分</font> |
| <font style="color:#1f2329;">custom</font> | <font style="color:#1f2329;">从图形节点内部流式传输自定义数据。</font> | <font style="color:#1f2329;">看到自定义数据</font> |
| <font style="color:#1f2329;">messages</font> | <font style="color:#1f2329;">从调用 LLM 的任何图形节点流式传输 2 元组（LLM 令牌、元数据）。</font> | <font style="color:#1f2329;">看到AI逐字输出</font> |
| <font style="color:#1f2329;">debug</font> | <font style="color:#1f2329;">在整个图表执行过程中传输尽可能多的信息。</font> | <font style="color:#1f2329;">看到调试信息</font> |


## 五种模式代码详解
```plain
from typing import Annotated
import operator
from langgraph.graph import StateGraph, START, END
from langgraph.constants import Send
from typing import TypedDict, List

"""values、updates、debug模式"""


# 状态定义
class State(TypedDict):
    numbers: List[int]  # 输入的数字
    results: Annotated[list[int], operator.add]  # worker的结果
    final_sum: int  # 最终求和


# 1. Map阶段：分发数字
def split_numbers(state: State):
    """把数字分发给不同的worker"""
    numbers = state["numbers"]

    # 每个数字发给一个worker
    return [Send("worker", {"number": num}) for num in numbers]


# 2. Worker阶段：计算平方
def calculate_square(state: State):
    """每个worker计算一个数字的平方"""
    number = state["number"]
    square = number * number
    return {"results": [square]}


# 3. Reduce阶段：求和
def sum_results(state: State):
    """把所有结果加起来"""
    results = state.get("results", [])
    total = sum(results)
    return {"final_sum": total}


# 构建图
def create_simple_graph():
    graph = StateGraph(state_schema=State)

    # 添加节点
    graph.add_node("splitter", lambda s: s)  # 分发器
    graph.add_node("worker", calculate_square)  # 工作节点
    graph.add_node("summer", sum_results)  # 求和器

    # 连接节点
    graph.add_edge(START, "splitter")
    graph.add_conditional_edges("splitter", split_numbers, ["worker"])  # Map阶段
    graph.add_edge("worker", "summer")  # Worker完成后求和
    graph.add_edge("summer", END)

    return graph.compile()


"""messages模式"""
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


class MyState(TypedDict):
    question: str
    results: str


def generate_answer(state: MyState):
    question = state["question"]
    answer = llm.invoke([
        {"role": "user", "content": f"{question}"}
    ])
    return {"answer": answer.content}


# 构建图
def create_llm_graph():
    graph = StateGraph(state_schema=MyState)

    # 添加节点
    graph.add_node("generate_answer", generate_answer)

    # 连接节点
    graph.add_edge(START, "generate_answer")
    graph.add_edge("generate_answer", END)

    return graph.compile()


"""自定义模式"""

from langgraph.config import get_stream_writer
import time


# 定义状态
class FileState(TypedDict):
    filename: str  # 文件名称
    content: str  # 文件内容
    word_count: int  # 内容数量
    processed: bool  # 是否处理


def read_file(state: FileState):
    """步骤1：读取文件"""
    writer = get_stream_writer()
    # 发送开始信息
    writer({"step": "读取文件", "status": "开始", "progress": 0})
    time.sleep(1)

    # 发送进度信息
    writer({"step": "读取文件", "status": "正在读取...", "progress": 50})
    time.sleep(1)

    # 模拟文件内容
    content = "这是一个示例文件，包含一些文本内容。"

    # 发送完成信息
    writer({
        "step": "读取文件",
        "status": "完成",
        "progress": 100,
        "data": {"size": len(content)}
    })

    return {"content": content}


def count_words(state: FileState):
    """步骤2：统计字数"""
    writer = get_stream_writer()
    writer({"step": "统计字数", "status": "开始", "progress": 0})
    time.sleep(0.5)

    writer({"step": "统计字数", "status": "正在分析...", "progress": 30})
    time.sleep(1)

    writer({"step": "统计字数", "status": "计算中...", "progress": 70})
    time.sleep(0.5)

    # 计算字数
    word_count = len(state["content"])

    writer({
        "step": "统计字数",
        "status": "完成",
        "progress": 100,
        "data": {"word_count": word_count}
    })

    return {"word_count": word_count}


def finalize_processing(state: FileState):
    """步骤3：完成处理"""
    writer = get_stream_writer()
    writer({"step": "完成处理", "status": "生成报告", "progress": 50})
    time.sleep(1)

    writer({
        "step": "完成处理",
        "status": "全部完成",
        "progress": 100,
        "data": {
            "filename": state["filename"],
            "total_chars": state["word_count"],
            "summary": f"文件 {state['filename']} 处理完成，共 {state['word_count']} 个字符"
        }
    })

    return {"processed": True}


# 构建图
def create_custom_graph():
    graph = (
        StateGraph(state_schema=FileState)
        .add_node("read_file", read_file)
        .add_node("count_words", count_words)
        .add_node("finalize", finalize_processing)
        .add_edge(START, "read_file")
        .add_edge("read_file", "count_words")
        .add_edge("count_words", "finalize")
        .compile()
    )
    return graph


# 运行例子
def run_example():
    app = create_simple_graph()
    app1 = create_llm_graph()
    app2 = create_custom_graph()

    # 测试数据
    initial_state = {
        "numbers": [1, 2, 3, 4, 5],
        "results": [],
        "final_sum": 0
    }

    print("====================VALUES模式=====================")
    for result in app.stream(initial_state, stream_mode="values"):
        print(result)

    print("====================UPDATES模式=====================")
    for result in app.stream(initial_state, stream_mode="updates"):
        print(result)

    print("====================DEBUG模式=====================")
    for result in app.stream(initial_state, stream_mode="debug"):
        print(result)

    print("====================MESSAGES模式=====================")
    for result in app1.stream({"question": "什么是状态图？"}, stream_mode="messages"):
        print(result[0].content)

    print("====================CUSTOM模式=====================")
    # 初始状态
    initial_state1 = {
        "filename": "example.txt",
        "content": "",
        "word_count": 0,
        "processed": False
    }
    # 使用Custom模式运行
    for chunk in app2.stream(initial_state1, stream_mode="custom"):
        step = chunk.get("step", "")
        status = chunk.get("status", "")
        progress = chunk.get("progress", 0)
        data = chunk.get("data", {})

        # 显示进度
        progress_bar = "█" * (progress // 10) + "░" * (10 - progress // 10)
        print(f"\n[{step}] {status}")
        print(f"进度: [{progress_bar}] {progress}%")

        # 显示额外数据
        if data:
            for key, value in data.items():
                print(f"📊 {key}: {value}")


if __name__ == "__main__":
    run_example()
```

## 融合多种模式
可以传递一个列表作为`stream_mode`参数来同时传输多种模式

输出将是流模式名称和`(mode, chunk)`

```plain
from typing import TypedDict
from dotenv import load_dotenv
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
import os

load_dotenv()


# ==========================================
# 1. 定义状态和工具
# ==========================================

class EditorState(TypedDict):
    topic: str  # 主题
    content: str  # 生成的内容
    score: int  # 评分
    status: str  # 当前状态描述


llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                       base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                       model_provider="openai",
                       model='qwen-plus-2025-07-28', )


# ==========================================
# 2. 定义节点逻辑
# ==========================================

def write_article(state: EditorState):
    """节点1：负责写文章（耗时操作，会有流式输出）"""
    topic = state["topic"]
    # 这里我们用 invoke，依靠 stream_mode="messages" 来捕获流
    response = llm.invoke(f"请写一段关于'{topic}'的短文，50字左右。")
    return {
        "content": response.content,
        "status": "写作完成"
    }


def review_article(state: EditorState):
    """节点2：负责打分（逻辑操作，瞬间完成）"""
    # 简单模拟打分逻辑
    content_len = len(state["content"])
    score = min(100, content_len * 2)
    return {
        "score": score,
        "status": "评分完成"
    }


# ==========================================
# 3. 构建图
# ==========================================

workflow = StateGraph(EditorState)

workflow.add_node("writer", write_article)
workflow.add_node("reviewer", review_article)

workflow.add_edge(START, "writer")
workflow.add_edge("writer", "reviewer")
workflow.add_edge("reviewer", END)

app = workflow.compile()


# ==========================================
# 4. 核心：融合流式输出处理
# ==========================================

def run_mixed_mode_demo():
    inputs = {
        "topic": "人工智能的未来",
        "content": "",
        "score": 0,
        "status": "开始任务"
    }

    print(f"🚀 任务启动：主题 - {inputs['topic']}\n")
    print("-" * 50)

    # 🌟 关键点：传入一个列表 ["messages", "updates", "values"]
    # 这样 app.stream 会返回一个元组：(mode, chunk)
    for mode, chunk in app.stream(inputs, stream_mode=["messages", "updates", "values"]):

        # --- 模式 A: Messages (处理打字机效果) ---
        if mode == "messages":
            # chunk 结构是 (message, metadata)
            message, metadata = chunk
            # 只显示 AI 生成的内容，过滤掉系统消息等
            if message.content:
                print(message.content, end="", flush=True)

        # --- 模式 B: Updates (处理节点完成通知) ---
        elif mode == "updates":
            # chunk 是该节点刚刚更新的字段
            # 这里的 chunk 类似于Key-Value：{'writer': {'content': '...', 'status': '...'}}
            node_name = list(chunk.keys())[0]
            updates = chunk[node_name]
            print(f"\n\n✅ [节点完成] {node_name} -> 状态: {updates.get('status')}")
            if "score" in updates:
                print(f"📊 [评分结果] 得分: {updates['score']}")
            print("-" * 50)  # 分割线

        # --- 模式 C: Values (处理全局状态快照) ---
        elif mode == "values":
            # chunk 是当前的完整 State
            print(f"\n📦 [全量状态快照] {chunk}")

    print("\n🎉 流程结束！")


if __name__ == "__main__":
    run_mixed_mode_demo()
```

## 从工具中流式传输数据
```plain
from langgraph.prebuilt import create_react_agent
from langchain.chat_models import init_chat_model
from langchain_tavily import TavilySearch
from langgraph.config import get_stream_writer
from langchain_core.tools import tool
from datetime import datetime
import time
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


# 定义工具函数
@tool
def search_web(query: str):
    """搜索网络信息的工具"""
    writer = get_stream_writer()
    writer({"step": "搜索", "status": "请等待...", "progress": 0})
    time.sleep(0.5)

    writer({"step": "搜索", "status": "请等待...", "progress": 50})
    time.sleep(0.5)

    writer({
        "step": "搜索",
        "status": "完成",
        "progress": 100,
        "data": {"结果": TavilySearch().invoke(query)}
    })


@tool
def get_data_tool():
    """获取目前日期的工具"""
    return datetime.now().date()


tools = [search_web, get_data_tool]

system_prompt = """你是一个智能助手。你有以下工具可以使用：

1. search_web: 用于搜索互联网获取最新信息，特别是产品价格、新闻、实时数据等
2. get_current_date: 获取今天的日期

重要规则：
- 当用户询问产品价格、最新信息、新闻等需要实时数据的问题时，必须使用search_web工具
- 当用户询问时间或日期时，使用相应的时间工具
- 如果你的知识库中没有准确或最新的信息，应该使用搜索工具
- 优先使用工具获取准确信息，而不是依赖可能过时的训练数据

请根据用户问题选择合适的工具来获取准确答案。"""

agent = create_react_agent(model=llm,
                           tools=tools,
                           prompt=system_prompt
                           )

for chunk in agent.stream({"messages": [{"role": "user", "content": "小米yu7价格"}]}, stream_mode="custom"):
    step = chunk.get("step", "")
    status = chunk.get("status", "")
    progress = chunk.get("progress", 0)
    data = chunk.get("data", {})

    # 显示进度
    progress_bar = "█" * (progress // 10) + "░" * (10 - progress // 10)
    print(f"\n[{step}] {status}")
    print(f"进度: [{progress_bar}] {progress}%")

    # 显示额外数据
    if data:
        for key, value in data.items():
            print(f"📊 {key}: {value}")
```

## 从子图中进行流式传输
只需要在父图的方法中`stream()`设置` subgraphs=True`

输出将作为元组进行流式传输`(namespace, data)`，其中`namespace`是包含调用子图的节点路径的元组，例如`("parent_node:<task_id>", "child_node:<task_id>")`。

```plain
from langgraph.graph import StateGraph, MessagesState, START
from typing_extensions import TypedDict, Annotated
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


# 创建子图
class SubgraphMessagesState(TypedDict):
    subgraph_messages: Annotated[list[AnyMessage], add_messages]


def subplot(state: SubgraphMessagesState) -> SubgraphMessagesState:
    # 获取大模型回答的内容进行摘要总结
    answer = state["subgraph_messages"][-1].content
    summary_prompt = f"请用一句话总结下面这句话：\n\n答：{answer}"
    response = llm.invoke(summary_prompt)
    # print("\n\n")
    # print("子图中问题和输出:", state["subgraph_messages"] + [response])
    return {"subgraph_messages": [response]}


summary_subgraph = (
    StateGraph(state_schema=SubgraphMessagesState)
    .add_node("subplot", subplot)
    .add_edge(START, "subplot")
    .compile()
)


# 创建父图

def llm_answer_node(state: MessagesState) -> MessagesState:
    # 使用大模型进行回答
    answer = llm.invoke(state["messages"])
    # print("父图中问题和输出:", state["messages"] + [answer])
    # 转换状态格式
    summary_result = summary_subgraph.invoke({"subgraph_messages": state["messages"] + [answer]})

    return {"messages": state["messages"] + [answer, summary_result["subgraph_messages"][2]]}


parent_graph = (
    StateGraph(state_schema=MessagesState)
    .add_node("llm_answer", llm_answer_node)
    .add_edge(START, "llm_answer")
    .compile()
)
# 测试输入
input_state = {
    "messages": [{"role": "user", "content": "langgraph是什么？"}],
}

for chunk in parent_graph.stream(
        input_state,
        stream_mode="updates",
        subgraphs=True,
):
    print(chunk)
```

## 禁用特定聊天模型的流式传输
如果您的应用程序将支持流式传输的模型与不支持流式传输的模型混合使用，则可能需要明确禁用不支持流式传输的模型。

```plain
model = init_chat_model(    
    "anthropic:claude-3-7-sonnet-latest",    
    disable_streaming=True 
)
```

# 持久性
检查点（Checkpointing）是 LangGraph 持久性的核心机制。它允许你在图执行过程中的任何点保存状态，并在需要时恢复。

![](./images/1766296155033-a43cc82e-e4c9-46fd-9034-8b7403bdd1cf.png)

## 核心概念
+ **检查点(Checkpoint)**: 图状态的快照
+ **线程(Thread)**: 用于访问检查点的唯一标识
+ **检查点保存器(Checkpointer)**: 负责保存和恢复状态的组件

## 线程(Threads)
线程是检查点保存器保存的每个检查点分配的唯一 ID 或线程标识符

当使用检查点调用图表时，**必须**指定`thread_id`作为`configurable`配置部分的一部分：

```plain
# 调用图时必须指定 thread_id
config = {"configurable": {"thread_id": "unique_thread_id"}}
result = graph.invoke(input_data, config=config)
```

### 特点
+ 每个线程代表一个独立的对话或执行上下文
+ 线程允许在图执行后访问图的状态
+ 支持多个并发线程

```plain
from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, MessagesState, START
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


def process_message(state: MessagesState):
    response = llm.invoke(state["messages"])
    return {"messages": response}


builder = StateGraph(state_schema=MessagesState)
builder.add_node("process_message", process_message)
builder.add_edge(START, "process_message")
# # 没有使用持久性
# graph = builder.compile()
#
# input_message = {"role": "user", "content": "你好呀！我的名字叫初见"}
# for chunk in graph.stream({"messages": [input_message]}, stream_mode="values"):
#     chunk["messages"][-1].pretty_print()
#
# input_message = {"role": "user", "content": "我的名字叫什么？"}
# for chunk in graph.stream({"messages": [input_message]}, stream_mode="values"):
#     chunk["messages"][-1].pretty_print()

# 使用持久性
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)
config = {"configurable": {"thread_id": "user_123"}}

input_message = {"role": "user", "content": "你好呀！我的名字叫初见"}
for chunk in graph.stream({"messages": [input_message]}, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()

input_message = {"role": "user", "content": "我的名字叫什么？"}
for chunk in graph.stream({"messages": [input_message]}, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

## 检查点
检查点是在每个超级步骤中保存的图状态的快照，由`StateSnapshot`具有以下关键属性的对象表示：

+ `config`：与此检查点相关的配置。
+ `metadata`：与此检查点相关的元数据。
+ `values`：当前 `State` 的值。也就是图执行到目前为止，所有变量的状态值（如 `"messages"`, `"steps"`, `"results"` 等字段的值）。
+ `next`图中接下来要执行的节点名称的元组。
+ `tasks`：包含具体要执行的任务的详细信息，用 `PregelTask` 类型表示。比 `next` 更详细

### LangGraph 中检查点的作用
| **<font style="color:rgb(0, 0, 0);">功能</font>** | **<font style="color:rgb(0, 0, 0);">描述</font>** |
| --- | --- |
| <font style="color:rgb(0, 0, 0);">🌟</font><font style="color:rgb(0, 0, 0);"> 容错恢复</font> | <font style="color:rgb(0, 0, 0);">如果执行中断（如容器崩溃、任务超时），可以从上次保存的状态恢复，不用重跑整个流程</font> |
| <font style="color:rgb(0, 0, 0);">💾</font><font style="color:rgb(0, 0, 0);"> 状态追踪/审计</font> | <font style="color:rgb(0, 0, 0);">可以记录每一步节点执行时的中间状态，方便 Debug、回溯和监控</font> |
| <font style="color:rgb(0, 0, 0);">🔁</font><font style="color:rgb(0, 0, 0);"> 实现有状态的异步/长流程图</font> | <font style="color:rgb(0, 0, 0);">对于多轮对话、多阶段任务，检查点使 LangGraph 支持状态持久化和任务跟踪</font> |


### 本质理解
LangGraph 中的图是围绕 `**State**`** 状态对象** 构建的：

每一步（Node）执行时会读取 `State`，返回一个新的 `State`。

所谓的“检查点”就是：

**在某个节点运行后，把当时的 **`**State**`** 存起来**（比如存到数据库或磁盘）

然后如果下次因为任何原因中断或重新运行，只需：

**加载上次的检查点状态 **`**State**`**，重新进入图流程**

### 获取状态内容
```plain
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.checkpoint.memory import InMemorySaver
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


# 创建子图
def subplot(state: MessagesState) -> MessagesState:
    # 获取大模型回答的内容进行摘要总结
    answer = state["messages"][-1].content
    summary_prompt = f"请用一句话总结下面这句话：\n\n答：{answer}"
    response = llm.invoke(summary_prompt)
    return {"messages": state["messages"] + [response]}


summary_subgraph = (
    StateGraph(state_schema=MessagesState)
    .add_node("subplot", subplot)
    .add_edge(START, "subplot")
    .compile()
)


# 创建父图
def llm_answer_node(state: MessagesState) -> MessagesState:
    # 使用大模型进行回答
    answer = llm.invoke(state["messages"])
    print("父图输出", answer)
    return {"messages": state["messages"] + [answer]}


checkpointer = InMemorySaver()
parent_graph = (
    StateGraph(MessagesState)
    .add_node("llm_answer", llm_answer_node)
    .add_node("summarize_subgraph", summary_subgraph)
    .add_edge(START, "llm_answer")
    .add_edge("llm_answer", "summarize_subgraph")
    .compile(checkpointer=checkpointer)
)
config = {"configurable": {"thread_id": "1"}}
# 测试输入
input_state = {
    "messages": [{"role": "user", "content": "langgraph是什么？请用100字介绍"}],
}

for chunk in parent_graph.stream(
        input_state,
        config,
        stream_mode="updates",
        subgraphs=True

):
    # print(chunk)
    pass
print("================获取状态=================")
"""
与已保存的图表状态交互时，必须指定线程标识符。您可以通过调用来查看图表的最新graph.get_state(config)状态。这将返回一个StateSnapshot对象
"""
print(parent_graph.get_state(config).values)

print("================状态历史记录=================")
"""
可以通过调用 获取给定线程的图形执行的完整历史记录graph.get_state_history(config)。
这将返回与配置中提供的线程 ID 关联的对象列表StateSnapshot。
重要的是，检查点将按时间顺序排序，最新的检查点 /StateSnapshot将位于列表中的第一个。

注意：这里采用的是共享状态的子图，可以将子图的内容持久化，如果使用的是不同状态的就需要分别存储
"""
history = list(parent_graph.get_state_history(config))
for idx, snapshot in enumerate(history):
    print(f"Step {idx}:")
    print(f"  Checkpoint ID: {snapshot.config['configurable']['checkpoint_id']}")
    print(f"  Node: {snapshot.metadata.get('source')}")
    print(f"  Messages: {[m.content for m in snapshot.values['messages']]}")
    print("")

print("================重放机制=================")
"""
可以重放先前的图执行。如果使用 thread_id 和 checkpoint_id 调用图，LangGraph 会：
    1.重放检查点之前已执行的步骤（不重新执行）
    2.执行检查点之后的步骤（即使之前已执行）
注意：必须传递这些内容thread_id， checkpoint_id

重要特性：
    LangGraph 知道特定步骤是否已执行
    检查点前的步骤会被重放（不重新执行）
    检查点后的步骤会被重新执行，不包括当前检查点（创建新分支）
"""
# 获取Step2检查点，开始进行重放
step2_level_checkpoint = None
if history:
    step2_level_checkpoint = list(history)[2].config['configurable']['checkpoint_id']

print("第二步检查点：", step2_level_checkpoint)
config = {"configurable": {"thread_id": "1", "checkpoint_id": step2_level_checkpoint}}
# 开始执行
result = parent_graph.invoke(None, config=config)
print(result)
```

## 更新对应状态
使用 `graph.update_state()` 方法编辑图状态。当其中某个节点需要人为进行控制的时候，需要用更新状态来确定是否执行剩下流程。

暂时无法在飞书文档外展示此内容

**一般和人机交互一起使用，**`**update_state**`**不会启动流程**

### 方法参数
1. **config**
+ 必须包含 `thread_id` 指定要更新的线程
+ 可选包含 `checkpoint_id` 来分叉选定的检查点
1. **values**
+ 用于更新状态的值
+ 更新会传递给 reducer 函数（如果定义了）
+ 没有 reducer 的通道会被覆盖
1. **as_node**
+ 可选参数，指定更新来自哪个节点
+ 影响下一步执行的节点

```plain
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from typing_extensions import TypedDict
from typing import Annotated
from operator import add


class TaskState(TypedDict):
    task_id: str  # 任务id
    title: str  # 标题
    assignee: str  # 接收任务的人
    priority: int  # 优先级
    comments: Annotated[list, add]  # 评论
    status: str  # 状态


def create_task(state: TaskState):
    """创建任务"""
    return {
        "status": "进行中",
        "comments": [f"任务 '{state['title']}' 已创建，分配给 {state['assignee']}"]
    }


def update_task(state: TaskState):
    """更新任务"""
    return {
        "status": "已更新",
        "comments": ["任务状态已更新"]
    }


def plan_task(state: TaskState):
    """准备任务完成"""
    return {
        "status": "已准备",
        "comments": [f"任务 '{state['title']}' 已准备"]
    }


# 创建任务管理流程
workflow = StateGraph(state_schema=TaskState)
workflow.add_node("create", create_task)
workflow.add_node("update", update_task)
workflow.add_node("complete", plan_task)
workflow.add_edge(START, "create")
workflow.add_edge("update", "complete")
workflow.add_edge("complete", END)

checkpointer = InMemorySaver()
app = workflow.compile(checkpointer=checkpointer)

# 创建任务
config = {"configurable": {"thread_id": "task_001"}}
result = app.invoke({
    "task_id": "T001",
    "title": "开发注册功能",
    "assignee": "张三",
    "priority": 1,
    "comments": [],
    "status": "待分配"
}, config)

print("=== 任务创建完成 ===")
print(f"状态: {result['status']}")
print("评论:", result['comments'])

# 演示 as_node 参数, update节点需要手动使用update_state进行触发， 才能执行后续操作
print("\n=== 使用 as_node 参数 ===")
app.update_state(config, {
    "status": "已更新",
    "comments": ["自动通知：任务优先级提升"],
    "priority": 3
}, as_node="update")
# 当手动修改了某个状态，想重新启动工作流需要手动调用invoke
final_result = app.invoke(None, config)
# 查看最终状态
final_state = app.get_state(config)
print("最终状态:", final_result["status"])
print("完整评论:", final_result["comments"])

# 手动更新状态 - 添加评论
print("\n=== 手动添加评论 ===")
app.update_state(config, {
    "comments": ["项目经理：请在周五前完成"],
    "priority": 2
})

# 查看更新后的状态
updated_state = app.get_state(config)
print(f"优先级: {updated_state.values['priority']}")
print("所有评论:", updated_state.values['comments'])

# 继续更新 - 添加更多评论
print("\n=== 添加更多评论 ===")
app.update_state(config, {
    "comments": ["张三：已完成开发"],
    "status": "开发完成"
})

final_state = app.get_state(config)
print(f"最终状态: {final_state.values['status']}")
print("完整评论历史:", final_state.values['comments'])
```

## 记忆存储
![](./images/1766296154950-8d9d9eb3-dc97-41b9-be0f-d880a206a23e.png)

状态模式指定在图执行时填充的键集合。但如果我们想在线程之间保留信息怎么办？

### Store 接口
+ 检查点保存器单独无法跨线程共享信息
+ Store 接口解决了这个问题
+ 可以在所有聊天对话中保留用户特定信息

### 基础用法
每种内存类型都是一个具有特定属性的 Python 类（`Item`）。我们可以通过上述转换将其作为字典访问`.dict`。它具有以下属性：

+ `value`：此内存的值（本身就是一个字典）
+ `key`：此命名空间中此内存的唯一键
+ `namespace`：字符串列表，此内存类型的命名空间
+ `created_at`：此内存创建的时间戳
+ `updated_at`：此内存更新的时间戳

```plain
print("-" * 8, "基础用法", "-" * 8)
from langgraph.store.memory import InMemoryStore
import uuid

# 创建存储
# in_memory_store = InMemoryStore()
#
# # 定义命名空间
# namespace_for_memory = ("user_id", "memories")
#
# # 存储记忆
# memory_id = str(uuid.uuid4())
# memory = {"hobby": "篮球、音乐、美食、编程..."}
# in_memory_store.put(namespace_for_memory, memory_id, memory)
#
# # 搜索记忆
# memories = in_memory_store.search(namespace_for_memory)
# # 打印数据
# print(memories[-1].dict())

print("-" * 8, "语义搜索", "-" * 8)
from langchain_huggingface import HuggingFaceEmbeddings

namespace_for_memory = ("user_id", "memories")

store = InMemoryStore(
    index={
        "embed": HuggingFaceEmbeddings(model_name=r"D:\llm\Local_model\BAAI\bge-large-zh-v1___5"),
        "dims": 1024,
        "fields": ["hobby", "food_preference"]
    }
)

# 3. 存储数据并检查
memory_id_1 = str(uuid.uuid4())
memory_1 = {"hobby": "我的爱好是：篮球、音乐、美食、编程..."}
store.put(namespace_for_memory, memory_id_1, memory_1)
print(f"✓ 存储 hobby 记忆: {memory_id_1}")

memory_id_2 = str(uuid.uuid4())
memory_2 = {"food_preference": "我最喜欢的美食是：臭豆腐、小龙虾、红烧肉..."}
store.put(namespace_for_memory, memory_id_2, memory_2)
print(f"✓ 存储 food_preference 记忆: {memory_id_2}")

# 4. 检查存储的数据
print("\n=== 调试信息 ===")
print(f"Namespace: {namespace_for_memory}")
print(f"存储的记忆数量: {len(store.search(namespace_for_memory))}")

# 5. 搜索测试
print("\n=== 搜索测试 ===")

# 测试 1: 搜索食物偏好
print("搜索: 用户喜欢吃什么？")
memories = store.search(
    namespace_for_memory,
    query="用户喜欢吃什么？",
    limit=3
)
print(f"搜索结果数量: {len(memories)}")
if memories:
    print(f"最相关结果: {memories[0].dict()}")
else:
    print("没有找到结果")

# 测试 2: 搜索爱好
print("\n搜索: 用户的爱好有哪些？")
memories = store.search(
    namespace_for_memory,
    query="用户的爱好有哪些？",
    limit=3
)
print(f"搜索结果数量: {len(memories)}")
if memories:
    print(f"最相关结果: {memories[0].dict()}")
else:
    print("没有找到结果")
```

### langgraph中使用存储功能
```plain
from typing import Annotated, List
from typing_extensions import TypedDict
from operator import add
import uuid
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore
from langchain_core.messages import HumanMessage, AIMessage, BaseMessage
from langchain_core.runnables import RunnableConfig
from langgraph.store.base import BaseStore

from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

# 初始化大模型
llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


# 定义状态结构
class MessagesState(TypedDict):
    messages: Annotated[List[BaseMessage], add]


# 创建检查点保存器和内存存储
checkpointer = InMemorySaver()
in_memory_store = InMemoryStore()


# 聊天机器人节点  *代表后面的参数必须使用显示写出参数名称  store=in_memory_store
def chatbot(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    """主聊天机器人节点，处理用户消息并生成回复"""

    # 获取用户ID和最新消息
    user_id = config["configurable"]["user_id"]
    last_message = state["messages"][-1]

    # 定义内存命名空间
    namespace = (user_id, "memories")

    # 简单的聊天逻辑
    user_input = last_message.content.lower()
    # 将聊天历史获取并组装提示词
    memories = store.search(namespace)
    memory_text = "\n".join(m.value["memory"] for m in memories)
    prompt = f"请参考聊天记录：{memory_text}\n\nHuman: {user_input}\nAI:"

    response = llm.invoke(prompt).content

    # 存储对应的问题和答案
    memory = f"问题：{user_input} --- 答案:{response}"
    memory_id = str(uuid.uuid4())
    store.put(namespace, memory_id, {"memory": memory})
    # 返回AI消息
    return {"messages": [AIMessage(content=response)]}


# 创建图
def create_persistent_graph():
    """创建持久化的聊天机器人图"""

    # 创建状态图
    workflow = StateGraph(MessagesState)
    # 添加节点
    workflow.add_node("chatbot", chatbot)
    # 添加边
    workflow.add_edge(START, "chatbot")
    workflow.add_edge("chatbot", END)

    # 编译图，使用检查点保存器和存储
    graph = workflow.compile(checkpointer=checkpointer, store=in_memory_store)

    return graph


# 工具函数：显示状态历史
def show_state_history(graph, config):
    """显示状态历史"""
    print("\n=== 状态历史 ===")
    history = graph.get_state_history(config)
    for i, snapshot in enumerate(history):
        print(f"\n步骤 {i}:")
        print(f"  配置: {snapshot.config}")
        print(f"  值: {snapshot.values}")
        print(f"  下一步: {snapshot.next}")
        print(f"  元数据: {snapshot.metadata}")


# 工具函数：显示存储的记忆
def show_memories(store, user_id):
    """显示用户的所有记忆"""
    print(f"\n=== 用户 {user_id} 的记忆 ===")
    namespace = (user_id, "memories")
    memories = store.search(namespace)

    if memories:
        for memory in memories:
            print(f"记忆ID: {memory.key}")
            print(f"内容: {memory.value}")
            print(f"创建时间: {memory.created_at}")
            print(f"更新时间: {memory.updated_at}")
            print("---")
    else:
        print("没有找到记忆")


# 主程序
def main():
    # 创建图
    graph = create_persistent_graph()

    # 用户配置
    user_id = "user_123"
    thread_id = "conversation_1"

    config = {
        "configurable": {
            "thread_id": thread_id,
            "user_id": user_id
        }
    }

    print("=== LangGraph 持久化聊天机器人 ===")
    print("输入 'quit' 退出，'history' 查看状态历史，'memories' 查看记忆")

    while True:
        user_input = input("\n用户: ").strip()

        if user_input.lower() == 'quit':
            break
        elif user_input.lower() == 'history':
            show_state_history(graph, config)
            continue
        elif user_input.lower() == 'memories':
            show_memories(in_memory_store, user_id)
            continue

        # 创建用户消息
        initial_state = {
            "messages": [HumanMessage(content=user_input)]
        }

        # 运行图
        try:
            result = graph.invoke(initial_state, config)

            # 显示AI回复
            ai_message = result["messages"][-1]
            print(f"AI: {ai_message.content}")

        except Exception as e:
            print(f"错误: {e}")

    print("\n=== 最终状态 ===")
    final_state = graph.get_state(config)
    print(f"最终状态: {final_state.values}")

    print("\n=== 所有记忆 ===")
    show_memories(in_memory_store, user_id)


# 演示不同线程间的记忆共享
def demo_cross_thread_memory():
    """演示跨线程记忆共享"""
    print("\n=== 跨线程记忆共享演示 ===")

    graph = create_persistent_graph()
    user_id = "user_456"

    # 第一个对话线程
    config1 = {
        "configurable": {
            "thread_id": "thread_1",
            "user_id": user_id
        }
    }

    print("线程1 - 建立记忆:")
    result1 = graph.invoke({
        "messages": [HumanMessage(content="我叫Alice，我喜欢音乐")]
    }, config1)
    print(f"AI: {result1['messages'][-1].content}")

    # 第二个对话线程（相同用户）
    config2 = {
        "configurable": {
            "thread_id": "thread_2",
            "user_id": user_id
        }
    }

    print("\n线程2 - 访问记忆:")
    result2 = graph.invoke({
        "messages": [HumanMessage(content="你还记得我吗？")]
    }, config2)
    print(f"AI: {result2['messages'][-1].content}")

    # 显示共享的记忆
    show_memories(in_memory_store, user_id)


if __name__ == "__main__":
    # 运行主程序
    main()

    # 演示跨线程记忆共享
    demo_cross_thread_memory()
```

InMemorySaver和InMemoryStore的使用场景：

InMemorySaver短期记忆：存储每个节点执行完成之后的状态（聊天历史）

InMemoryStore长期记忆：跨会话（线程），存储用户相关的内容

# 记忆
对于人工智能代理来说，记忆至关重要，因为它能让它们记住之前的交互，从反馈中学习，并适应用户的偏好。随着代理需要处理更复杂的任务，并进行大量的用户交互，这种能力对于效率和用户满意度都至关重要。

+ 短期记忆（或线程范围的记忆）通过维护会话中的消息历史记录来跟踪正在进行的对话。LangGraph 将短期记忆作为代理状态的一部分进行管理。状态使用检查点持久化到数据库中，以便线程可以随时恢复。短期记忆会在图被调用或某个步骤完成时更新，并且在每个步骤开始时读取状态。
+ 长期记忆跨会话存储用户特定或应用程序级别的数据，并在对话线程之间共享。它可以在任何时间、任何线程中调用。记忆的作用域是任何自定义命名空间，而不仅仅是单个线程 ID。LangGraph 提供存储，方便您保存和调用长期记忆。

![](./images/1766296154997-8233dd87-1072-410b-a7d8-2a6f8c30b5f2.png)

## 短期记忆
**短期记忆**（线程级持久性）使代理能够跟踪多轮对话。在持久性中已经讲过简单的线程级的短期记忆。

### 1.使用redis作为存储
```plain
pip install -U langgraph-checkpoint-redis==0.0.8
```

```plain
import os
from dotenv import load_dotenv
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.redis import RedisSaver
from redis import Redis, ConnectionPool

# 加载环境变量
load_dotenv()

# 1. 初始化模型
# 建议：将 temperature 设置为 0.7 让回答更自然，或者 0 让回答更稳定
llm = init_chat_model(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    model_provider="openai",
    model='qwen-plus-2025-07-28',  # 建议使用通用别名，除非你必须锁定特定日期版本
    temperature=0.7
)

# 2. 优化 Redis 连接：使用连接池
# 这样可以避免每次运行都建立新连接，也不需要把代码全部缩进在 with 块里
pool = ConnectionPool.from_url("redis://localhost:6379", max_connections=10)
# Redis 客户端接受 connection_pool 参数
redis_client = Redis(connection_pool=pool)
checkpointer = RedisSaver(redis_client=redis_client)


# 3. 定义节点逻辑
def call_llm(state: MessagesState):
    # 可以在这里添加 System Prompt 逻辑
    # system_msg = SystemMessage(content="你是一个幽默的助手")
    # messages = [system_msg] + state["messages"]
    response = llm.invoke(state["messages"])
    return {"messages": [response]}  # 注意：返回列表，LangGraph 会自动 append


# 4. 构建图
def build_graph():
    builder = StateGraph(MessagesState)
    builder.add_node("chatbot", call_llm)
    builder.add_edge(START, "chatbot")
    builder.add_edge("chatbot", END)  # 显式添加 END 边，逻辑更清晰

    # 编译时传入 checkpointer
    return builder.compile(checkpointer=checkpointer)


# 5. 封装运行逻辑
def run_chat(thread_id: str):
    graph = build_graph()
    config = {"configurable": {"thread_id": thread_id}}

    print(f"--- 开始对话 (Thread ID: {thread_id}) ---")

    while True:
        user_input = input("User: ")
        if user_input.lower() in ["q", "quit", "exit"]:
            print("再见！")
            break

        # 这样只返回“最新发生变化”的部分，而不是整个历史记录
        events = graph.stream(
            {"messages": [{"role": "user", "content": user_input}]},
            config,
            stream_mode="updates"
        )

        for event in events:
            # event 的格式是 {'node_name': {'messages': [AIMessage]}}
            for node_name, values in event.items():
                if "messages" in values:
                    last_msg = values["messages"][-1]
                    print(f"AI: {last_msg.content}")


if __name__ == "__main__":
    # 第一次运行会建立 Redis 结构
    checkpointer.setup()

    # 运行聊天
    run_chat(thread_id="user_123")
```

### **2.使用****Postgres****存储**
下载postgresSql，数据库存储是的加密数据。

```plain
# 1.使用docker下载对应镜像
docker pull postgres:alpine # 这边使用的是体积更小的镜像

# 2.运行对应镜像
docker run -id --name=postgresql -v postgre-data:/var/lib/postgresql/data -p 5432:5432 -e POSTGRES_PASSWORD=123456 -e LANG=C.UTF-8 postgres:alpine
```

```plain
pip install -U "psycopg[binary,pool]" langgraph-checkpoint-postgres==2.0.23
```

```plain
import os
import uuid
import logging
import sys
from dotenv import load_dotenv
from langchain.chat_models import init_chat_model
from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.store.postgres import PostgresStore
from langgraph.store.base import BaseStore
from psycopg_pool import ConnectionPool

# 配置日志：只显示 INFO，避免太吵
logging.basicConfig(level=logging.ERROR)
logger = logging.getLogger(__name__)

load_dotenv()

# ==========================================
# 1. 初始化资源
# ==========================================
try:
    llm = init_chat_model(
        api_key=os.getenv("DASHSCOPE_API_KEY"),
        base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
        model_provider="openai",
        model='qwen-plus-2025-07-28',
        temperature=0.7
    )

    DB_URI = "postgresql://postgres:123456@localhost:5432/postgres?sslmode=disable"
    pool = ConnectionPool(conninfo=DB_URI, max_size=10, kwargs={"autocommit": True})

    store = PostgresStore(pool)
    checkpointer = PostgresSaver(conn=pool)
    # 创建表结构
    store.setup()
    checkpointer.setup()
    print("✅ 数据库连接成功，记忆系统已就绪。")

except Exception as e:
    print(f"❌ 初始化失败: {e}")
    sys.exit(1)


# ==========================================
# 2. 核心图逻辑
# ==========================================
def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    user_id = config["configurable"]["user_id"]
    namespace = ("memories", user_id)

    last_message = state["messages"][-1]
    last_content = last_message.content

    # --- 🧠 写入记忆 ---
    if "记住" in last_content:
        # 简单提取：去掉关键词
        memory_content = last_content.replace("记住", "").replace("：", "").replace(":", "").strip()
        if memory_content:
            memory_id = str(uuid.uuid4())
            store.put(namespace, memory_id, {"data": memory_content})
            print(f"   💾 [系统提示] 已将信息写入长期记忆库: '{memory_content}'")

    # --- 🔍 读取记忆 ---
    # 搜索最近的 3 条相关记忆
    memories = store.search(namespace, query=last_content, limit=3)

    memory_context = ""
    if memories:
        memory_list = [f"- {m.value['data']}" for m in memories]
        memory_context = "\n".join(memory_list)
        print(f"   📖 [系统提示] 检索到相关长期记忆: {memory_list}")

    # --- 🤖 生成回复 ---
    system_prompt = "你是一个智能助手。"
    if memory_context:
        system_prompt += f"\n\n【长期记忆库】:\n{memory_context}\n\n请结合记忆回答用户。"

    messages = [{"role": "system", "content": system_prompt}] + state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response]}


def build_graph():
    builder = StateGraph(MessagesState)
    builder.add_node("chatbot", call_model)
    builder.add_edge(START, "chatbot")
    builder.add_edge("chatbot", END)
    return builder.compile(checkpointer=checkpointer, store=store)


# ==========================================
# 3. 交互式主程序
# ==========================================
def main():
    graph = build_graph()

    # 默认初始身份
    current_user = "user_001"
    current_thread = "thread_1"

    print("\n" + "=" * 60)
    print("🤖 LangGraph 记忆系统演示终端")
    print("=" * 60)
    print("💡 使用指南:")
    print("   1. 输入 '记住：xxx' 让 AI 存入长期记忆")
    print("   2. 输入 '/new' 开启新对话 (Thread ID 变更，短期记忆清空，长期记忆保留)")
    print("   3. 输入 '/user <id>' 切换用户 (User ID 变更，完全隔离的新记忆)")
    print("   4. 输入 '/info' 查看当前 ID 信息")
    print("   5. 输入 'q' 或 'exit' 退出")
    print("=" * 60)

    while True:
        # 显示当前状态栏
        status_bar = f"\n📍 当前状态 [用户: {current_user}] | [对话: {current_thread}]"
        print(status_bar)
        user_input = ""
        try:
            user_input = input("👤 你: ").strip()
        except EOFError:
            break

        if not user_input:
            continue

        # --- 指令处理 ---
        if user_input.lower() in ["q", "exit", "quit"]:
            print("👋 再见！")
            break

        elif user_input.lower() == "/new":
            # 生成一个新的随机 Thread ID
            current_thread = f"thread_{uuid.uuid4().hex[:8]}"
            print(f"🔄 已切换到新对话窗口 (ID: {current_thread})。短期上下文已重置。")
            continue

        elif user_input.lower().startswith("/user"):
            parts = user_input.split()
            if len(parts) > 1:
                current_user = parts[1]
                # 切换用户通常也意味着开启新对话
                current_thread = f"thread_{uuid.uuid4().hex[:8]}"
                print(f"👤 已切换用户身份为: {current_user}")
            else:
                print("⚠️ 用法: /user <new_user_id>")
            continue

        elif user_input.lower() == "/info":
            # 仅显示信息，不发送给 LLM
            continue

        # --- 发送给 AI ---
        config = {
            "configurable": {
                "thread_id": current_thread,
                "user_id": current_user
            }
        }

        print("🤖 AI: ", end="", flush=True)
        try:
            # 使用 stream 获取流式输出
            events = graph.stream(
                {"messages": [{"role": "user", "content": user_input}]},
                config,
                stream_mode="updates"
            )
            for event in events:
                if 'chatbot' in event:
                    print(event['chatbot']['messages'][-1].content)
        except Exception as e:
            print(f"\n❌ 发生错误: {e}")

    pool.close()


if __name__ == "__main__":
    main()
```

## 长期记忆
使用长期记忆来存储对话中特定于用户或特定于应用程序的数据。

### 1.**使用****Postgres****存储**
```plain
import os
import uuid
import logging
import sys
from dotenv import load_dotenv
from langchain.chat_models import init_chat_model
from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.store.postgres import PostgresStore
from langgraph.store.base import BaseStore
from psycopg_pool import ConnectionPool

# 配置日志：只显示 INFO，避免太吵
logging.basicConfig(level=logging.ERROR)
logger = logging.getLogger(__name__)

load_dotenv()

# ==========================================
# 1. 初始化资源
# ==========================================
try:
    llm = init_chat_model(
        api_key=os.getenv("DASHSCOPE_API_KEY"),
        base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
        model_provider="openai",
        model='qwen-plus-2025-07-28',
        temperature=0.7
    )

    DB_URI = "postgresql://postgres:123456@localhost:5432/postgres?sslmode=disable"
    pool = ConnectionPool(conninfo=DB_URI, max_size=10, kwargs={"autocommit": True})

    store = PostgresStore(pool)
    checkpointer = PostgresSaver(conn=pool)

    store.setup()
    checkpointer.setup()
    print("✅ 数据库连接成功，记忆系统已就绪。")

except Exception as e:
    print(f"❌ 初始化失败: {e}")
    sys.exit(1)


# ==========================================
# 2. 核心图逻辑
# ==========================================
def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    user_id = config["configurable"]["user_id"]
    namespace = ("memories", user_id)

    last_message = state["messages"][-1]
    last_content = last_message.content

    # --- 🧠 写入记忆 ---
    if "记住" in last_content:
        # 简单提取：去掉关键词
        memory_content = last_content.replace("记住", "").replace("：", "").replace(":", "").strip()
        if memory_content:
            memory_id = str(uuid.uuid4())
            store.put(namespace, memory_id, {"data": memory_content})
            print(f"   💾 [系统提示] 已将信息写入长期记忆库: '{memory_content}'")

    # --- 🔍 读取记忆 ---
    # 搜索最近的 3 条相关记忆
    memories = store.search(namespace, query=last_content, limit=3)

    memory_context = ""
    if memories:
        memory_list = [f"- {m.value['data']}" for m in memories]
        memory_context = "\n".join(memory_list)
        print(f"   📖 [系统提示] 检索到相关长期记忆: {memory_list}")

    # --- 🤖 生成回复 ---
    system_prompt = "你是一个智能助手。"
    if memory_context:
        system_prompt += f"\n\n【长期记忆库】:\n{memory_context}\n\n请结合记忆回答用户。"

    messages = [{"role": "system", "content": system_prompt}] + state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response]}


def build_graph():
    builder = StateGraph(MessagesState)
    builder.add_node("chatbot", call_model)
    builder.add_edge(START, "chatbot")
    builder.add_edge("chatbot", END)
    return builder.compile(checkpointer=checkpointer, store=store)


# ==========================================
# 3. 交互式主程序
# ==========================================
def main():
    graph = build_graph()

    # 默认初始身份
    current_user = "user_001"
    current_thread = "thread_1"

    print("\n" + "=" * 60)
    print("🤖 LangGraph 记忆系统演示终端")
    print("=" * 60)
    print("💡 使用指南:")
    print("   1. 输入 '记住：xxx' 让 AI 存入长期记忆")
    print("   2. 输入 '/new' 开启新对话 (Thread ID 变更，短期记忆清空，长期记忆保留)")
    print("   3. 输入 '/user <id>' 切换用户 (User ID 变更，完全隔离的新记忆)")
    print("   4. 输入 '/info' 查看当前 ID 信息")
    print("   5. 输入 'q' 或 'exit' 退出")
    print("=" * 60)

    while True:
        # 显示当前状态栏
        status_bar = f"\n📍 当前状态 [用户: {current_user}] | [对话: {current_thread}]"
        print(status_bar)
        user_input = ""
        try:
            user_input = input("👤 你: ").strip()
        except EOFError:
            break

        if not user_input:
            continue

        # --- 指令处理 ---
        if user_input.lower() in ["q", "exit", "quit"]:
            print("👋 再见！")
            break

        elif user_input.lower() == "/new":
            # 生成一个新的随机 Thread ID
            current_thread = f"thread_{uuid.uuid4().hex[:8]}"
            print(f"🔄 已切换到新对话窗口 (ID: {current_thread})。短期上下文已重置。")
            continue

        elif user_input.lower().startswith("/user"):
            parts = user_input.split()
            if len(parts) > 1:
                current_user = parts[1]
                # 切换用户通常也意味着开启新对话
                current_thread = f"thread_{uuid.uuid4().hex[:8]}"
                print(f"👤 已切换用户身份为: {current_user}")
            else:
                print("⚠️ 用法: /user <new_user_id>")
            continue

        elif user_input.lower() == "/info":
            # 仅显示信息，不发送给 LLM
            continue

        # --- 发送给 AI ---
        config = {
            "configurable": {
                "thread_id": current_thread,
                "user_id": current_user
            }
        }

        print("🤖 AI: ", end="", flush=True)
        try:
            # 使用 stream 获取流式输出
            events = graph.stream(
                {"messages": [{"role": "user", "content": user_input}]},
                config,
                stream_mode="updates"
            )
            for event in events:
                if 'chatbot' in event:
                    print(event['chatbot']['messages'][-1].content)
        except Exception as e:
            print(f"\n❌ 发生错误: {e}")

    pool.close()


if __name__ == "__main__":
    main()
```

### 2.**使用****Redis****存储**
```plain
from langchain_core.runnables import RunnableConfig
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.checkpoint.redis import RedisSaver
from langgraph.store.redis import RedisStore
from langgraph.store.base import BaseStore
import uuid
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')

DB_URI = "redis://localhost:6379"

with (
    RedisStore.from_conn_string(DB_URI) as store,
    RedisSaver.from_conn_string(DB_URI) as checkpointer,
):
    # 第一次使用postgres存储是需要调用初始化
    store.setup()
    checkpointer.setup()


    def call_model(
            state: MessagesState,
            config: RunnableConfig,
            *,
            store: BaseStore,
    ):
        user_id = config["configurable"]["user_id"]
        namespace = ("memories", user_id)
        # 获取记忆
        memories = store.search(namespace, query=str(state["messages"][-1].content))
        info = "\n".join([d.value["data"] for d in memories])
        system_msg = f"你是一个与用户交谈的有帮助的助手. 用户信息: {info}"

        response = llm.invoke(
            [{"role": "system", "content": system_msg}] + state["messages"]
        )
        
        # 存储新的记忆，如果用户要求模型记住
        last_message = state["messages"][-1]
        if "记住" in last_message.content.lower():
            memory = "用户名是初见"
            # 存储记忆
            store.put(namespace, str(uuid.uuid4()), {"data": memory})
        return {"messages": response}


    # 创建图
    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    builder.add_edge(START, "call_model")

    graph = builder.compile(
        checkpointer=checkpointer,
        store=store,
    )

    config = {
        "configurable": {
            "thread_id": "1",
            "user_id": "1",
        }
    }
    for chunk in graph.stream(
            {"messages": [{"role": "user", "content": "嗨！记住：我的名字是初见"}]},
            config,
            stream_mode="values",
    ):
        chunk["messages"][-1].pretty_print()

    # 使用第二个线程去访问用户信息
    config = {
        "configurable": {
            "thread_id": "2",
            "user_id": "1",
        }
    }

    for chunk in graph.stream(
            {"messages": [{"role": "user", "content": "我的名字是什么？"}]},
            config,
            stream_mode="values",
    ):
        chunk["messages"][-1].pretty_print()
```

### 具有语义搜索的记忆
```plain
from langchain_huggingface import HuggingFaceEmbeddings
from langchain.chat_models import init_chat_model
from langgraph.store.base import BaseStore
from langgraph.store.memory import InMemoryStore
from langgraph.graph import START, MessagesState, StateGraph
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')

# 加载嵌入模型
embeddings = HuggingFaceEmbeddings(model_name=r"D:\llm\Local_model\BAAI\bge-large-zh-v1___5")
store = InMemoryStore(
    index={
        "embed": embeddings,
        "dims": 1024,
    }
)

store.put(("memories", "user_123"), "1", {"text": "我喜欢吃披萨"})
store.put(("memories", "user_123"), "2", {"text": "我喜欢吃红烧肉"})
store.put(("memories", "user_123"), "3", {"text": "我的职业是程序员"})

def chat(state, *, store: BaseStore):
    # 根据用户的最后一条消息进行搜索
    items = store.search(
        ("memories","user_123"), query=state["messages"][-1].content, limit=2
    )
    print(items)
    memories = "\n".join(item.value["text"] for item in items)
    memories = f"## 用户记忆\n{memories}" if memories else ""
    response = llm.invoke(
        [
            {"role": "system", "content": f"你是一个乐于助人的助手.\n{memories}"},
    
        ] + state["messages"]
    )
    return {"messages": [response]}


builder = StateGraph(MessagesState)
builder.add_node(chat)
builder.add_edge(START, "chat")
graph = builder.compile(store=store)

for message, metadata in graph.stream(
    input={"messages": [{"role": "user", "content": "我饿了？"}]},
    stream_mode="messages",
):
    print(message.content, end="")
```

## 管理短期记忆
启用短期记忆后，长对话可能会超出 LLM 的上下文窗口。常见的解决方案如下：

+ 修剪消息：删除前 N 条或后 N 条消息（在调用 LLM 之前）
+ 从 LangGraph 状态中永久删除消息
+ 总结消息：总结历史记录中较早的消息，并用摘要替换它们
+ 管理检查点以存储和检索消息历史记录
+ 自定义策略（例如，消息过滤等）

### 修剪消息
大多数 LLM 都有一个最大支持的上下文窗口（以 token 为单位）。决定何时截断消息的一种方法是计算消息历史记录中的 token 数量，并在接近该限制时进行截断。

```plain
from langchain_core.messages.utils import (
    trim_messages,
    count_tokens_approximately
)
from langgraph.checkpoint.memory import InMemorySaver
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, MessagesState
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


def call_llm(state: MessagesState):
    messages = trim_messages(
        state["messages"],
        strategy="last",  # 修剪策略（last从末尾，first从开头， middle从中间）
        token_counter=count_tokens_approximately,  # 用来估算token数量
        max_tokens=50,  # 修剪后的消息总 token 不超过 200
        start_on="human",  # 控制修剪的起始消息类型（确保修剪后以human消息开始）
        end_on=("human", "tool"),  # 允许哪些角色作为修剪终点
    )
    print(f"修剪后消息数量: {len(messages)}")
    print(f"修剪后总tokens: {count_tokens_approximately(messages)}")
    print("修剪后的消息:")
    for i, msg in enumerate(messages):
        print(f"  {i}: {msg.type} - {msg.content[:50]}...")
    print("\n" + "=" * 50 + "\n")

    response = llm.invoke(messages)
    return {"messages": [response]}


checkpointer = InMemorySaver()
builder = StateGraph(MessagesState)
builder.add_node(call_llm)
builder.add_edge(START, "call_llm")
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "1"}}
graph.invoke({"messages": "我的名字叫初见"}, config)
graph.invoke({"messages": "帮我家的猫写一首诗"}, config)
graph.invoke({"messages": "现在对狗做一样的事情"}, config)
final_response = graph.invoke({"messages": "我的名字叫什么?"}, config)

# print(final_response)
# final_response["messages"][-1].pretty_print()
```

### 删除消息
可以从图表状态中删除消息，以管理消息历史记录。当您想要移除特定消息或清除整个消息历史记录时，此功能非常有用。

```plain
from langchain_core.messages import RemoveMessage
from langgraph.graph import StateGraph, START, MessagesState
from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


def delete_messages(state):
    messages = state["messages"]
    if len(messages) > 2:
        # 删除最早的两条消息
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}
    return None


def call_llm(state: MessagesState):
    response = llm.invoke(state["messages"])
    return {"messages": response}


builder = StateGraph(MessagesState)
builder.add_sequence([call_llm, delete_messages])
builder.add_edge(START, "call_llm")

checkpointer = InMemorySaver()
app = builder.compile(checkpointer=checkpointer)

config = {
    "configurable": {
        "thread_id": "1"
    }
}

for event in app.stream(
        {"messages": [{"role": "user", "content": "你好呀，我是初见哦"}]},
        config,
        stream_mode="values"
):
    print([(message.type, message.content) for message in event["messages"]])

for event in app.stream(
        {"messages": [{"role": "user", "content": "我的名字是什么？"}]},
        config,
        stream_mode="values"
):
    # 最终回复会把最开始的两条消息删除
    print([(message.type, message.content) for message in event["messages"]])
```

### 总结消息
修剪或删除消息的问题在于，可能会因剔除消息队列而丢失信息。因此，一些应用程序受益于一种更复杂的方法，即使用聊天模型来汇总消息历史记录。

![](./images/1766296155022-37e0e3c7-4194-4472-b7ae-0b2b0fbbe1dc.png)

下载模块

```plain
pip install langmem
```

```plain
from typing import Any, TypedDict
from langchain_core.prompts.chat import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.messages import AnyMessage
from langchain_core.messages.utils import count_tokens_approximately
from langgraph.graph import StateGraph, START, MessagesState
from langgraph.checkpoint.memory import InMemorySaver
from langmem.short_term import SummarizationNode, RunningSummary
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')

# 创建一个专用于摘要的模型实例，限制输出最多 128 tokens
summarization_model = llm.bind(max_tokens=128)


# 定义状态结构，包含对话历史和摘要上下文
class State(MessagesState):
    context: dict[str, RunningSummary]  # 用于存储用户摘要记忆（running_summary）


# 定义输入格式，传给 call_model 函数使用
class LLMInputState(TypedDict):
    summarized_messages: list[AnyMessage]  # 已被压缩/摘要过的消息
    context: dict[str, RunningSummary]


# 首次生成摘要
initial_summary_prompt = ChatPromptTemplate.from_template(
    """请阅读以下对话内容，并生成一个简洁的摘要，用于帮助理解对话的主要内容：

对话内容：
{messages}

摘要："""
)

# 在已有摘要基础上追加新的对话内容，更新摘要
existing_summary_prompt = ChatPromptTemplate.from_template(
    """你之前已经生成了如下摘要：
{existing_summary}

现在，对话继续发展了，请根据新增的对话内容，更新这个摘要，使其覆盖所有关键内容。

新增对话内容：
{messages}

更新后的摘要："""
)

# 用于最终调用模型之前，将摘要和剩余消息一起传入
final_prompt = ChatPromptTemplate.from_template(
    """你是一位智能助理。以下是用户和你的对话摘要，可帮助你快速理解上下文：

摘要：
{summary}

这是对话中未被总结的新消息，请继续处理这些信息：
{messages}
"""
)
# 创建摘要节点：超过一定 token 数时会对历史消息自动进行摘要
summarization_node = SummarizationNode(
    token_counter=count_tokens_approximately,  # 使用近似 token 计算
    model=summarization_model,  # 使用绑定了 max_tokens 的模型
    max_tokens=200,  # 在进行摘要之前，传给模型的输入上下文的最大token长度限制
    max_tokens_before_summary=150,  # 超过这个数就会触发摘要
    max_summary_tokens=128,  # 每次摘要最多保留 128 tokens
    initial_summary_prompt=initial_summary_prompt,  # 首次生成摘要的提示词
    existing_summary_prompt=existing_summary_prompt,  # 更新摘要的提示词
    final_prompt=final_prompt  # 模型回答问题之前参考的摘要上下文的提示词
)


# 模型调用节点：对压缩过的历史消息进行问答
def call_llm(state: LLMInputState):
    response = llm.invoke(state["summarized_messages"])
    return {
        "messages": [response],
        "context": state.get("context", {})  # 把上下文原样返回，里面就有摘要
    }


# 使用内存存储器（可换成 Redis/Postgres）
checkpointer = InMemorySaver()

# 构建 LangGraph 的流程图
builder = StateGraph(State)

# 添加两个节点：摘要节点 和 模型调用节点
builder.add_node(call_llm)
builder.add_node("summarize", summarization_node)

# 定义边：从 START 开始 → 先摘要 → 再模型调用
builder.add_edge(START, "summarize")
builder.add_edge("summarize", "call_llm")

# 编译图
graph = builder.compile(checkpointer=checkpointer)

# ========== 流程调用 ==========
config = {"configurable": {"thread_id": "1"}}  # 每个线程维护一个上下文

# 第1轮：告诉模型「我叫小明」
graph.invoke({"messages": "你好，我叫初见"}, config)

# 第2轮：要求写一首猫的诗
graph.invoke({"messages": "请写一首关于猫的诗"}, config)

# 第3轮：让它对狗做一样的事
graph.invoke({"messages": "现在也请为狗写一首诗"}, config)

# 第4轮：问它「我叫什么名字？」
final_response = graph.invoke({"messages": "你还记得我叫什么名字吗？"}, config)

# 输出最终回复
final_response["messages"][-1].pretty_print()

# 输出摘要内容（短期记忆）
print("\n摘要记忆内容（summary）:", final_response)
```

# 工具
**工具**封装了可调用函数及其输入模式。这些可以传递给兼容的聊天模型，让模型决定是否调用工具以及使用哪些参数。

![](./images/1766296155654-9ea1616c-69b3-4910-b2b1-6eb361f9c943.png)

## 预建工具
LangChain 为常见的外部系统（包括 API、数据库、文件系统和 Web 数据）提供预构建的工具集成。

浏览[集成目录](https://python.langchain.com/docs/integrations/tools/)以查找可用的工具。

常见类别：

+ **搜索**：Bing、SerpAPI、Tavily
+ **代码执行**：Python REPL、Node.js REPL
+ **数据库**：SQL、MongoDB、Redis
+ **Web 数据**：抓取和浏览
+ **API**：OpenWeatherMap、NewsAPI等。

## 自定义工具
使用@tool装饰器来定义工具

```plain
from langchain_core.tools import tool


@tool
def multiply(a: int, b: int) -> int:
    """将两个数目相乘."""
    return a * b


# 运行工具
print(multiply.invoke({"a": 6, "b": 7}))  # returns 42

tool_call = {
    "type": "tool_call",
    "id": "1",
    "args": {"a": 42, "b": 7}
}
print(multiply.invoke(tool_call))

print("=" * 8, "在代理中使用", "=" * 8)
from langgraph.prebuilt import create_react_agent
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')

agent = create_react_agent(
    model=llm,
    tools=[multiply]
)
print(agent.invoke({"messages": [{"role": "user", "content": "100乘以37等于多少？"}]}))

print("=" * 8, "在工作流中使用", "=" * 8)

from langgraph.prebuilt import ToolNode
from langgraph.graph import StateGraph, MessagesState, START, END
from langchain_tavily import TavilySearch


@tool
def tavily_search_tool(query: str) -> str:
    """这是一个搜索工具"""
    tool_instance = TavilySearch()
    return tool_instance.run(query)

# 执行工具的节点
tool_node = ToolNode([tavily_search_tool])
# 绑定工具到模型
model_with_tools = llm.bind_tools([tavily_search_tool])


def should_continue(state: MessagesState):
    messages = state["messages"]
    last_message = messages[-1]
    if last_message.tool_calls:
        return "tools"
    return END


def call_model(state: MessagesState):
    messages = state["messages"]
    response = model_with_tools.invoke(messages)
    return {"messages": [response]}


builder = StateGraph(MessagesState)

# 定义节点和边
builder.add_node("call_model", call_model)  
builder.add_node("tools", tool_node)

builder.add_edge(START, "call_model")  # 模型因为绑定了工具，决定十分要使用工具，返回：tool message
builder.add_conditional_edges("call_model", should_continue, ["tools", END])
builder.add_edge("tools", "call_model")

graph = builder.compile()

print(graph.invoke({"messages": [{"role": "user", "content": "上海的天气?"}]}))
```

用户输入："上海的天气？"

 ↓

🧠 call_model 节点

+ 使用绑定了工具的 LLM（model_with_tools）调用模型
+ 模型判断是否需要工具（如搜索）
+ 如果有 tool_calls，走向 tools 节点
+ 否则直接结束

   ↓

🔧 tools 节点（ToolNode）

+ 检查模型返回是否有 tool_calls（如 tavily_search_tool）
+ 执行对应工具（如 TavilySearch.run(query)）
+ 工具执行结果构建成 AI Message 返回

   ↓

🧠 再次回到 call_model（循环处理新消息）

+ 把 tool_result 消息输入给模型，继续推理

 ↓

✅ END（当模型不再调用工具，流程结束）

## 工具定制
### 参数说明
```plain
from langchain_core.tools import tool

@tool("multiply_tool", parse_docstring=True)
def multiply(a: int, b: int) -> int:
    """Multiply two numbers.

    Args:
        a: First operand
        b: Second operand
    """
    return a * b
```

### 显式输入模式
```plain
from pydantic import BaseModel, Field
from langchain_core.tools import tool

class MultiplyInputSchema(BaseModel):
    """Multiply two numbers"""
    # 通过创建一个类继承BaseModel去作为参数验证和工具描述
    a: int = Field(description="First operand")
    b: int = Field(description="Second operand")

@tool("multiply_tool", args_schema=MultiplyInputSchema)
def multiply(a: int, b: int) -> int:
    return a * b
```

## 上下文管理
LangGraph 中的工具有时需要上下文数据，例如仅在运行时使用的参数（例如，用户 ID 或会话详细信息），这些数据不应由模型控制。LangGraph 提供了三种方法来管理此类上下文：

| **<font style="color:rgba(0, 0, 0, 0.87);">类型</font>** | **<font style="color:rgba(0, 0, 0, 0.87);">使用场景</font>** | **<font style="color:rgba(0, 0, 0, 0.87);">可变的</font>** | **<font style="color:rgba(0, 0, 0, 0.87);">寿命</font>** |
| :--- | :--- | :--- | :--- |
| <font style="color:rgb(30, 136, 229);">配置</font> | <font style="color:rgba(0, 0, 0, 0.87);">静态、不可变的运行时数据</font> | <font style="color:rgba(0, 0, 0, 0.87);">❌</font> | <font style="color:rgba(0, 0, 0, 0.87);">单次调用</font> |
| <font style="color:rgb(30, 136, 229);">短期记忆</font> | <font style="color:rgba(0, 0, 0, 0.87);">调用期间动态变化的数据</font> | <font style="color:rgba(0, 0, 0, 0.87);">✅</font> | <font style="color:rgba(0, 0, 0, 0.87);">单次调用</font> |
| <font style="color:rgb(30, 136, 229);">长期记忆</font> | <font style="color:rgba(0, 0, 0, 0.87);">持久的跨会话数据</font> | <font style="color:rgba(0, 0, 0, 0.87);">✅</font> | <font style="color:rgba(0, 0, 0, 0.87);">跨多个会话</font> |


### 配置
```plain
from langchain_core.runnables import RunnableConfig
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')
@tool
def get_user_info(
    config: RunnableConfig,
) -> str:
    """查找用户信息."""
    user_id = config["configurable"].get("user_id")
    return "用户的名字是初见" if user_id == "user_123" else "Unknown user"

# 创建Agent
agent = create_react_agent(
    model=llm,
    tools=[get_user_info],
)

response = agent.invoke(
    {"messages": [{"role": "user", "content": "查询用户信息"}]},
    config={"configurable": {"user_id": "user_123"}}
)
print(response)
```

### 短期记忆
短期记忆保持在单次执行期间发生变化的**动态状态**

```plain
from typing import Annotated, NotRequired
from langgraph.prebuilt import InjectedState, create_react_agent
from langgraph.prebuilt.chat_agent_executor import AgentState
from langchain_core.messages import ToolMessage
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-01-25')


class CustomState(AgentState):
    # user_name字段处于短期状态
    user_name: NotRequired[str]


# 只有通过 InjectedState，LangGraph 才会自动把当前运行状态（CustomState）注入到工具函数中。
@tool
def get_user_name(
        state: Annotated[CustomState, InjectedState]
) -> str:
    """从state中检索当前用户名。"""
    # 返回存储的名称，如果未设置则返回默认值
    return state.get("user_name", "Unknown user")


@tool
def update_user_name(
        new_name: str,
        # 注入当前工具调用ID（LLM会忽略此参数，用于工具消息追踪）
        tool_call_id: Annotated[str, InjectedToolCallId]
) -> Command:
    """更新短期记忆中的用户名"""
    return Command(update={
        "user_name": new_name,
        "messages": [
            ToolMessage(content=f"名字已更新为：{new_name}", tool_call_id=tool_call_id)
        ]
    })


# 创建内存持久化器
checkpointer = InMemorySaver()

# 创建代理
agent = create_react_agent(
    model=llm,
    tools=[get_user_name, update_user_name],
    state_schema=CustomState,
    checkpointer=checkpointer,
)
config = {"configurable": {"thread_id": "1"}}

# 第一次：更新用户名（模型会调用update_user_name工具）
response = agent.invoke({"messages": [{"role": "user", "content": "我的名字是初见"}]}, config)
response["messages"][-1].pretty_print()

# 第二次：获取用户名（调用get_user_name工具）
response = agent.invoke({"messages": [{"role": "user", "content": "我的名字是什么?"}]}, config)
response["messages"][-1].pretty_print()
```

### 长期记忆
使用[长期记忆](https://langchain-ai.github.io/langgraph/concepts/memory/#long-term-memory)来存储对话中特定于用户或应用程序的数据。这对于像聊天机器人这样的应用程序非常有用。

要使用长期记忆，需要：

1. 配置存储以在调用之间保留数据。
2. 使用该`get_store`功能从工具或提示中访问store。

```plain
from langchain_core.runnables import RunnableConfig
from typing_extensions import TypedDict
from langchain_core.tools import tool
from langgraph.config import get_store
from langgraph.prebuilt import create_react_agent
from langgraph.store.memory import InMemoryStore
from langgraph.checkpoint.memory import InMemorySaver
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(api_key=os.getenv("DASHSCOPE_API_KEY"),
                      base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                      model_provider="openai",
                      model='qwen-plus-2025-07-28')
# 创建内存存储对象
store = InMemoryStore()
# 创建内存持久化器
checkpointer = InMemorySaver()

# 存储初始化用户信息
store.put(
    ("users",),
    "user_123",
    {
        "name": "初见",
        "language": "中文",
    }
)


class UserInfo(TypedDict):
    name: str
    language: str


@tool
def update_user_info(user_info: UserInfo, config: RunnableConfig) -> str:
    """更新用户信息"""
    print("工具被调用，接收到的 user_info：", user_info)
    store = get_store()
    user_id = config["configurable"].get("user_id")
    store.put(("users",), user_id, user_info)
    return "用户信息更新成功"


@tool
def get_user_info(config: RunnableConfig) -> str:
    """查找用户信息."""
    store = get_store()
    user_id = config["configurable"].get("user_id")
    user_info = store.get(("users",), user_id)
    return str(user_info.value) if user_info else "Unknown user"


# 创建代理
agent = create_react_agent(
    model=llm,
    tools=[update_user_info, get_user_info],
    checkpointer=checkpointer,
    store=store
)

config = {"configurable": {"thread_id": "1", "user_id": "user_123"}}
# 运行代理
response = agent.invoke(
    {"messages": [{"role": "user", "content": "查询用户信息"}]},
    config=config
)
response["messages"][-1].pretty_print()

# 运行代理-更新用户信息
agent.invoke(
    {"messages": [{"role": "user", "content": "我的名字叫李铭，使用的语言是西班牙语"}]},
    config=config
)

# 运行代理
response = agent.invoke(
    {"messages": [{"role": "user", "content": "查询用户姓名和使用语言"}]},
    config=config
)
response["messages"][-1].pretty_print()
```

# 人机交互
要在代理或工作流中审核、编辑和批准工具调用，请使用中断来暂停图表并等待人工输入。中断使用 LangGraph 的持久层（该层会保存图表状态）无限期暂停图表执行，直到恢复为止。

## 暂停使用`interrupt`
动态中断（也称为动态断点）根据图表的当前状态触发。您可以通过在适当的位置调用`interrupt`函数来设置动态中断。图表将暂停，以便人工干预，然后根据人工输入恢复图表。这对于审批、编辑或收集其他上下文等任务非常有用。

```plain
from dotenv import load_dotenv
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.messages import AIMessage
import os

load_dotenv()
llm = init_chat_model(
        api_key=os.getenv("DASHSCOPE_API_KEY"),
        base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
        model_provider="openai",
        model='qwen-plus-2025-07-28',
        temperature=0.7
    )


"""
    interrupt_before AI 已经干完活了（写了代码、写了邮件），你只是想在它造成后果（部署代码、发送邮件）之前检查一下。
    关键用作审核、拦截
"""

def writer_node(state: MessagesState):
    """节点 A: 负责写内容"""
    print("✍️ [Writer]: 正在撰写推文...")
    msg = llm.invoke([{"role": "user", "content": "写一条关于学习Python的幽默推文，50字以内"}])
    return {"messages": [msg]}


def publisher_node(state: MessagesState):
    """节点 B: 负责发布 (副作用)"""
    content = state["messages"][-1].content
    print(f"\n[Publisher]: 推文已发布到互联网: \n{content}")
    return {"messages": [AIMessage(content="发布成功")]}


# --- 构建图 ---

builder = StateGraph(MessagesState)
builder.add_node("writer", writer_node)
builder.add_node("publisher", publisher_node)

builder.add_edge(START, "writer")
builder.add_edge("writer", "publisher")
builder.add_edge("publisher", END)

# 核心配置：在进入 publisher 之前必须暂停！
checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer, interrupt_before=["publisher"])


# --- 运行逻辑 ---

def run_demo_1():
    print("\n=== 案例 1: interrupt_before (审核模式) ===")
    config = {"configurable": {"thread_id": "tweet_1"}}

    # 1. 启动：它会跑完 writer，然后在 publisher 门口停下
    print(">>> 启动任务...")
    graph.invoke({"messages": []}, config)

    # 2. 检查状态
    snapshot = graph.get_state(config)
    ai_draft = snapshot.values["messages"][-1].content

    print(f"\n[人类审核员]: 看到 AI 写了: {ai_draft}")
    user_input = input("是否批准? (y=批准 / 输入文字=修改并发布): ").strip()

    # 3. 处理决策
    if user_input.lower() == 'y':
        print("批准！放行...")
        # 传入 None 表示继续执行原计划
        graph.invoke(None, config)
    else:
        print("修改中...")
        # 修改状态：用人类的话替换 AI 的话
        new_msg = AIMessage(content=user_input)
        graph.update_state(config, {"messages": [new_msg]}, as_node="writer")

        print("修改完成，放行...")
        graph.invoke(None, config)


"""
    interrupt 函数 AI 正在执行一个复杂的长任务，跑到第 3 步发现缺个 API Key，或者缺个参数，它需要停下来找你要，拿到后继续跑第 4 步。
    关键用作填空
"""
# --- 节点定义 ---
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt, Command

# 1. 定义 State：我们需要一个字段来在节点间传递 days
class TripState(TypedDict):
    messages: Annotated[list, add_messages]
    days: str  # 新增字段，用来存储天数

# --- 节点 1: 负责开场白 (副作用) ---
def start_node(state: TripState):
    # 这里放你只想执行一次的代码
    print("\n🤖 [AI]: 收到，正在为你规划去‘云南’的旅行...")
    # 这个节点不返回 messages，只负责打印和过渡
    return {}

# --- 节点 2: 负责提问 + 生成 ---
def planner_node(state: TripState):
    # 1. 中断逻辑
    # 第一次运行：在这里暂停
    # 恢复运行：直接从这里拿到值，继续往下走，不会回头去跑 start_node
    days = interrupt("请问你打算去几天？")

    print(f"✅ [AI]: 收到，{days}天。正在生成行程...")

    # 2. 生成逻辑
    # (这里为了演示简单，直接返回文本，实际可用 LLM)
    content = f"这是为你生成的云南 {days} 天行程：大理 -> 丽江 -> 香格里拉..."
    return {"messages": [AIMessage(content=content)], "days": days}

# --- 构建图 ---
builder_2 = StateGraph(TripState)

builder_2.add_node("start", start_node)
builder_2.add_node("planner", planner_node)

# 🔗 连线： Start -> Planner -> End
builder_2.add_edge(START, "start")
builder_2.add_edge("start", "planner")
builder_2.add_edge("planner", END)

checkpointer_2 = MemorySaver()
graph_2 = builder.compile(checkpointer=checkpointer_2)


# --- 运行逻辑 ---

def run_demo_2():
    print("\n=== 案例 2: interrupt 函数 (填空模式) ===")
    config = {"configurable": {"thread_id": "trip_1"}}

    print(">>> 启动任务...")
    # 1. 第一次运行
    graph_2.stream({"messages": []}, config, stream_mode="values")

    # 2. 捕获中断
    snapshot = graph_2.get_state(config)

    # 检查是否有中断任务
    if snapshot.tasks and snapshot.tasks[0].interrupts:
        # 获取 interrupt("...") 里的问题
        question = snapshot.tasks[0].interrupts[0].value
        print(f"\n[系统暂停] AI 询问: {question}")

        # 3. 获取人类回答
        answer = input("回答: ")

        print("恢复执行...")
        # 4. 恢复执行：使用 Command 将答案传回给节点内部的 days 变量
        res = graph_2.invoke(Command(resume=answer), config)
        print(res)


if __name__ == "__main__":
    run_demo_1()
    # run_demo_2()
```

**注意：**就开发者体验而言，中断类似于 Python 的 input() 函数，但它们不会自动从中断点恢复执行。相反，它们会重新运行发生中断的整个节点。因此，中断通常最好放置在节点的起始位置或专用节点中。

# 多智能体系统
代理是一种使用 LLM 来决定应用程序控制流的系统。随着这些系统的开发，它们可能会随着时间的推移变得更加复杂，从而更难以管理和扩展。例如，您可能会遇到以下问题：

+ 代理可以使用的工具太多，无法决定下一步调用哪个工具
+ 环境变得过于复杂，单个代理无法跟踪
+ 系统中需要多个专业领域（例如规划师、研究员、数学专家等）

为了解决这些问题，您可以考虑将应用程序拆分成多个较小的独立代理，并将它们组合成一个**多代理系统**。这些独立代理可以像提示符和 LLM 调用一样简单，也可以像ReAct代理一样复杂（甚至更多！）。

使用多代理系统的主要好处是：

+ **模块化**：独立的代理使得代理系统的开发、测试和维护变得更加容易。
+ **专业化**：您可以创建专注于特定领域的专家代理，这有助于提高整体系统性能。
+ **控制**：您可以明确控制代理如何通信。

![](./images/1766296155546-4cc30b47-7d52-43fa-bceb-1c2c780f96f7.png)

在多代理系统中，有几种连接代理的方法：

+ **网络**：每个代理都可以与其他代理通信。任何代理都可以决定接下来要呼叫哪个代理。
+ **主管代理**：每个代理只与一个主管代理进行通信。主管代理负责决定接下来应该调用哪个代理。
+ **主管（工具调用）**：这是主管架构的一个特例。单个代理可以表示为工具。在这种情况下，主管代理使用工具调用 LLM 来决定调用哪些代理工具，以及传递给这些代理的参数。
+ **分层结构**：你可以定义一个多智能体系统，其中包含多个主管的主管。这是主管架构的泛化，允许更复杂的控制流。
+ **自定义多代理工作流**：每个代理仅与一部分代理进行通信。流程的某些部分是确定性的，只有部分代理可以决定接下来要调用哪些其他代理。

## 交接（Handoffs）
### 交接概念
在多智能体架构中，智能体可以表示为图节点。每个智能体节点执行其步骤，并决定是完成执行还是路由至其他智能体，包括可能路由至自身（例如，循环运行）。多智能体交互中一种常见的模式是**切换**，即一个智能体将控制权移交给另一个智能体

### 基础交接示例
**在线订餐系统**

**场景描述**：顾客下单 → 订单处理 → 支付处理 → 配送安排

```plain
import json
import random
import re
from typing import Literal, TypedDict, Annotated
from langgraph.types import Command
from langgraph.graph import StateGraph, START, END, add_messages
from langchain_core.messages import HumanMessage, SystemMessage, BaseMessage
from langchain_openai import ChatOpenAI
import os
from dotenv import load_dotenv

load_dotenv()

# --- 1. 配置与模拟数据 ---

# 模拟数据库：菜单与价格
MENU_DB = {
    "汉堡": 25.0,
    "芝士汉堡": 28.0,
    "薯条": 12.0,
    "可乐": 8.0,
    "雪碧": 8.0,
    "炸鸡": 35.0
}

# 初始化大模型
llm = ChatOpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    model='qwen-plus-2025-07-28',
    temperature=0.5
)


# --- 2. 工具函数 ---

def parse_json_output(text: str) -> dict:
    """
    鲁棒的JSON解析器：自动清洗Markdown标记，处理常见的LLM格式问题
    """
    try:
        # 移除 ```json 和 ``` 标记
        cleaned_text = re.sub(r'```json\s*', '', text)
        cleaned_text = re.sub(r'```', '', cleaned_text)
        return json.loads(cleaned_text.strip())
    except Exception as e:
        print(f"JSON解析警告: {e}, 原始内容: {text[:50]}...")
        return {}


def call_llm(system_prompt: str, user_message: str, temperature: float = 0.7) -> str:
    """封装的LLM调用"""
    try:
        messages = [
            SystemMessage(content=system_prompt),
            HumanMessage(content=user_message)
        ]
        llm.temperature = temperature
        response = llm.invoke(messages)
        return response.content
    except Exception as e:
        return f"Error: {str(e)}"


# --- 3. 状态定义 ---

class OrderState(TypedDict):
    customer_name: str  # 客户姓名
    # 使用 add_messages 保留对话历史
    messages: Annotated[list[BaseMessage], add_messages]
    order_items: list  # 结构: [{"name": "汉堡", "price": 25.0}]
    raw_order_text: str  # 用户原始输入
    total_amount: float  # 总金额
    payment_status: str  # 支付结果
    delivery_address: str  # 配送地址
    order_status: str  # 订单状态
    llm_analysis: dict  # 大模型返回的分析结果


# --- 4. 智能体节点定义 ---

def order_receiver(state: OrderState) -> Command[Literal["payment_processor", "order_validator"]]:
    """
    订单接收节点：负责 NLU（自然语言理解）和 实体映射
    """
    raw_text = state.get("raw_order_text", "")
    customer_name = state.get("customer_name", "客户")

    print(f"\n[接收] 正在处理 {customer_name} 的需求: {raw_text}")

    # 如果已经有 items（可能是重试或手动输入的），跳过解析
    if state.get("order_items") and not raw_text:
        total = sum(item['price'] for item in state['order_items'])
        return Command(goto="payment_processor", update={"total_amount": total})

    # 1. 调用 LLM 解析意图
    system_prompt = f"""你是一个餐厅订单解析员。
当前菜单包括：{', '.join(MENU_DB.keys())}。
请分析用户输入，提取菜品。如果用户说的菜不在菜单里，请标记为未知。

返回严格的JSON格式：
{{
    "items_identified": ["菜名1", "菜名2"],
    "unknown_items": ["不在菜单的词"],
    "intent_clarity": "high/medium/low",
    "missing_info": "缺少的关键信息(如地址等)"
}}"""

    llm_response = call_llm(system_prompt, f"用户说：{raw_text}", temperature=0.1)
    analysis = parse_json_output(llm_response)

    print(f"AI分析: {analysis}")

    # 2. 核心逻辑：将识别出的菜名映射为真实订单数据
    identified_names = analysis.get("items_identified", [])
    new_order_items = []

    for name in identified_names:
        # 简单的模糊匹配逻辑（实际项目中可用向量搜索）
        if name in MENU_DB:
            new_order_items.append({"name": name, "price": MENU_DB[name]})
        else:
            # 尝试查找最接近的菜单项（简化版）
            for menu_item in MENU_DB:
                if menu_item in name or name in menu_item:
                    new_order_items.append({"name": menu_item, "price": MENU_DB[menu_item]})
                    break

    total = sum(item['price'] for item in new_order_items)

    # --- 修复后的路由逻辑 ---

    # 检查是否有未知商品
    has_unknown_items = len(analysis.get("unknown_items", [])) > 0

    # 检查是否完全没识别出商品
    no_valid_items = len(new_order_items) == 0

    # 检查清晰度
    is_unclear = analysis.get("intent_clarity") == "low"

    # 只要满足任一负面条件，就去验证节点
    if no_valid_items or is_unclear or has_unknown_items:

        # 构建提示消息
        if has_unknown_items:
            msg_content = f"系统检测到未知商品：{', '.join(analysis['unknown_items'])}，需要确认。"
        elif no_valid_items:
            msg_content = "没能识别出具体菜品。"
        else:
            msg_content = "订单信息不明确。"

        return Command(
            goto="order_validator",
            update={
                "order_status": "解析存疑",
                "llm_analysis": analysis,
                "messages": [HumanMessage(content=msg_content)]
            }
        )

    # 一切完美才去支付
    return Command(
        goto="payment_processor",
        update={
            "order_items": new_order_items,
            "total_amount": total,
            "order_status": "待支付",
            "llm_analysis": analysis,
            "messages": [HumanMessage(content=f"已生成订单：{[i['name'] for i in new_order_items]}，总价：{total}元")]
        }
    )


def order_validator(state: OrderState) -> Command[Literal["payment_processor", END]]:
    """
    订单验证节点：处理异常或信息缺失
    """
    print("[验证] 正在检查订单完整性...")
    analysis = state.get("llm_analysis", {})
    order_items = state.get("order_items", [])

    # 简单的验证逻辑：如果有未知商品或金额为0
    unknowns = analysis.get("unknown_items", [])

    if unknowns:
        error_msg = f"抱歉，我们暂时不提供：{', '.join(unknowns)}。请重新下单。"
        return Command(
            goto=END,
            update={
                "order_status": "验证失败",
                "messages": [HumanMessage(content=error_msg)]
            }
        )

    if not order_items:
        return Command(
            goto=END,
            update={
                "order_status": "空订单",
                "messages": [HumanMessage(content="未能识别任何有效菜品，流程结束。")]
            }
        )

    # 如果验证通过（比如虽然有小问题但可忽略）
    return Command(
        goto="payment_processor",
        update={"messages": [HumanMessage(content="经二次验证，订单有效。")]}
    )


def payment_processor(state: OrderState) -> Command[Literal["delivery_scheduler", END]]:
    """
    支付节点
    """
    amount = state.get("total_amount", 0)
    print(f"[支付] 正在处理金额: {amount}元")

    # 模拟支付
    if amount > 1000:  # 假设大额风控
        return Command(
            goto=END,
            update={
                "payment_status": "拒绝",
                "messages": [HumanMessage(content="金额过大，支付被系统拒绝。")]
            }
        )

    return Command(
        goto="delivery_scheduler",
        update={
            "payment_status": "成功",
            "messages": [HumanMessage(content="支付成功！")]
        }
    )


def delivery_scheduler(state: OrderState) -> Command[Literal[END]]:
    """
    配送节点
    """
    address = state.get("delivery_address", "未填写地址")
    items = [i['name'] for i in state.get('order_items', [])]

    # 使用 LLM 生成最终通知
    prompt = f"""为客户生成一条外卖配送通知。
    菜品：{', '.join(items)}
    地址：{address}
    风格：热情、期待。"""

    msg = call_llm(prompt, "生成通知", temperature=0.7)
    print(f"[配送] {msg}")

    return Command(
        goto=END,
        update={
            "order_status": "配送中",
            "messages": [HumanMessage(content=msg)]
        }
    )


# --- 5. 构建图 ---

builder = StateGraph(OrderState)
builder.add_node("order_receiver", order_receiver)
builder.add_node("order_validator", order_validator)
builder.add_node("payment_processor", payment_processor)
builder.add_node("delivery_scheduler", delivery_scheduler)

builder.add_edge(START, "order_receiver")

graph = builder.compile()

# --- 6. 测试运行 ---

print("--- 测试用例 1: 纯自然语言输入 (自动解析) ---")
test_input = {
    "customer_name": "李四",
    "raw_order_text": "你好，我想要两个汉堡和一杯可乐，送到科技园A栋。",
    "delivery_address": "科技园A栋",  # 实际场景中这个也应该由LLM提取
    "messages": [],
    "order_items": []  # 故意留空，测试自动解析
}

try:
    final_state = graph.invoke(test_input)
    print("\n流程结束")
    print(f"最终状态: {final_state['order_status']}")
    print(f"最终金额: {final_state['total_amount']}")
    print(f"订单内容: {final_state['order_items']}")
except Exception as e:
    print(f"运行出错: {e}")

print("\n--- 测试用例 2: 包含不在菜单的商品 ---")
test_input_2 = {
    "customer_name": "王五",
    "raw_order_text": "来一份披萨和一杯可乐。",  # 披萨不在菜单里
    "messages": [],
    "order_items": []
}

try:
    final_state_2 = graph.invoke(test_input_2)
    print(f"\n最终状态: {final_state_2['order_status']}")
    # 应该打印出验证失败的消息
    print(f"系统回复: {final_state_2['messages'][-1].content}")
except Exception as e:
    print(f"运行出错: {e}")
```

### 交接的关键要点
+ 任务超出当前智能体能力范围
+ 需要专业化处理
+ 错误处理和重试机制
+ 工作流程的自然转换点

### 工具进行交接
暂时无法在飞书文档外展示此内容

```plain
from typing import Annotated
from typing_extensions import Literal

from langchain_core.tools import tool
from langchain_core.tools.base import InjectedToolCallId
from langchain_core.messages import ToolMessage, convert_to_messages
from langgraph.prebuilt import InjectedState
from langgraph.types import Command
from langgraph.graph import MessagesState, StateGraph, START, END

from langchain_openai import ChatOpenAI
import os
from dotenv import load_dotenv

load_dotenv()

# 初始化大模型
llm = ChatOpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    model='qwen-plus-2025-01-25',
    temperature=0.7
)


def make_handoff_tool(*, agent_name: str):
    """
    创建一个工具交接函数，用于在代理之间进行转接

    Args:
        agent_name (str): 目标代理的名称

    Returns:
        tool: 返回一个可以执行代理转接的工具函数
    """
    # 根据目标代理名称动态生成工具名称
    tool_name = f"transfer_to_{agent_name}"

    @tool(tool_name)
    def handoff_to_agent(
            # 注入当前图状态（LLM会忽略此参数，但工具内部可以使用）
            state: Annotated[dict, InjectedState],
            # 注入当前工具调用ID（LLM会忽略此参数，用于工具消息追踪）
            tool_call_id: Annotated[str, InjectedToolCallId],
    ):
        """请求另一个代理的帮助进行任务交接"""

        # 创建工具响应消息，表示成功转接到目标代理
        tool_message = {
            "role": "tool",
            "content": f"成功转接到 {agent_name} 代理",
            "name": tool_name,
            "tool_call_id": tool_call_id,
        }

        # 返回Command对象，用于导航到父图中的另一个代理节点
        return Command(
            # 导航到目标代理节点
            goto=agent_name,
            # 在父图中执行导航
            graph=Command.PARENT,
            # 更新状态：将完整的消息历史传递给目标代理，并添加工具消息
            # 这确保了聊天历史的完整性和有效性
            update={"messages": state["messages"] + [tool_message]},
        )

    return handoff_to_agent


def make_agent(model, tools, system_prompt=None):
    """
    创建一个智能代理，能够使用工具并在需要时进行代理转接

    Args:
        model: 语言模型实例
        tools: 代理可用的工具列表
        system_prompt: 系统提示词，定义代理的角色和行为

    Returns:
        compiled_graph: 编译后的代理图
    """
    # 将工具绑定到模型上
    model_with_tools = model.bind_tools(tools)

    # 创建工具名称到工具对象的映射，便于快速查找
    tools_by_name = {tool.name: tool for tool in tools}

    def call_model(state: MessagesState) -> Command[Literal["call_tools", END]]:
        """
        调用语言模型生成响应

        Args:
            state: 当前消息状态

        Returns:
            Command: 如果需要调用工具则转到call_tools，否则结束
        """
        messages = state["messages"]

        # 如果有系统提示词，将其添加到消息开头
        if system_prompt:
            messages = [{"role": "system", "content": system_prompt}] + messages

        # 调用绑定了工具的模型
        response = model_with_tools.invoke(messages)

        # 检查模型是否决定使用工具
        if len(response.tool_calls) > 0:
            # 如果有工具调用，转到工具执行节点
            return Command(goto="call_tools", update={"messages": [response]})

        # 如果没有工具调用，直接返回响应消息
        return {"messages": [response]}

    def call_tools(state: MessagesState) -> Command[Literal["call_model"]]:
        """
        执行工具调用

        Args:
            state: 当前消息状态

        Returns:
            list[Command]: 工具执行结果的命令列表
        """
        # 获取最后一条消息中的所有工具调用
        tool_calls = state["messages"][-1].tool_calls
        results = []

        # 逐个执行工具调用
        for tool_call in tool_calls:
            # 根据工具名称获取对应的工具对象
            tool_ = tools_by_name[tool_call["name"]]

            # 获取工具的输入参数架构
            tool_input_fields = tool_.get_input_schema().model_json_schema()[
                "properties"
            ]

            # 检查工具是否需要状态注入（简化版实现）
            if "state" in tool_input_fields:
                # 如果工具需要状态，将当前状态注入到工具参数中
                tool_call = {**tool_call, "args": {**tool_call["args"], "state": state}}

            # 执行工具调用
            tool_response = tool_.invoke(tool_call)

            # 处理不同类型的工具响应
            if isinstance(tool_response, ToolMessage):
                # 标准工具消息响应
                results.append(Command(update={"messages": [tool_response]}))
            elif isinstance(tool_response, Command):
                # 直接返回Command对象的工具（如转接工具）
                results.append(tool_response)
            else:
                # 普通响应，转换为工具消息
                tool_message = ToolMessage(
                    content=str(tool_response),
                    tool_call_id=tool_call["id"]
                )
                results.append(Command(update={"messages": [tool_message]}))

        # 返回所有工具执行结果
        return results

    # 构建代理的内部图结构
    graph = StateGraph(MessagesState)

    # 添加模型调用节点和工具调用节点
    graph.add_node("call_model", call_model)
    graph.add_node("call_tools", call_tools)

    # 设置图的边：从开始到模型调用，从工具调用回到模型调用
    graph.add_edge(START, "call_model")
    graph.add_edge("call_tools", "call_model")

    # 编译并返回图
    return graph.compile()


def pretty_print_messages(update):
    """
    美化打印消息更新，用于调试和展示

    Args:
        update: 图更新信息，可能是元组或字典
    """
    # 检查是否是来自子图的更新
    if isinstance(update, tuple):
        ns, update = update

        # 跳过父图更新的打印
        if len(ns) == 0:
            return

        # 提取子图ID并打印
        graph_id = ns[-1].split(":")[0]
        print(f"来自子图 {graph_id} 的更新:")
        print()

    # 遍历所有节点更新
    for node_name, node_update in update.items():
        print(f"来自节点 {node_name} 的更新:")
        print()

        # 美化打印所有消息
        if "messages" in node_update:
            for m in convert_to_messages(node_update["messages"]):
                m.pretty_print()
        print()


# ============= 定义数学工具 =============

@tool
def add(a: int, b: int) -> int:
    """执行两个数字的加法运算"""
    result = a + b
    print(f"执行加法: {a} + {b} = {result}")
    return result


@tool
def multiply(a: int, b: int) -> int:
    """执行两个数字的乘法运算"""
    result = a * b
    print(f"执行乘法: {a} × {b} = {result}")
    return result


@tool
def subtract(a: int, b: int) -> int:
    """执行两个数字的减法运算"""
    result = a - b
    print(f"执行减法: {a} - {b} = {result}")
    return result


@tool
def divide(a: int, b: int) -> float:
    """执行两个数字的除法运算"""
    if b == 0:
        return "错误：不能除以零"
    result = a / b
    print(f"执行除法: {a} ÷ {b} = {result}")
    return result


# ============= 演示单个代理 =============

def demo_single_agent():
    """演示单个具有所有数学工具的代理"""
    print("=" * 60)
    print("演示：单个数学代理")
    print("=" * 60)

    # 创建一个拥有所有数学工具的代理
    math_agent = make_agent(
        llm,
        [add, multiply, subtract, divide],
        system_prompt="你是一个数学专家，可以执行各种数学运算。请一步步解决问题。"
    )

    print("问题: 计算 (3 + 5) × 12")
    print()

    # 运行代理并显示结果
    for chunk in math_agent.stream({"messages": [("user", "计算 (3 + 5) × 12")]}):
        pretty_print_messages(chunk)


# ============= 演示多代理协作 =============

def demo_multi_agent_collaboration():
    """演示多个专业代理之间的协作"""
    print("=" * 60)
    print("演示：多代理协作系统")
    print("=" * 60)

    # 创建加法专家代理
    addition_expert = make_agent(
        llm,
        [add, subtract, make_handoff_tool(agent_name="multiplication_expert")],
        system_prompt="""你是加法和减法专家。你精通加法和减法运算。
            当你完成加法或减法运算后，如果后续还需要乘法或除法运算，
            请立即使用 transfer_to_multiplication_expert 工具转接给乘法专家。
            不要尝试自己完成乘法运算。"""
    )

    # 创建乘法专家代理
    multiplication_expert = make_agent(
        llm,
        [multiply, divide, make_handoff_tool(agent_name="addition_expert")],
        system_prompt="""你是乘法和除法专家。你精通乘法和除法运算。
            当你接收到需要乘法运算的任务时，请立即执行乘法运算。
            如果后续还需要加法或减法运算，请转接给加法专家。
            当前任务：执行乘法运算并给出最终答案。"""
    )

    # 构建多代理协作图
    builder = StateGraph(MessagesState)

    # 添加两个专家代理节点
    builder.add_node("addition_expert", addition_expert)
    builder.add_node("multiplication_expert", multiplication_expert)

    # 设置入口点为加法专家
    builder.add_edge(START, "addition_expert")

    # 编译协作图
    collaboration_graph = builder.compile()

    print("问题: 计算 (3 + 5) × 12")
    print("加法专家将处理加法，然后转接给乘法专家处理乘法")
    print()

    # 运行协作图并显示子图中的所有更新
    for chunk in collaboration_graph.stream(
            {"messages": [("user", "请计算 (3 + 5) × 12")]},
            subgraphs=True  # 包含子图更新
    ):
        pretty_print_messages(chunk)


# ============= 更复杂的协作示例 =============

def demo_complex_collaboration():
    """演示更复杂的多步骤协作"""
    print("=" * 60)
    print("演示：复杂多步协作")
    print("=" * 60)

    # 创建基础运算专家
    basic_math_expert = make_agent(
        llm,
        [add, subtract, make_handoff_tool(agent_name="advanced_math_expert")],
        system_prompt="""你是基础数学专家，专门处理加法和减法。
        对于乘法、除法等高级运算，请转接给高级数学专家。
        你需要先处理括号内的基础运算。"""
    )

    # 创建高级运算专家
    advanced_math_expert = make_agent(
        llm,
        [multiply, divide, make_handoff_tool(agent_name="basic_math_expert")],
        system_prompt="""你是高级数学专家，专门处理乘法和除法。
        对于加法、减法等基础运算，请转接给基础数学专家。
        你负责处理复杂的乘除运算。"""
    )

    # 构建协作图
    builder = StateGraph(MessagesState)
    builder.add_node("basic_math_expert", basic_math_expert)
    builder.add_node("advanced_math_expert", advanced_math_expert)
    builder.add_edge(START, "basic_math_expert")

    complex_graph = builder.compile()

    print("复杂问题: 计算 ((10 + 5) × 3 - 8) ÷ 2")
    print("将需要多次代理转接来完成计算")
    print()

    for chunk in complex_graph.stream(
            {"messages": [("user", "请逐步计算 ((10 + 5) × 3 - 8) ÷ 2")]},
            subgraphs=True
    ):
        pretty_print_messages(chunk)


# ============= 主程序入口 =============

def main():
    """主程序，运行所有演示"""
    print("LangGraph工具交接案例演示")
    print("展示单代理和多代理协作的数学计算系统")
    print()

    try:
        # 演示1：单个代理
        # demo_single_agent()
        #
        # print("\n" + "-" * 20 + "\n")

        # 演示2：多代理协作
        demo_multi_agent_collaboration()

        # print("\n" + "-" * 20 + "\n")
        #
        # # 演示3：复杂协作
        # demo_complex_collaboration()

    except Exception as e:
        print(f"运行出错: {e}")


if __name__ == "__main__":
    main()
```

## 如何构建多智能体应用
### 自定义主管架构
```plain
from typing import Literal, Annotated, Dict, Any, List
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import InjectedState, create_react_agent
import json
import os
from dotenv import load_dotenv

load_dotenv()

# 模拟数据
KNOWLEDGE_BASE = {
    "login": "请清除浏览器缓存并重新登录，或重置密码。",
    "payment": "请检查银行卡余额，确认交易状态，或联系银行。",
    "bug": "我们已记录此问题，技术团队将在24小时内处理。",
    "network": "请检查网络连接，或尝试切换网络环境。",
    "performance": "建议清理缓存、重启应用或检查系统资源使用情况。"
}

PRODUCTS = {
    "basic": {"name": "基础版", "price": 99, "features": ["基础功能", "邮件支持"]},
    "pro": {"name": "专业版", "price": 299, "features": ["高级功能", "优先支持", "API访问"]},
    "enterprise": {"name": "企业版", "price": 999, "features": ["企业功能", "专属客服", "定制开发"]}
}

USER_DATABASE = {
    "user123": {"plan": "pro", "status": "active", "support_level": "premium", "balance": 500},
    "user456": {"plan": "basic", "status": "active", "support_level": "standard", "balance": 100}
}

TICKET_SYSTEM = []  # 模拟工单系统

llm = ChatOpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    model='qwen-plus-2025-07-28',  # 建议使用能力较强的模型作为监督者
    temperature=0.1
)


# 扩展状态定义
class CustomerServiceState(MessagesState):
    current_agent: str  # 目前智能体
    customer_id: str  # 用户id
    issue_type: str  # 问题类别
    priority: str  # 优先级
    resolution_status: str  # 解决状态
    routing_reason: str  # 路由原因
    ticket_id: str  # 工单id
    remaining_steps: int  #


# ================================
# 为ReAct Agent定义工具函数
# ================================

@tool
def search_knowledge_base(query: str) -> str:
    """搜索技术知识库，查找解决方案"""
    print(f"进入搜索技术知识库工具")
    query_lower = query.lower()
    print(query_lower)
    results = []
    for issue, solution in KNOWLEDGE_BASE.items():
        if issue in query_lower:
            results.append(f"{issue}: {solution}")

    if results:
        return f"找到以下解决方案:\n" + "\n".join(results)
    else:
        return "未在知识库中找到相关解决方案，建议创建技术工单进行人工处理。"


@tool
def get_product_info(product_query: str) -> str:
    """获取产品信息和价格"""
    print(f"进入查询{product_query}产品信息和价格工具")
    if not product_query:
        # 返回所有产品信息
        result = "我们的产品线包括:\n\n"
        for key, product in PRODUCTS.items():
            result += f"**{product['name']}** - ¥{product['price']}/月\n"
            result += f"功能: {', '.join(product['features'])}\n\n"
        return result

    # 搜索特定产品
    query_lower = product_query.lower()
    for key, product in PRODUCTS.items():
        if key in query_lower or product['name'] in query_lower:
            return f"**{product['name']}**\n价格: ¥{product['price']}/月\n功能: {', '.join(product['features'])}"

    return f"未找到关于'{product_query}'的产品信息。请查看我们的完整产品列表。"


@tool
def get_user_account_info(user_id: str) -> str:
    """查询用户账户信息"""
    print(f"进入查询{user_id}的信息工具")
    if not user_id:
        return "请提供您的用户ID以查询账户信息。"

    if user_id in USER_DATABASE:
        user_info = USER_DATABASE[user_id]
        return f"""账户信息:
用户ID: {user_id}
当前套餐: {user_info['plan']}
账户状态: {user_info['status']}
支持级别: {user_info['support_level']}
账户余额: ¥{user_info['balance']}"""
    else:
        return f"未找到用户ID '{user_id}' 的账户信息。请检查用户ID是否正确。"


@tool
def create_support_ticket(issue_description: str, priority: str,
                          state: Annotated[Dict, InjectedState]) -> str:
    """创建技术支持工单"""
    import uuid
    import datetime
    print(f"进入创建技术支持工单工具")

    ticket_id = f"TICKET-{str(uuid.uuid4())[:8].upper()}"
    customer_id = state.get("customer_id", "anonymous")

    ticket = {
        "id": ticket_id,
        "customer_id": customer_id,
        "issue": issue_description,
        "priority": priority,
        "status": "open",
        "created_at": datetime.datetime.now()  # 模拟时间
    }

    TICKET_SYSTEM.append(ticket)

    # 更新状态
    state["ticket_id"] = ticket_id

    return f"已创建支持工单: {ticket_id}\n问题描述: {issue_description}\n优先级: {priority}\n我们的技术团队将在24小时内处理您的问题。"


@tool
def calculate_upgrade_cost(current_plan: str, target_plan: str) -> str:
    """计算升级费用"""
    print(f"进入计算升级费用的工具")
    if current_plan not in PRODUCTS or target_plan not in PRODUCTS:
        return "无效的套餐类型。请检查套餐名称。"

    current_price = PRODUCTS[current_plan]["price"]
    target_price = PRODUCTS[target_plan]["price"]

    if target_price <= current_price:
        return f"目标套餐 ({PRODUCTS[target_plan]['name']}) 价格不高于当前套餐 ({PRODUCTS[current_plan]['name']})，无需升级费用。"

    upgrade_cost = target_price - current_price
    return f"""升级费用计算:
当前套餐: {PRODUCTS[current_plan]['name']} (¥{current_price}/月)
目标套餐: {PRODUCTS[target_plan]['name']} (¥{target_price}/月)
升级费用: ¥{upgrade_cost}/月

新增功能: {', '.join(set(PRODUCTS[target_plan]['features']) - set(PRODUCTS[current_plan]['features']))}"""


@tool
def process_refund_request(user_id: str, reason: str) -> str:
    """处理退款请求"""
    print(f"进入处理退款请求工具")
    if not user_id or user_id not in USER_DATABASE:
        return "请提供有效的用户ID以处理退款请求。"

    user_info = USER_DATABASE[user_id]
    if user_info["status"] != "active":
        return "只有活跃账户才能申请退款。"

    # 模拟退款处理
    refund_amount = PRODUCTS[user_info["plan"]]["price"]

    return f"""退款申请已提交:
用户ID: {user_id}
退款原因: {reason}
退款金额: ¥{refund_amount}
处理时间: 3-5个工作日
退款将原路返回到您的支付账户。"""


# ================================
# 使用create_react_agent创建专业Agent
# ================================

def create_llm_router_with_react_agents():
    """创建使用ReAct Agent的智能路由系统"""

    # 1. 创建技术支持ReAct Agent
    tech_tools = [search_knowledge_base, create_support_ticket]
    tech_system_prompt = """你是技术支持工程师。

【职责】解决技术故障、报错、创建工单

【工作流程】
1. 使用 search_knowledge_base 搜索解决方案
2. 如果知识库无法解决,使用 create_support_ticket 创建工单
3. 给出简洁明确的答案后立即停止

【严格禁止】
- 不要回答产品价格、账户余额等非技术问题
- 不要说"我无法回答XXX",直接忽略
- 不要追问"还需要什么帮助"

【示例】
用户: "系统报错500,另外我想查余额"
正确: [使用search_knowledge_base] → "关于报错500: 建议清理缓存..."
错误: "关于报错500:... 至于余额查询,我无法帮助" """

    tech_agent = create_react_agent(
        llm,
        tech_tools,
        state_schema=CustomerServiceState,
        prompt=tech_system_prompt
    )

    # 2. 创建销售ReAct Agent - 关键修复
    sales_tools = [get_product_info, calculate_upgrade_cost]
    sales_system_prompt = """你是销售顾问。你的眼里只有产品和价格,其他问题视而不见。

【职责】回答产品、价格、套餐对比、升级费用

【工作流程】
1. 使用 get_product_info 获取产品信息(可以不传参数获取所有产品)
2. 使用 calculate_upgrade_cost 计算升级费用
3. 给出简洁答案后立即停止

【严格禁止 - 核心规则】
- 用户问了余额/账户/退款 → **当作没看见,一个字都不提**
- 用户问了技术故障 → **当作没看见,一个字都不提**
- 不要说"我无法回答XXX"
- 不要说"请联系客服/其他部门"
- 不要给任何非产品相关的建议

【正确示例】
用户: "专业版多少钱?我的余额是多少?ID是user123"
正确回复: 
[使用get_product_info('pro')] 
"专业版299元/月,包含高级功能、优先支持、API访问"
(完全不提余额)

错误回复:
"专业版299元/月... 至于余额查询,我无法提供,请联系客服"

【再次强调】
你只回答产品相关的部分,其他问题就像不存在一样!"""

    sales_agent = create_react_agent(
        llm,
        sales_tools,
        state_schema=CustomerServiceState,
        prompt=sales_system_prompt
    )

    # 3. 创建客户管理ReAct Agent - 关键修复
    admin_tools = [get_user_account_info, process_refund_request]
    admin_system_prompt = """你是客户管理专员。你的眼里只有账户和余额,其他问题视而不见。

【职责】查询账户信息、余额、处理退款

【工作流程】
1. 如果用户提供了ID,立即使用 get_user_account_info(user_id) 查询
2. 如果用户没提供ID,询问"请提供您的用户ID"后立即停止
3. 处理退款使用 process_refund_request

【严格禁止】
- 用户问了产品/价格 → **当作没看见,一个字都不提**
- 用户问了技术故障 → **当作没看见,一个字都不提**
- 不要说"我无法回答XXX"
- 查询完账户后不要追问"需要充值吗"

【正确示例】
用户: "专业版多少钱?查询余额,ID是user123"
正确回复:
[使用get_user_account_info('user123')]
"您的账户余额为500元"
(完全不提产品价格)

错误回复:
"关于专业版价格,请咨询销售... 您的余额为500元"

【核心原则】
只处理账户相关的部分,其他问题当作不存在!
如果用户提供了ID,必须立即调用工具查询!"""

    admin_agent = create_react_agent(
        llm,
        admin_tools,
        state_schema=CustomerServiceState,
        prompt=admin_system_prompt
    )

    # 4. 创建智能路由器 - 优化判断逻辑
    def llm_router(state: CustomerServiceState) -> Command[Literal["tech_agent", "sales_agent", "admin_agent", END]]:
        """LLM驱动的智能路由器"""
        messages = state["messages"]
        if not messages:
            return Command(goto=END)

        # 提取用户原始问题
        user_request = "未知"
        for msg in reversed(messages):
            if isinstance(msg, HumanMessage):
                user_request = msg.content
                break

        last_agent = state.get("current_agent", "start")

        # 构建对话历史摘要
        conversation_summary = []
        for i, msg in enumerate(messages[-8:]):
            if isinstance(msg, HumanMessage):
                conversation_summary.append(f"[{i}] 用户: {msg.content[:150]}")
            elif isinstance(msg, AIMessage):
                agent_name = getattr(msg, 'name', last_agent)
                content_preview = msg.content[:150] if msg.content else "[使用了工具]"
                conversation_summary.append(f"[{i}] {agent_name}: {content_preview}")

        conversation_context = "\n".join(conversation_summary)

        # 构建路由Prompt - 强化判断标准
        routing_prompt = f"""你是智能客服总控中心。

【用户原始请求】
"{user_request}"

【对话历史】
{conversation_context}

【当前状态】
上一步处理者: {last_agent}

【分析任务】
严格按以下步骤判断:

1. **任务拆解**
   将用户请求拆分成独立子任务
   例如: "查价格+查余额" = 2个任务

2. **完成度检查 - 严格标准**
   从对话历史找证据,判断每个任务是否**真正完成**:

   已完成的标准:
   - 产品查询: 给出了具体价格和功能
   - 余额查询: 给出了具体数字(如"余额500元")
   - 技术问题: 给出了解决方案或创建了工单

   未完成的标准:
   - 说"我无法回答XXX"
   - 说"请联系客服"
   - 说"建议等待进一步确认"
   - 完全没有提及该任务

   **关键**: 如果某个agent说"我无法处理XXX",这个任务算**未完成**!

3. **决策**
   - 有未完成的任务 → 路由到对应部门
   - 所有任务都有**具体答案** → 返回 "FINISH"

【部门职责】
- tech_agent: 技术故障、报错
- sales_agent: 产品、价格、升级费用
- admin_agent: 账户、余额、退款

【输出格式】
{{
    "task_breakdown": ["任务1", "任务2"],
    "completed_tasks": ["任务1"],
    "pending_tasks": ["任务2"],
    "analysis": "详细分析每个任务的完成情况",
    "route_to": "admin_agent 或 FINISH",
    "reason": "路由原因"
}}

【示例】
用户: "专业版多少钱?查余额,ID是user123"
Sales回复: "专业版299元... 至于余额,我无法提供"

正确判断:
{{
    "task_breakdown": ["查询专业版价格", "查询余额"],
    "completed_tasks": ["查询专业版价格"],
    "pending_tasks": ["查询余额"],
    "analysis": "价格已回答(299元),但余额查询Sales说无法提供,算未完成",
    "route_to": "admin_agent",
    "reason": "余额查询未完成,需admin处理"
}}"""

        try:
            response = llm.invoke([SystemMessage(content=routing_prompt)])

            # 提取JSON
            content = response.content.strip()
            if content.startswith("```"):
                content = content.split("```")[1]
                if content.startswith("json"):
                    content = content[4:]

            decision = json.loads(content.strip())

            # 提取字段
            task_breakdown = decision.get("task_breakdown", [])
            completed_tasks = decision.get("completed_tasks", [])
            pending_tasks = decision.get("pending_tasks", [])
            analysis = decision.get("analysis", "无分析")
            raw_route = decision.get("route_to", "FINISH").strip()
            reason = decision.get("reason", "智能调度")

            # 详细日志
            print(f"\n{'=' * 70}")
            print(f"[Router 分析报告]")
            print(f"用户原始请求: {user_request[:100]}")
            print(f"上一步处理者: {last_agent}")
            print(f"")
            print(f"任务拆解 (共{len(task_breakdown)}个):")
            for task in task_breakdown:
                status = "✅" if task in completed_tasks else "⏳"
                print(f"  {status} {task}")
            print(f"")
            print(f"分析: {analysis}")
            print(f"决策: {raw_route}")
            print(f"原因: {reason}")
            print(f"{'=' * 70}\n")

            # 1. 结束条件
            if raw_route.upper() in ["END", "FINISH", "STOP"]:
                print("✅ LLM判断: 所有任务已完成,结束对话\n")
                return Command(goto=END)

            # 2. 有效节点检查
            valid_nodes = ["tech_agent", "sales_agent", "admin_agent"]
            if raw_route not in valid_nodes:
                print(f"⚠️  无效节点 '{raw_route}',自动结束\n")
                return Command(goto=END)

            # 3. 防死循环检查
            if raw_route == last_agent and not pending_tasks:
                print(f"⚠️  检测到重复路由 ({raw_route}) 且无待处理任务,强制结束\n")
                return Command(goto=END)

            # 4. 正常跳转
            print(f"→ 路由到: {raw_route}\n")
            return Command(
                goto=raw_route,
                update={
                    "current_agent": raw_route,
                    "issue_type": raw_route.replace("_agent", ""),
                    "routing_reason": reason
                }
            )

        except json.JSONDecodeError as e:
            print(f"JSON解析失败: {e}")
            print(f"LLM原始回复:\n{response.content}\n")
            return Command(goto=END)
        except Exception as e:
            print(f"路由异常: {e}\n")
            import traceback
            traceback.print_exc()
            return Command(goto=END)

    # 5. 构建图结构
    builder = StateGraph(CustomerServiceState)

    builder.add_node("llm_router", llm_router)
    builder.add_node("tech_agent", tech_agent)
    builder.add_node("sales_agent", sales_agent)
    builder.add_node("admin_agent", admin_agent)

    builder.add_edge(START, "llm_router")
    builder.add_edge("tech_agent", "llm_router")
    builder.add_edge("sales_agent", "llm_router")
    builder.add_edge("admin_agent", "llm_router")

    return builder.compile()


# ================================
# 纯ReAct Agent架构 - 监督者也是ReAct Agent-工具进行交接
# ================================

def create_pure_react_system():
    """
    纯 ReAct 架构 - Supervisor 模式 (支持多任务自动切换)
    流程: Supervisor -> Sub-Agent -> Supervisor -> (Next Agent or End)
    """

    llm = ChatOpenAI(
        api_key=os.getenv("DASHSCOPE_API_KEY"),
        base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
        model='qwen-plus-2025-01-25',
        temperature=0.1
    )

    # --- 1. 定义监督者的路由工具 ---

    @tool
    def route_to_technical_support(issue_description: str, state: Annotated[dict, InjectedState],
                                   tool_call_id: Annotated[str, InjectedToolCallId]) -> Command:
        """将技术问题路由到技术支持部门

        Args:
            issue_description: 技术问题的描述
        """
        tool_message = {
            "role": "tool",
            "content": f"✅ 已转接至技术支持部门处理: {issue_description}",
            "name": "route_to_technical_support",
            "tool_call_id": tool_call_id,
        }
        print(f"\n🔀 [Supervisor] 路由到 tech_agent")
        print(f"   原因: {issue_description}\n")
        return Command(
            goto="tech_agent",
            graph=Command.PARENT,
            update={
                "current_agent": "tech_agent",
                "issue_type": "technical",
                "messages": state["messages"] + [tool_message]
            }
        )

    @tool
    def route_to_sales_department(inquiry_description: str, state: Annotated[dict, InjectedState],
                                  tool_call_id: Annotated[str, InjectedToolCallId]) -> Command:
        """将产品咨询路由到销售部门

        Args:
            inquiry_description: 产品咨询的描述
        """
        tool_message = {
            "role": "tool",
            "content": f"✅ 已转接至销售部门处理: {inquiry_description}",
            "name": "route_to_sales_department",
            "tool_call_id": tool_call_id,
        }
        print(f"\n🔀 [Supervisor] 路由到 sales_agent")
        print(f"   原因: {inquiry_description}\n")
        return Command(
            goto="sales_agent",
            graph=Command.PARENT,
            update={
                "current_agent": "sales_agent",
                "issue_type": "sales",
                "messages": state["messages"] + [tool_message]
            }
        )

    @tool
    def route_to_customer_management(request_description: str, state: Annotated[dict, InjectedState],
                                     tool_call_id: Annotated[str, InjectedToolCallId]) -> Command:
        """将账户/退款请求路由到客户管理部门

        Args:
            request_description: 账户管理请求的描述
        """
        tool_message = {
            "role": "tool",
            "content": f"✅ 已转接至客户管理部门处理: {request_description}",
            "name": "route_to_customer_management",
            "tool_call_id": tool_call_id,
        }
        print(f"\n🔀 [Supervisor] 路由到 admin_agent")
        print(f"   原因: {request_description}\n")
        return Command(
            goto="admin_agent",
            graph=Command.PARENT,
            update={
                "current_agent": "admin_agent",
                "issue_type": "administration",
                "messages": state["messages"] + [tool_message]
            }
        )

    # --- 2. 定义 Supervisor Prompt (关键优化) ---

    supervisor_prompt = """你是智能客服总管 (Supervisor)。你的职责是协调多部门协作,确保用户的所有问题都被妥善处理。

【核心职责】
1. 分析用户请求,识别所有子任务
2. 将任务分配给对应部门
3. 跟踪任务完成情况
4. 确保所有任务都被处理后才结束对话

【工作流程】

**阶段1: 接收用户请求**
- 仔细分析用户的完整请求
- 识别所有独立的子任务(可能有多个)
- 例如: "查价格+查余额" = 2个任务

**阶段2: 任务分配**
使用路由工具将任务分配给对应部门:
- 技术故障/报错 → route_to_technical_support
- 产品咨询/价格/升级 → route_to_sales_department  
- 账户查询/余额/退款 → route_to_customer_management

**阶段3: 任务跟踪 (关键!)**
当子部门返回结果后:
1. 检查对话历史 (messages)
2. 核对用户的原始请求
3. 判断是否还有未处理的任务:
   - 已完成: 部门给出了具体答案(价格、余额、解决方案等)
   - 未完成: 
     * 完全没有提及该任务
     * 部门说"我无法处理XXX"
     * 部门说"请联系其他部门"
4. 如果有未完成任务 → 立即路由到下一个部门
5. 如果所有任务都完成 → 给用户最终总结

【路由决策规则】
- 一次只路由一个任务到一个部门
- 优先处理技术问题,再处理销售,最后处理账户
- 不要试图自己回答具体业务问题
- 不要在有未完成任务时就结束对话

【判断标准 - 何时结束对话】
必须同时满足:
1. 用户的所有子任务都已被某个部门回答
2. 每个任务都有具体的答案(不是"无法处理")
3. 没有新的用户问题

【示例场景】

**场景1: 多任务处理**
用户: "专业版多少钱?我的余额是多少?ID是user123"

正确流程:
1. 识别任务: [查价格, 查余额]
2. 调用 route_to_sales_department("查询专业版价格")
3. Sales返回: "专业版299元/月"
4. 检查: 价完成, 余额❌未完成
5. 调用 route_to_customer_management("查询user123的余额")
6. Admin返回: "余额500元"
7. 检查: 所有任务完成
8. 给用户总结: "专业版299元/月。您的账户余额为500元。"

**场景2: 单任务处理**
用户: "系统报错500"

正确流程:
1. 识别任务: [技术故障]
2. 调用 route_to_technical_support("处理报错500")
3. Tech返回: "已找到解决方案..."
4. 检查: 所有任务完成
5. 给用户总结: "关于报错500,技术部门已提供解决方案..."

【错误示例】
错误1: 过早结束
用户问了2个问题,只处理了1个就结束

错误2: 自己回答
不使用路由工具,直接回复"专业版299元"

错误3: 重复路由
Sales已回答价格,又路由回Sales

【关键提醒】
- 你是协调者,不是执行者
- 必须使用路由工具分配任务
- 在所有任务完成前,不要结束对话
- 子部门说"无法处理"的任务,算未完成"""

    # --- 3. 定义子 Agent Prompt (强化边界) ---

    tech_prompt = """你是技术支持工程师。你的眼里只有技术问题。

【职责】
- 使用 search_knowledge_base 搜索解决方案
- 使用 create_support_ticket 创建工单

【工作原则】
1. 只处理技术故障、报错、系统问题
2. 使用工具后给出简洁答案
3. 回答完立即停止,不要追问
4. 用户问了产品/账户问题 → 完全忽略,当作没看见

【示例】
用户: "报错500,另外查下余额"
正确: [使用工具] → "关于报错500: 建议清理缓存..."
错误: "报错500已解决,余额请联系客服" """

    sales_prompt = """你是销售顾问。你的眼里只有产品和价格。

【职责】
- 使用 get_product_info 获取产品信息
- 使用 calculate_upgrade_cost 计算升级费用

【工作原则】
1. 只处理产品、价格、套餐对比
2. 使用工具后给出简洁答案
3. 回答完立即停止,不要追问
4. 用户问了技术/账户问题 → 完全忽略,当作没看见
5. 不要说"我无法回答XXX"或"请联系其他部门"

【示例】
用户: "专业版多少钱?查下余额"
正确: [使用get_product_info] → "专业版299元/月,包含..."
错误: "专业版299元,余额请联系客服" """

    admin_prompt = """你是客户管理专员。你的眼里只有账户和余额。

【职责】
- 使用 get_user_account_info 查询账户
- 使用 process_refund_request 处理退款

【工作原则】
1. 只处理账户、余额、退款
2. 如果用户提供了ID,立即使用工具查询
3. 如果没有ID,询问"请提供用户ID"后立即停止
4. 用户问了产品/技术问题 → 完全忽略,当作没看见
5. 不要说"我无法回答XXX"或"请联系其他部门"

【示例】
用户: "查余额,ID是user123,另外专业版多少钱"
正确: [使用get_user_account_info] → "您的余额为500元"
错误: "余额500元,价格请联系销售" """

    # --- 4. 创建 Agents ---

    supervisor_tools = [route_to_technical_support, route_to_sales_department, route_to_customer_management]
    supervisor_agent = create_react_agent(
        llm,
        supervisor_tools,
        state_schema=CustomerServiceState,
        prompt=supervisor_prompt
    )

    tech_agent = create_react_agent(
        llm,
        [search_knowledge_base, create_support_ticket],
        state_schema=CustomerServiceState,
        prompt=tech_prompt
    )

    sales_agent = create_react_agent(
        llm,
        [get_product_info, calculate_upgrade_cost],
        state_schema=CustomerServiceState,
        prompt=sales_prompt
    )

    admin_agent = create_react_agent(
        llm,
        [get_user_account_info, process_refund_request],
        state_schema=CustomerServiceState,
        prompt=admin_prompt
    )

    # --- 5. 构建图结构 ---

    builder = StateGraph(CustomerServiceState)

    # 添加节点
    builder.add_node("supervisor_agent", supervisor_agent)
    builder.add_node("tech_agent", tech_agent)
    builder.add_node("sales_agent", sales_agent)
    builder.add_node("admin_agent", admin_agent)

    # 设置入口 -> Supervisor
    builder.add_edge(START, "supervisor_agent")

    # 🔄 关键: 所有子 Agent 完成后返回 Supervisor
    builder.add_edge("tech_agent", "supervisor_agent")
    builder.add_edge("sales_agent", "supervisor_agent")
    builder.add_edge("admin_agent", "supervisor_agent")


    return builder.compile()


# ================================
# 测试运行函数
# ================================

def run_react_agent_examples():
    """运行ReAct Agent示例"""

    print("🤖 基于create_react_agent的多智能体客户服务系统")
    print("=" * 70)

    # 测试用例
    test_cases = [
        {
            "message": "我的应用login有问题",
            "customer_id": "user123"
        },
        # {
        #     "message": "我想了解专业版和企业版的区别，我们公司大概50人,另外我想查询我的余额，用户ID是user123",
        #     "customer_id": ""
        # },
        # {
        #     "message": "我想查看我的账户余额，用户ID是user123，另外我想申请退款",
        #     "customer_id": "user123"
        # },
        # {
        #     "message": "系统报错500，数据库连接失败，这是生产环境的紧急问题",
        #     "customer_id": "user456"
        # }
    ]

    systems = [
        ("混合架构 (LLM路由 + ReAct Agent)", create_llm_router_with_react_agents),
        # ("纯ReAct架构 (监督者也是ReAct Agent)", create_pure_react_system)
    ]

    for system_name, create_system in systems:
        print(f"\n🔥 {system_name}")
        print("=" * 70)

        try:
            graph = create_system()

            for i, test_case in enumerate(test_cases, 1):
                print(f"\n--- 测试案例 {i} ---")
                print(f"用户: {test_case['message']}")
                print("-" * 50)

                initial_state = {
                    "messages": [HumanMessage(content=test_case['message'])],
                    "current_agent": "start",
                    "customer_id": test_case['customer_id'],
                    "issue_type": "",
                    "priority": "normal",
                    "resolution_status": "pending",
                    "routing_reason": "",
                    "ticket_id": "",
                    "remaining_steps": 10
                }

                try:
                    result = graph.invoke(initial_state)

                    # 显示AI回复
                    print("🤖 AI回复:")
                    for msg in result["messages"]:
                        if isinstance(msg, AIMessage):
                            # 截取长回复以保持可读性
                            content = msg.content
                            if len(content) > 300:
                                content = content[:300] + "..."
                            print(f"  {content}")


                except Exception as e:
                    print(f"❌ 处理失败: {e}")

        except Exception as e:
            print(f"❌ 系统创建失败: {e}")


if __name__ == "__main__":
    # 检查API配置
    if not os.getenv("DASHSCOPE_API_KEY"):
        print("⚠️  请设置DASHSCOPE_API_KEY环境变量")
        print("export DASHSCOPE_API_KEY='your-api-key-here'")
    else:
        run_react_agent_examples()

        # 显示工单系统状态
        if TICKET_SYSTEM:
            print(f"\n📋 创建的工单 ({len(TICKET_SYSTEM)} 个):")
            for ticket in TICKET_SYSTEM:
                print(f"  {ticket['id']}: {ticket['issue'][:50]}...")
```

### 预构建主管架构
```plain
pip install langgraph-supervisor==0.0.29
```

```plain
from typing import List
from enum import Enum
from dataclasses import dataclass
from datetime import datetime
import uuid

from langchain_core.messages import HumanMessage, ToolMessage
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
from langgraph_supervisor import create_supervisor
from langchain.chat_models import init_chat_model
import os
from dotenv import load_dotenv

load_dotenv()

llm = init_chat_model(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    model_provider="openai",
    model='qwen-plus-2025-07-28'
)

# ============================================================================
# 模拟数据库
# ============================================================================

MOCK_ORDERS = {
    "ORD001": {
        "order_id": "ORD001",
        "customer": "张三",
        "product": "iPhone 15 Pro",
        "quantity": 1,
        "price": 8999.0,
        "status": "已发货",
        "tracking": "SF1234567890",
        "order_date": "2024-12-01"
    },
    "ORD002": {
        "order_id": "ORD002",
        "customer": "李四",
        "product": "MacBook Pro",
        "quantity": 1,
        "price": 15999.0,
        "status": "处理中",
        "tracking": None,
        "order_date": "2024-12-10"
    },
    "ORD003": {
        "order_id": "ORD003",
        "customer": "王五",
        "product": "AirPods Pro",
        "quantity": 2,
        "price": 3998.0,
        "status": "已完成",
        "tracking": "YTO9876543210",
        "order_date": "2024-11-25"
    }
}

MOCK_PRODUCTS = {
    "手机": [
        {"name": "iPhone 15 Pro", "price": 8999, "stock": 50, "rating": 4.8},
        {"name": "小米14 Ultra", "price": 5999, "stock": 100, "rating": 4.7},
        {"name": "华为Mate 60", "price": 6999, "stock": 30, "rating": 4.9}
    ],
    "电脑": [
        {"name": "MacBook Pro", "price": 15999, "stock": 20, "rating": 4.9},
        {"name": "ThinkPad X1", "price": 12999, "stock": 35, "rating": 4.6},
        {"name": "Dell XPS 15", "price": 13999, "stock": 25, "rating": 4.7}
    ],
    "耳机": [
        {"name": "AirPods Pro", "price": 1999, "stock": 200, "rating": 4.8},
        {"name": "索尼WH-1000XM5", "price": 2499, "stock": 80, "rating": 4.9},
        {"name": "Bose QC45", "price": 2299, "stock": 60, "rating": 4.7}
    ]
}


# ============================================================================
# 任务管理
# ============================================================================

class TaskStatus(Enum):
    PENDING = "待处理"
    COMPLETED = "已完成"
    FAILED = "失败"


@dataclass
class ServiceTask:
    task_id: str
    description: str
    status: TaskStatus = TaskStatus.PENDING
    created_at: datetime = None
    result: str = None

    def __post_init__(self):
        if not self.created_at:
            self.created_at = datetime.now()


# ============================================================================
# 工具函数
# ============================================================================

@tool
def query_order(order_id: str) -> str:
    """查询订单信息"""
    order_id = order_id.strip().upper()

    if order_id in MOCK_ORDERS:
        order = MOCK_ORDERS[order_id]
        result = f"""📦 订单信息查询结果

订单号: {order['order_id']}
客户姓名: {order['customer']}
商品名称: {order['product']}
购买数量: {order['quantity']}
订单金额: ¥{order['price']:,.2f}
订单状态: {order['status']}
下单日期: {order['order_date']}"""

        if order['tracking']:
            result += f"\n物流单号: {order['tracking']}\n物流状态: 运输中，预计3-5天送达"

        return result
    else:
        return f"❌ 未找到订单号 {order_id}，请检查订单号是否正确"


@tool
def check_logistics(tracking_number: str) -> str:
    """查询物流信息"""
    return f"""🚚 物流信息查询

物流单号: {tracking_number}
当前状态: 运输中

物流轨迹:
• 2024-12-10 14:30 - 已揽收
• 2024-12-10 18:45 - 到达转运中心
• 2024-12-11 08:20 - 运输中
• 2024-12-11 15:00 - 到达目的地城市
• 预计送达: 2024-12-12

配送员: 王师傅
联系电话: 138****5678"""


@tool
def search_products(category: str, max_price: float = None) -> str:
    """搜索产品"""
    if category not in MOCK_PRODUCTS:
        return f"❌ 未找到类别 '{category}'，可选类别: 手机、电脑、耳机"

    products = MOCK_PRODUCTS[category]

    if max_price:
        products = [p for p in products if p['price'] <= max_price]

    if not products:
        return f"❌ 在 {category} 类别中未找到符合条件的产品"

    result = f"🛍️ {category} 产品推荐\n\n"

    for i, product in enumerate(products, 1):
        result += f"{i}. {product['name']}\n"
        result += f"   💰 价格: ¥{product['price']:,}\n"
        result += f"   📦 库存: {product['stock']} 件\n"
        result += f"   ⭐ 评分: {product['rating']}/5.0\n"
        result += f"   {'✅ 推荐' if product['rating'] >= 4.8 else '👍 不错'}\n\n"

    return result


@tool
def compare_products(product_names: str) -> str:
    """比较产品"""
    names = [n.strip() for n in product_names.split(',')]

    result = "📊 产品对比分析\n\n"

    found_products = []
    for category, products in MOCK_PRODUCTS.items():
        for product in products:
            if product['name'] in names:
                found_products.append(product)

    if len(found_products) < 2:
        return "❌ 请提供至少两个有效的产品名称进行对比"

    result += "| 产品名称 | 价格 | 库存 | 评分 |\n"
    result += "|---------|------|------|------|\n"

    for product in found_products:
        result += f"| {product['name']} | ¥{product['price']:,} | {product['stock']} | {product['rating']} |\n"

    best = max(found_products, key=lambda x: x['rating'])
    cheapest = min(found_products, key=lambda x: x['price'])

    result += f"\n💡 推荐建议:\n"
    result += f"• 性价比之选: {cheapest['name']} (¥{cheapest['price']:,})\n"
    result += f"• 口碑最佳: {best['name']} (评分 {best['rating']})\n"

    return result


@tool
def submit_return_request(order_id: str, reason: str) -> str:
    """提交退货申请"""
    order_id = order_id.strip().upper()

    if order_id not in MOCK_ORDERS:
        return f"❌ 订单号 {order_id} 不存在"

    order = MOCK_ORDERS[order_id]
    return_id = f"RET-{uuid.uuid4().hex[:8].upper()}"

    return f"""✅ 退货申请已提交

退货单号: {return_id}
订单号: {order_id}
商品名称: {order['product']}
退货原因: {reason}
申请时间: {datetime.now().strftime('%Y-%m-%d %H:%M')}

处理流程:
1. ✅ 申请已提交
2. ⏳ 等待审核 (1-2个工作日)
3. ⏳ 审核通过后安排上门取件
4. ⏳ 收到商品后退款

预计退款时间: 3-7个工作日
客服电话: 400-123-4567"""


@tool
def submit_complaint(complaint_type: str, description: str) -> str:
    """提交投诉"""
    complaint_id = f"CMP-{uuid.uuid4().hex[:8].upper()}"

    return f"""📝 投诉已受理

投诉单号: {complaint_id}
投诉类型: {complaint_type}
投诉内容: {description}
受理时间: {datetime.now().strftime('%Y-%m-%d %H:%M')}

处理承诺:
• 24小时内专人联系
• 3个工作日内给出处理方案
• 全程跟踪直至问题解决

客服热线: 400-123-4567"""


# ============================================================================
# 创建智能体
# ============================================================================

def create_order_agent():
    """订单查询智能体"""
    return create_react_agent(
        model=llm,
        tools=[query_order, check_logistics],
        prompt=(
            "你是订单查询专员。\n"
            "使用工具查询实际数据，用中文回复。\n"
            "获得结果后直接回复，不要调用其他工具。"
        ),
        name="order_agent"
    )


def create_recommend_agent():
    """产品推荐智能体"""
    return create_react_agent(
        model=llm,
        tools=[search_products, compare_products],
        prompt=(
            "你是产品推荐专员。\n"
            "使用工具搜索实际产品数据，用中文回复。\n"
            "获得结果后直接回复，不要调用其他工具。"
        ),
        name="recommend_agent"
    )


def create_service_agent():
    """售后服务智能体"""
    return create_react_agent(
        model=llm,
        tools=[submit_return_request],
        prompt=(
            "你是售后服务专员。\n"
            "使用工具提交实际申请，用中文回复。\n"
            "获得结果后直接回复，不要调用其他工具。"
        ),
        name="service_agent"
    )


def create_complaint_agent():
    """投诉处理智能体"""
    return create_react_agent(
        model=llm,
        tools=[submit_complaint],
        prompt=(
            "你是投诉处理专员。\n"
            "使用工具提交实际投诉记录，用中文回复。\n"
            "获得结果后直接回复，不要调用其他工具。"
        ),
        name="complaint_agent"
    )


# ============================================================================
# 优化的客户服务系统
# ============================================================================

class CustomerServiceSystem:
    """客户服务系统 - 优化版"""

    def __init__(self):
        self.tasks: List[ServiceTask] = []
        self.supervisor = self._create_supervisor()

    def _create_supervisor(self):
        """创建supervisor"""
        return create_supervisor(
            model=llm,
            agents=[
                create_order_agent(),
                create_recommend_agent(),
                create_service_agent(),
                create_complaint_agent()
            ],
            prompt=(
                "你是客户服务主管，协调四个团队:\n"
                "• order_agent - 订单查询、物流追踪\n"
                "• recommend_agent - 产品推荐、产品对比\n"
                "• service_agent - 退换货、售后服务\n"
                "• complaint_agent - 投诉受理、问题反馈\n\n"
                "根据需求选择合适团队，直接转发结果。\n"
                "不要总结或改写，专员完成后任务即结束。"
            ),
            add_handoff_back_messages=False,  # 🔥 关键：禁用回传
            output_mode="full_history"
        ).compile()

    def add_task(self, description: str) -> str:
        """添加任务"""
        task = ServiceTask(
            task_id=f"TASK-{len(self.tasks) + 1:03d}",
            description=description
        )
        self.tasks.append(task)
        print(f"✅ [{task.task_id}] {description[:50]}...")
        return task.task_id

    def add_batch_tasks(self, descriptions: List[str]):
        """批量添加任务"""
        print(f"\n📋 添加 {len(descriptions)} 个任务")
        print("=" * 60)
        for desc in descriptions:
            self.add_task(desc)
        print("=" * 60)

    def process_all_tasks(self):
        """处理所有任务 - 优化版"""
        pending_tasks = [t for t in self.tasks if t.status == TaskStatus.PENDING]

        if not pending_tasks:
            print("📭 没有待处理的任务")
            return

        print(f"\n🎯 处理 {len(pending_tasks)} 个任务")
        print("=" * 60)

        for i, task in enumerate(pending_tasks, 1):
            print(f"\n[{i}/{len(pending_tasks)}] {task.task_id}: {task.description}")
            print("-" * 60)

            try:
                tool_results = []

                # 只收集工具结果
                for chunk in self.supervisor.stream({"messages": [HumanMessage(content=task.description)]}):
                    if isinstance(chunk, dict):
                        for node_data in chunk.values():
                            if "messages" in node_data:
                                for msg in node_data["messages"]:
                                    # 只提取工具执行结果
                                    if isinstance(msg, ToolMessage):
                                        # 过滤掉 transfer 消息
                                        if "transfer" not in msg.content.lower() and "supervisor" not in msg.content.lower():
                                            tool_results.append(msg.content)

                # 保存结果
                task.result = "\n\n".join(tool_results) if tool_results else "任务完成"
                task.status = TaskStatus.COMPLETED

                # 显示最终结果
                print(f"\n📄 最终结果:")
                print(task.result)
                print("\n✅ 完成")
                print("=" * 60)

            except Exception as e:
                print(f"❌ 失败: {e}")
                task.status = TaskStatus.FAILED
                import traceback
                traceback.print_exc()
                print("=" * 60)

        self.print_summary()

    def print_summary(self):
        """打印摘要"""
        total = len(self.tasks)
        completed = sum(1 for t in self.tasks if t.status == TaskStatus.COMPLETED)
        failed = sum(1 for t in self.tasks if t.status == TaskStatus.FAILED)

        print(f"\n📊 摘要: 总计 {total} | 完成 {completed} | 失败 {failed}")


# ============================================================================
# 演示
# ============================================================================

def run_demo():
    """运行演示"""
    print("🎉 智能客户服务系统")
    print("=" * 60)

    system = CustomerServiceSystem()

    tasks = [
        # "我想查询订单号 ORD001 的物流信息",
        "我想买一部5000-7000元的手机，有什么推荐？",
    ]

    system.add_batch_tasks(tasks)
    system.process_all_tasks()


def main():
    try:
        run_demo()
    except Exception as e:
        print(f"❌ 程序错误: {e}")
        import traceback
        traceback.print_exc()


if __name__ == "__main__":
    main()
```

