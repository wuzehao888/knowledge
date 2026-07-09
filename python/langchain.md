# LangChain 核心知识详解

## 目录

1. [概述](#概述)
2. [核心架构总览](#核心架构总览)
3. [Model（模型）](#model模型)
4. [Prompt（提示词模板）](#prompt提示词模板)
5. [Output Parser（输出解析器）](#output-parser输出解析器)
6. [LCEL（LangChain 表达式语言）](#lcellangchain-表达式语言)
7. [Chain（链）](#chain链)
8. [Memory（记忆）](#memory记忆)
9. [Embedding（词嵌入）](#embedding词嵌入)
10. [Vector Store（向量存储）](#vector-store向量存储)
11. [Retriever（检索器）](#retriever检索器)
12. [RAG（检索增强生成）](#rag检索增强生成)
13. [Tool（工具）](#tool工具)
14. [Agent（智能体）](#agent智能体)
15. [Callback（回调）](#callback回调)
16. [LangGraph（图式工作流）](#langgraph图式工作流)
17. [LangSmith（观测与评估）](#langsmith观测与评估)
18. [实战案例](#实战案例)
19. [最佳实践](#最佳实践)
20. [总结](#总结)

---

## 概述

### 什么是 LangChain？

LangChain 是一个用于**构建大语言模型（LLM）应用**的开源框架。它把「调用模型」「组织提示词」「连接外部数据」「让模型使用工具」这些繁琐的工作，封装成一个个可组合的积木，让你像搭乐高一样拼出复杂的 AI 应用。

### 为什么需要 LangChain？

直接调用 OpenAI 等 API 其实很简单，但当你想做下面这些事时，就会变得很麻烦：

| 需求 | 裸调 API 的痛点 | LangChain 的解决方式 |
|------|----------------|---------------------|
| 模型可切换 | 代码和厂商绑定 | 统一的 Model 接口，换一行就能换模型 |
| 组织提示词 | 手动拼接字符串，容易出错 | PromptTemplate 模板化 |
| 解析输出 | 模型返回的是纯文本，需要自己提取结构 | Output Parser 自动解析 |
| 记住上下文 | 每次都要手动拼接历史对话 | 消息历史管理 + LangGraph 持久化 |
| 接入私有数据 | 模型不知道你的文档 | RAG 检索增强生成 |
| 让模型做事 | 模型只会说话，不会执行 | Tool + Agent 工具调用 |

> **本文基于 LangChain 1.0+ 版本**。v1.0 是 LangChain 首个稳定版本，引入了 `create_agent`、middleware、`content_blocks` 等新特性，旧版 `ConversationChain`、`AgentExecutor` 等已移至 `langchain-classic` 包。

### 一个最简例子

```python
# pip install langchain langchain-openai

from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.output_parsers import StrOutputParser

# 1. 创建模型
model = ChatOpenAI(model="gpt-4o-mini")

# 2. 创建提示词模板
prompt = ChatPromptTemplate.from_template("用一句话解释什么是 {topic}")

# 3. 用管道符串起来：提示词 -> 模型 -> 解析器
chain = prompt | model | StrOutputParser()

# 4. 运行
result = chain.invoke({"topic": "量子计算"})
print(result)
# 输出: 量子计算是利用量子叠加和纠缠原理进行并行运算的新型计算技术。
```

> 上面用到的 `|` 管道符就是 LCEL（LangChain 表达式语言），后续会详细讲解。

---

## 核心架构总览

LangChain 的核心思想是**组件化 + 可组合**。所有组件都遵循统一的 Runnable 接口，可以用 `|` 串联。

```
┌─────────────────────────────────────────────────────────┐
│                    LangChain 应用                         │
│                                                          │
│   ┌──────────┐   ┌──────────┐   ┌──────────────┐        │
│   │  Prompt  │──▶│  Model   │──▶│ Output Parser│        │
│   │  提示词  │   │   模型   │   │   输出解析   │        │
│   └──────────┘   └──────────┘   └──────────────┘        │
│         │              │              │                  │
│         └──────────────┼──────────────┘                  │
│                        ▼                                 │
│                   ┌──────────┐                           │
│                   │  Chain   │  链：把上面三者串起来      │
│                   └──────────┘                           │
│                                                          │
│   ┌──────────┐   ┌────────────┐   ┌──────────┐          │
│   │ Memory   │   │  Retriever │   │   Tool   │          │
│   │  记忆    │   │  检索器    │   │   工具   │          │
│   └──────────┘   └────────────┘   └──────────┘          │
│         │              │              │                  │
│         └──────────────┼──────────────┘                  │
│                        ▼                                 │
│                   ┌──────────┐                           │
│                   │  Agent   │  智能体：自主决策+工具调用 │
│                   └──────────┘                           │
│                                                          │
│   底层支撑: Embedding(嵌入) + Vector Store(向量存储)      │
└─────────────────────────────────────────────────────────┘
```

### Runnable 统一接口

LangChain 所有组件都实现了 `Runnable` 接口，拥有以下标准方法：

| 方法 | 作用 | 适用场景 |
|------|------|---------|
| `invoke(input)` | 处理单个输入，返回单个输出 | 最常用，同步调用 |
| `batch(inputs)` | 批量处理多个输入 | 提高吞吐量 |
| `stream(input)` | 流式返回（逐 token） | 打字机效果 |
| `ainvoke(input)` | 异步处理单个输入 | 高并发场景 |
| `abatch(inputs)` | 异步批量处理 | 高并发批量 |

```python
# 所有组件都可以用同样的方式调用
result = prompt.invoke({"topic": "AI"})
result = model.invoke("你好")
result = parser.invoke(some_message)

# 也都支持流式
for chunk in chain.stream({"topic": "AI"}):
    print(chunk, end="", flush=True)
```

---

## Model（模型）

Model 是 LangChain 中**与 LLM 交互的统一入口**。无论你用 OpenAI、Anthropic、通义千问还是本地模型，都通过同样的接口调用。

### 两种模型类型

LangChain 把模型分为两大类：

| 类型 | 类名 | 输入 | 输出 | 特点 |
|------|------|------|------|------|
| **LLM** | `LLM` | 纯文本字符串 | 纯文本字符串 | 旧式，输入输出都是字符串 |
| **ChatModel** | `ChatModel` | 消息列表 | 消息对象 | 新式，支持多轮对话、角色区分 |

> 现在实际开发中**几乎都用 ChatModel**，即使是单轮问答也用它，因为它功能更强大、更灵活。LLM 类型主要存在于旧代码中。

### ChatModel 的消息类型

ChatModel 接收的消息有不同角色：

```python
from langchain.messages import (
    SystemMessage,    # 系统消息：设定 AI 的人设和规则
    HumanMessage,     # 人类消息：用户的输入
    AIMessage,        # AI 消息：模型的回复
    ToolMessage       # 工具消息：工具执行的结果
)

from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

messages = [
    SystemMessage(content="你是一个简洁的技术翻译官"),
    HumanMessage(content="把 'garbage collection' 翻译成中文并解释"),
]

response = model.invoke(messages)
print(response.content)
# 输出: 垃圾回收（GC）是自动释放程序不再使用的内存的机制...
```

### 角色对照表

| LangChain 消息 | OpenAI 角色 | 含义 |
|----------------|------------|------|
| `SystemMessage` | `system` | 给模型设定身份、规则、约束 |
| `HumanMessage` | `user` | 用户的提问或指令 |
| `AIMessage` | `assistant` | 模型之前的回复 |
| `ToolMessage` | `tool` | 工具调用的返回结果 |

### 模型可切换

切换模型只需改一行，其余代码完全不变：

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_community.chat_models.tongyi import ChatTongyi
from langchain_ollama import ChatOllama
```

### `init_chat_model`：统一初始化（v1.0 推荐）

v1.0 提供了 `init_chat_model()`，用一个函数初始化任何模型，不用记住不同厂商的类名：

```python
from langchain.chat_models import init_chat_model

# 用字符串指定模型，自动选择对应的集成包
model = init_chat_model("gpt-4o-mini", model_provider="openai")
model = init_chat_model("claude-sonnet-4-6", model_provider="anthropic")
model = init_chat_model("llama3", model_provider="ollama")

# 也支持可配置方式（适合从配置文件读取）
model = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai",
    temperature=0,
    max_tokens=1000,
)
```

> `init_chat_model` 的好处：换模型只改字符串，不用改 import，特别适合从环境变量/配置文件读取模型配置。

### 模型的常用参数

```python
model = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0.7,    # 0=确定性强, 1=随机性强
    max_tokens=1000,    # 最大输出 token 数
    timeout=30,         # 超时秒数
    stop=["\n\n"],      # 遇到则停止生成
)
```

**temperature 参数详解：**

| temperature | 行为 | 适用场景 |
|------------|------|---------|
| 0 | 每次回答几乎相同 | 代码生成、数据抽取、事实问答 |
| 0.3-0.5 | 略有变化 | 翻译、总结 |
| 0.7-1.0 | 较有创意 | 头脑风暴、创意写作 |
| >1.0 | 非常随机 | 一般不推荐 |

### 流式输出

```python
# 逐 token 输出，实现打字机效果
for chunk in model.stream("讲一个关于 Python 的冷笑话"):
    print(chunk.content, end="", flush=True)
# 输出: 为什么 Python 程序员不怕黑夜？因为他们有 with open() 照亮一切。
```

### `content_blocks`：统一内容访问（v1.0 新特性）

不同模型厂商返回的内容格式不同（有的有推理过程、有的有引用、有的有工具调用）。v1.0 引入了 `content_blocks`，用**统一的 API** 访问所有类型的内容：

```python
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-sonnet-4-6")
response = model.invoke("法国的首都是哪里？")

# 统一访问内容块，不管用哪个厂商的模型
for block in response.content_blocks:
    if block["type"] == "reasoning":
        print(f"模型推理过程: {block['reasoning']}")  # 思考过程
    elif block["type"] == "text":
        print(f"回答: {block['text']}")                # 文本回答
    elif block["type"] == "tool_call":
        print(f"工具调用: {block['name']}({block['args']})")  # 工具调用
```

> `content_blocks` 的价值：**一套代码兼容所有厂商**。不管用 OpenAI、Anthropic 还是 Google，访问推理过程、工具调用、文本内容的代码完全一样。

---

## Prompt（提示词模板）

Prompt 是**给模型的输入模板**。直接拼字符串容易出错且难维护，LangChain 提供了模板化的方式。

### 为什么需要模板？

```python
# ❌ 手动拼接，容易出错
user_input = "量子力学"
prompt_str = f"请用一句话解释什么是{user_input}，要求通俗易懂"

# ✅ 模板化，可复用、可校验
from langchain.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template("请用一句话解释什么是{topic}，要求通俗易懂")
formatted = prompt.invoke({"topic": "量子力学"})
# prompt.invoke 返回 ChatPromptValue，包含格式化后的消息
```

### ChatPromptTemplate（最常用）

```python
from langchain.prompts import ChatPromptTemplate

# 方式一：from_messages（推荐，结构清晰）
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}，回答要{style}"),
    ("human", "{question}"),
])

result = prompt.invoke({
    "role": "中学老师",
    "style": "通俗易懂",
    "question": "什么是黑洞？"
})
print(result)
# 输出:
# System: 你是一个中学老师，回答要通俗易懂
# Human: 什么是黑洞？

# 方式二：from_template（单条消息）
simple_prompt = ChatPromptTemplate.from_template("翻译这句话: {text}")
```

### 模板变量语法

```python
# 基本变量: {variable}
prompt = ChatPromptTemplate.from_template("你好，{name}！")

# 在 from_messages 中使用元组:
#   ("system", "内容")  → SystemMessage
#   ("human",  "内容")  → HumanMessage
#   ("ai",     "内容")  → AIMessage
```

### Few-Shot Prompting（少样本提示）

给模型几个示例，让它「照葫芦画瓢」：

```python
from langchain.prompts import ChatPromptTemplate
from langchain.prompts import FewShotChatMessagePromptTemplate

# 示例数据
examples = [
    {"input": "开心", "output": "😀 今天阳光真好！"},
    {"input": "难过", "output": "😢 需要一个拥抱"},
    {"input": "生气", "output": "😤 深呼吸，冷静一下"},
]

# 示例模板
example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

# 组装成 Few-Shot 提示词
few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=example_prompt,
    examples=examples,
)

# 最终提示词 = 系统指令 + 示例 + 真实问题
final_prompt = ChatPromptTemplate.from_messages([
    ("system", "根据情绪，输出对应的 emoji 和一句话"),
    few_shot_prompt,
    ("human", "{input}"),
])

result = final_prompt.invoke({"input": "兴奋"})
# 模型会模仿示例格式，输出类似: 🤩 太棒了！
```

### Partial（部分填充）

先填一部分变量，剩下的稍后填：

```python
prompt = ChatPromptTemplate.from_template("{greeting}, {name}! 今天是{day}")

# 先固定 greeting 和 day
partial = prompt.partial(greeting="你好", day="周三")

# 后面再补 name
result = partial.invoke({"name": "小明"})
# "你好, 小明! 今天是周三"
```

---

## Output Parser（输出解析器）

Output Parser 负责**把模型返回的纯文本，解析成你想要的数据结构**（字典、列表、对象等）。

### 工作流程

```
模型输出（纯文本）
      │
      ▼
  Output Parser
      │
      ▼
结构化数据（dict / list / Pydantic 对象）
```

### 1. StrOutputParser（最简单）

直接提取模型的文本内容，去掉消息的元数据：

```python
from langchain.output_parsers import StrOutputParser

parser = StrOutputParser()

# 模型返回的是 AIMessage 对象
# parser 把它变成纯字符串
# AIMessage(content="你好") → "你好"
```

### 2. JsonOutputParser（解析 JSON）

让模型输出 JSON，并自动解析：

```python
from langchain.output_parsers import JsonOutputParser

parser = JsonOutputParser()

# 配合提示词，要求模型输出 JSON
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个 JSON 生成器，只输出合法 JSON"),
    ("human", "生成一个关于{animal}的信息，包含 name 和 habitat 字段\n{format_instructions}"),
])

# 自动注入格式说明
prompt = prompt.partial(
    format_instructions=parser.get_format_instructions()
)
```

### 3. PydanticOutputParser（解析为对象，最强大）

把输出解析为 Pydantic 模型，自带类型校验：

```python
from pydantic import BaseModel, Field
from langchain.output_parsers import PydanticOutputParser

# 1. 定义你想要的数据结构
class BookInfo(BaseModel):
    title: str = Field(description="书名")
    author: str = Field(description="作者")
    year: int = Field(description="出版年份")
    tags: list[str] = Field(description="标签列表")

# 2. 创建解析器
parser = PydanticOutputParser(pydantic_object=BookInfo)

# 3. 把格式说明注入提示词
prompt = ChatPromptTemplate.from_template(
    "提取这本书的信息: {book_desc}\n{format_instructions}"
).partial(format_instructions=parser.get_format_instructions())

# 4. 组装运行
chain = prompt | model | parser

result = chain.invoke({"book_desc": "《三体》刘慈欣2008年出版的科幻小说"})
print(type(result))       # <class 'BookInfo'>
print(result.title)       # 三体
print(result.author)      # 刘慈欣
print(result.year)        # 2008
print(result.tags)        # ['科幻', '硬科幻']
```

### 三种解析器对比

| 解析器 | 输出类型 | 校验 | 适用场景 |
|--------|---------|------|---------|
| `StrOutputParser` | 字符串 | 无 | 最简单，只要文本 |
| `JsonOutputParser` | dict | 无 | 只要 JSON 字典 |
| `PydanticOutputParser` | Pydantic 对象 | 有（类型校验） | 需要结构化+类型安全 |

### 自动重试（OutputFixingParser）

模型偶尔会输出格式错误的内容，可以用修复解析器自动重试：

```python
from langchain.output_parsers import OutputFixingParser

# 如果解析失败，会自动让模型重新格式化
fixing_parser = OutputFixingParser.from_llm(parser=parser, llm=model)
```

---

## LCEL（LangChain 表达式语言）

LCEL（LangChain Expression Language）是 LangChain 的**核心语法**，用 `|` 管道符把组件串联成链。它让代码简洁、可读性强，同时自带流式、异步、批量等能力。

### 基本语法

```python
chain = prompt | model | parser
# 等价理解: 数据从左到右流过每个组件
# prompt → 生成提示词 → model → 生成回复 → parser → 解析输出
```

### 为什么用 LCEL？

用 `|` 串联的链，**自动获得**以下能力（不用自己写）：

| 能力 | 说明 |
|------|------|
| 流式输出 | `chain.stream()` 自动逐 token 返回 |
| 异步支持 | `chain.ainvoke()` 自动异步执行 |
| 批量处理 | `chain.batch()` 自动并行处理 |
| 重试机制 | 配置后自动重试失败请求 |
| 可观测性 | 配合 LangSmith 自动追踪每一步 |
| 回调支持 | 每一步都能触发回调 |

### 完整示例

```python
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.output_parsers import StrOutputParser

model = ChatOpenAI(model="gpt-4o-mini")

prompt = ChatPromptTemplate.from_template("用{words}个字总结: {text}")

chain = prompt | model | StrOutputParser()

# 同步调用
print(chain.invoke({"words": 20, "text": "Python是一种解释型高级编程语言..."}))

# 流式调用（打字机效果）
for chunk in chain.stream({"words": 20, "text": "Python是一种解释型高级编程语言..."}):
    print(chunk, end="", flush=True)

# 批量调用（并行处理多个输入）
results = chain.batch([
    {"words": 10, "text": "Java是面向对象的语言..."},
    {"words": 10, "text": "Go是谷歌开发的并发语言..."},
    {"words": 10, "text": "Rust是内存安全的系统语言..."},
])
```

### RunnablePassthrough：原样传递

有时候你想把原始输入同时传给模型和保留下来，用 `RunnablePassthrough`：

```python
from langchain.runnables import RunnablePassthrough

# RunnablePassthrough: 输入什么就输出什么（透传）
chain = RunnablePassthrough()
chain.invoke("hello")  # "hello"

# assign: 在透传的同时，添加新字段
chain = RunnablePassthrough.assign(length=lambda x: len(x["text"]))
chain.invoke({"text": "hello"})  # {"text": "hello", "length": 5}
```

### RunnableLambda：把普通函数变成组件

```python
from langchain.runnables import RunnableLambda

# 普通函数
def word_count(text: str) -> str:
    return f"字数: {len(text)}"

# 包装成 Runnable，就能放进管道
count_runnable = RunnableLambda(word_count)

chain = prompt | model | StrOutputParser() | count_runnable
# 文本 → 提示词 → 模型 → 解析为字符串 → 统计字数
```

### RunnableParallel：并行执行

```python
from langchain.runnables import RunnableParallel

# 同时生成两个不同视角的总结
chain = RunnableParallel(
    简洁版=(simple_prompt | model | StrOutputParser()),
    详细版=(detail_prompt | model | StrOutputParser()),
)

result = chain.invoke({"topic": "机器学习"})
# result = {"简洁版": "...", "详细版": "..."}
```

---

## Chain（链）

Chain 是**把多个步骤组合成整体**的概念。在 v1.0 中，旧的 Chain 类（如 `LLMChain`、`ConversationChain`）已移至 `langchain-classic` 包，官方推荐用 LCEL（`|` 管道符）自己拼，更灵活。

### 旧方式 vs 新方式

```python
# ❌ 旧方式（v1.0 已移至 langchain-classic，不推荐）
# pip install langchain-classic
# from langchain_classic.chains import LLMChain
# chain = LLMChain(llm=model, prompt=prompt)
# result = chain.run(topic="量子计算")

# ✅ 新方式（LCEL，v1.0 标准）
chain = prompt | model | StrOutputParser()
result = chain.invoke({"topic": "量子计算"})
```

### 常见链模式

#### 1. 顺序链

前一步的输出作为后一步的输入：

```python
from langchain.runnables import RunnableLambda

# 第一步：生成一个主题
step1 = ChatPromptTemplate.from_template("生成一个关于{field}的有趣话题") | model | StrOutputParser()

# 第二步：根据话题写文章
step2 = ChatPromptTemplate.from_template("就这个话题写一段话: {topic}") | model | StrOutputParser()

# 串联：注意变量名要对应
# step1 输出是字符串，需要包装成 {"topic": ...} 给 step2
chain = step1 | RunnableLambda(lambda x: {"topic": x}) | step2

print(chain.invoke({"field": "太空探索"}))
```

#### 2. 路由链

根据输入内容，动态选择不同的处理链：

```python
from langchain.runnables import RunnableLambda

# 三个不同的处理链
math_chain = ChatPromptTemplate.from_template("你是数学老师，回答: {question}") | model | StrOutputParser()
history_chain = ChatPromptTemplate.from_template("你是历史老师，回答: {question}") | model | StrOutputParser()
code_chain = ChatPromptTemplate.from_template("你是程序员，回答: {question}") | model | StrOutputParser()

# 路由函数：根据问题内容选择链
def route(info):
    if "数学" in info["question"] or "计算" in info["question"]:
        return math_chain
    elif "历史" in info["question"]:
        return history_chain
    else:
        return code_chain

# RunnablePassthrough + 路由
from langchain.runnables import RunnablePassthrough
chain = {"question": RunnablePassthrough()} | RunnableLambda(route)

print(chain.invoke("帮我计算 123 * 456"))
```

### Chain vs LCEL 的关系

```
Chain 是概念（把步骤组合起来）
LCEL 是实现方式（用 | 管道符）
```

> 可以理解为：LCEL 就是构建 Chain 的现代语法。

---

## Memory（记忆）

模型本身是**无状态**的——每次调用都是独立的，它不记得上一次说了什么。在 v1.0 中，旧的 Memory 类（`ConversationBufferMemory` 等）已移至 `langchain-classic`，官方推荐两种新方式：**手动管理消息列表**（简单场景）和 **LangGraph checkpoint**（需要持久化的场景）。

### 为什么需要记忆？

```
无 Memory:
  用户: 我叫小明
  AI:   你好小明！
  用户: 我叫什么？     ← 模型完全不知道
  AI:   抱歉，我不知道你的名字

有 Memory:
  用户: 我叫小明
  AI:   你好小明！
  用户: 我叫什么？     ← Memory 保存了历史
  AI:   你叫小明
```

### Memory 的工作原理

```
┌────────────────────────────────────────────────┐
│                  对话循环                        │
│                                                │
│   用户输入 ──▶ 拼接历史消息 ──▶ 发给模型          │
│                    ▲                    │       │
│                    │                    ▼       │
│              ┌──────────┐         AI 回复        │
│              │  Memory  │ ◀──────────┘          │
│              │  存储    │ 保存新的一轮对话        │
│              └──────────┘                       │
└────────────────────────────────────────────────┘
```

### v1.0 的两种记忆方式

| 方式 | 适用场景 | 特点 |
|------|---------|------|
| **手动管理消息列表** | 简单对话、单次会话 | 最灵活，自己控制历史消息的拼接和裁剪 |
| **LangGraph checkpoint** | 需要持久化、跨会话、人机协作 | 自动持久化，支持断点续跑、时间旅行 |
| ~~ConversationBufferMemory~~ | ~~旧代码~~ | 已移至 `langchain-classic`，不推荐 |

### 方式一：手动管理消息列表（推荐）

v1.0 推荐直接管理消息列表，不依赖任何 Memory 类——最灵活、最透明：

```python
from langchain.messages import SystemMessage, HumanMessage, AIMessage
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")

# 用一个列表保存所有对话消息
history = [SystemMessage(content="你是一个友好的助手")]

def chat(user_input: str) -> str:
    history.append(HumanMessage(content=user_input))
    response = model.invoke(history)   # 把完整历史发给模型
    history.append(response)            # 把 AI 回复也存进历史
    return response.content

print(chat("我叫小明，今年25岁"))
print(chat("我叫什么名字？"))  # 记得：小明
print(chat("我今年几岁？"))     # 记得：25
```

> 原理很简单：每次调用时，把**全部历史消息** + **新问题**一起发给模型，模型自然就「记住」了上下文。

### 控制历史长度：`trim_messages`

对话越长，历史消息越多，token 消耗越大。v1.0 提供了 `trim_messages` 来裁剪历史：

```python
from langchain.messages import trim_messages, SystemMessage, HumanMessage, AIMessage

# 只保留最近的消息，控制 token 数量
trimmed = trim_messages(
    history,
    max_tokens=500,                # 最多保留 500 token
    strategy="last",               # 保留最后（最近的）消息
    token_counter=len,             # 简单按消息条数算，也可用模型的 token 计数器
    start_on="human",              # 确保从 human 消息开始（避免截断不完整）
    include_system=True,           # 始终保留 system 消息
)

response = model.invoke(trimmed + [HumanMessage(content="继续聊")])
```

> `trim_messages` 相当于旧版的 `ConversationBufferWindowMemory`，但更灵活——你可以按 token 数、消息条数来裁剪。

### 方式二：LangGraph checkpoint（持久化记忆）

如果需要**跨会话保持记忆**（用户关掉程序再打开，还能接着聊），用 LangGraph 的 checkpoint：

```python
from langgraph.checkpoint.memory import MemorySaver
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
checkpointer = MemorySaver()

# create_agent 基于 LangGraph，支持 checkpoint 持久化
agent = create_agent(
    model="gpt-4o-mini",
    tools=[],
    checkpointer=checkpointer,
)

# 用 thread_id 标识会话
config = {"configurable": {"thread_id": "user-001"}}

# 第一轮对话
result = agent.invoke(
    {"messages": [{"role": "user", "content": "我叫小明"}]},
    config=config
)

# 第二轮对话（新的程序运行也能记住）
result = agent.invoke(
    {"messages": [{"role": "user", "content": "我叫什么？"}]},
    config=config
)
# → "你叫小明"（checkpoint 自动恢复了上一轮的对话）
```

> LangGraph checkpoint 的优势：自动持久化、支持时间旅行（回到任意历史节点）、支持人机协作（暂停等待人工确认）。详见 [LangGraph](#langgraph图式工作流) 章节。

### 记忆方式选择指南

```
需要记住上下文吗？
    │
    ├─ 不需要 → 直接调 model.invoke()，不用记忆
    │
    ├─ 需要，但只在单次会话内 → 手动管理消息列表 + trim_messages
    │
    └─ 需要，跨会话持久化 → LangGraph checkpoint
```

---

## Embedding（词嵌入）

Embedding 是把文本转换成**一串数字（向量）**的技术。这串数字代表了文本的「语义」，语义相近的文本，向量也相近。

### 直观理解

```
"猫"      → [0.2, 0.8, 0.1, ...]   ┐
"狗"      → [0.3, 0.7, 0.2, ...]   ┘ 向量接近（都是宠物）

"汽车"    → [0.9, 0.1, 0.8, ...]   ┐
"猫"      → [0.2, 0.8, 0.1, ...]   ┘ 向量差异大（语义无关）
```

### 为什么需要 Embedding？

模型只能理解数字，不能直接理解文本。Embedding 是**文本和模型之间的翻译层**，更是 RAG（检索增强生成）的基础——通过比较向量距离，就能找到语义最相关的文档。

### 代码示例

```python
from langchain_openai import OpenAIEmbeddings

# 创建嵌入模型
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 嵌入单个文本
vector = embeddings.embed_query("我喜欢吃苹果")
print(len(vector))  # 1536（向量维度）
print(vector[:5])   # [0.012, -0.034, 0.056, ...]

# 嵌入多个文本
texts = ["猫是宠物", "狗是宠物", "汽车是交通工具"]
vectors = embeddings.embed_documents(texts)
print(len(vectors))     # 3
print(len(vectors[0]))  # 1536
```

### 不同 Embedding 模型

```python
# OpenAI
from langchain_openai import OpenAIEmbeddings
emb = OpenAIEmbeddings(model="text-embedding-3-small")

# 通义千问
from langchain_community.embeddings import DashScopeEmbeddings
emb = DashScopeEmbeddings(model="text-embedding-v2")

# 本地 Ollama（免费离线）
from langchain_ollama import OllamaEmbeddings
emb = OllamaEmbeddings(model="nomic-embed-text")

# HuggingFace（免费开源）
from langchain_huggingface import HuggingFaceEmbeddings
emb = HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2")
```

---

## Vector Store（向量存储）

Vector Store 是**专门存储和检索向量**的数据库。你把文档的 Embedding 存进去，查询时就能快速找到语义最相似的内容。

### 工作流程

```
写入阶段:
  文档 → 切片 → Embedding → 存入 Vector Store

查询阶段:
  问题 → Embedding → 在 Vector Store 中找最相似的 → 返回相关文档
```

### 常用 Vector Store

| Vector Store | 类型 | 特点 |
|-------------|------|------|
| **Chroma** | 本地 | 轻量级，适合开发和测试 |
| **FAISS** | 本地 | Facebook 出品，纯内存，速度快 |
| **Pinecone** | 云服务 | 全托管，适合生产环境 |
| **Milvus** | 自部署 | 开源，支持大规模 |
| **pgvector** | 数据库扩展 | 基于 PostgreSQL |

### 代码示例（Chroma）

```python
# pip install langchain-chroma

from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1. 准备文档
docs = [
    "Python 是一种解释型编程语言",
    "Java 是一种面向对象编程语言",
    "Go 是谷歌开发的并发编程语言",
    "Rust 是一种内存安全的系统编程语言",
    "机器学习是人工智能的一个分支",
]

# 2. 创建嵌入模型
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 3. 存入向量数据库（自动嵌入）
vectorstore = Chroma.from_texts(
    texts=docs,
    embedding=embeddings,
    collection_name="lang_docs",
)

# 4. 相似度搜索
results = vectorstore.similarity_search("什么语言适合系统编程", k=2)
for doc in results:
    print(doc.page_content)
# 输出:
# Rust 是一种内存安全的系统编程语言
# Go 是谷歌开发的并发编程语言
```

### 带分数的搜索

```python
# similarity_search_with_score 返回 (文档, 距离分数)
# 分数越小越相似（L2 距离）
results = vectorstore.similarity_search_with_score("Python", k=3)
for doc, score in results:
    print(f"[{score:.4f}] {doc.page_content}")
# [0.1234] Python 是一种解释型编程语言
# [0.5678] Java 是一种面向对象编程语言
# ...
```

---

## Retriever（检索器）

Retriever 是**从数据源中检索相关文档的统一接口**。Vector Store 可以变成 Retriever，但 Retriever 不限于 Vector Store——也可以是从数据库、搜索引擎检索。

### Vector Store → Retriever

```python
# 把向量数据库转成检索器
retriever = vectorstore.as_retriever(
    search_type="similarity",  # 相似度搜索
    search_kwargs={"k": 3},    # 返回最相关的 3 条
)

# 使用
docs = retriever.invoke("系统编程用什么语言")
for doc in docs:
    print(doc.page_content)
```

### 检索方式

| search_type | 说明 |
|-------------|------|
| `"similarity"` | 纯相似度搜索（默认） |
| `"mmr"` | 最大边际相关性，兼顾相关性和多样性 |
| `"similarity_score_threshold"` | 设置相似度阈值，只返回足够相关的 |

```python
# MMR: 既相关又多样，避免返回内容重复的文档
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 3, "fetch_k": 10},
)
```

### Retriever 在 LCEL 中的用法

```python
from langchain.runnables import RunnablePassthrough
from langchain.output_parsers import StrOutputParser

# RAG 的核心链: 检索文档 → 拼进提示词 → 模型回答
prompt = ChatPromptTemplate.from_template("""
根据以下上下文回答问题。如果上下文中没有答案，就说不知道。

上下文: {context}

问题: {question}
""")

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)

print(rag_chain.invoke("Python 是什么类型的语言？"))
# 输出: 根据上下文，Python 是一种解释型编程语言。
```

---

## RAG（检索增强生成）

RAG（Retrieval-Augmented Generation）是 LangChain 最重要的应用模式：**先从你的私有数据中检索相关内容，再让模型基于这些内容回答**。

### 为什么需要 RAG？

| 问题 | 不用 RAG | 用 RAG |
|------|---------|--------|
| 模型不知道你的私有数据 | ✗ | ✓ 从你的文档检索 |
| 模型知识过时 | ✗ | ✓ 实时检索最新数据 |
| 模型会「幻觉」（编造） | 容易 | 引用真实文档，减少幻觉 |
| 需要标注信息来源 | ✗ | ✓ 可追溯到原文 |

### RAG 完整流程

```
┌─────────────┐     ┌──────────┐     ┌─────────────┐
│  原始文档    │────▶│  切片     │────▶│  Embedding  │
│  PDF/TXT/MD │     │ TextSplit│     │  词嵌入      │
└─────────────┘     └──────────┘     └──────┬──────┘
                                            │
                                            ▼
┌─────────────┐     ┌──────────┐     ┌─────────────┐
│  模型回答    │◀────│ 拼接提示词│◀────│  检索相关    │
│  Generation │     │  Prompt  │     │  文档片段    │
└─────────────┘     └──────────┘     └─────────────┘
      ▲                                      ▲
      │                                      │
   用户问题 ──────────────────────────────────┘
```

### 第一步：文档加载与切片

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 假设这是你的文档内容
raw_text = """
Python 是一种解释型、高级编程语言。它由 Guido van Rossum 于 1991 年创建。
Python 强调代码可读性，使用缩进来定义代码块。
Python 支持多种编程范式：面向对象、命令式、函数式和过程式。
Python 拥有庞大的标准库和丰富的第三方包生态系统。
Python 广泛应用于 Web 开发、数据科学、人工智能、自动化脚本等领域。
"""

# 切片：把长文档切成小片段
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=100,        # 每个片段最多 100 字符
    chunk_overlap=20,      # 片段之间重叠 20 字符（避免切断语义）
    separators=["\n\n", "\n", "。", "，", " "],  # 分隔符优先级
)

chunks = text_splitter.create_documents([raw_text])
print(f"切成了 {len(chunks)} 个片段")
for i, chunk in enumerate(chunks):
    print(f"片段{i}: {chunk.page_content[:30]}...")
```

### 第二步：存入向量数据库

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(chunks, embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 2})
```

### 第三步：构建 RAG 链

```python
from langchain.prompts import ChatPromptTemplate
from langchain.runnables import RunnablePassthrough
from langchain.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("""
你是文档助手。请仅根据以下上下文回答问题。
如果上下文中没有相关信息，请说"根据已知文档无法回答"。

上下文:
{context}

问题: {question}

回答:
""")

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough(),
    }
    | prompt
    | model
    | StrOutputParser()
)

# 提问
print(rag_chain.invoke("Python 是谁创建的？"))
# 输出: Python 由 Guido van Rossum 于 1991 年创建。

print(rag_chain.invoke("Java 有什么特点？"))
# 输出: 根据已知文档无法回答。
```

### RAG 进阶：引用来源

```python
from langchain.runnables import RunnableLambda

# 让检索结果带着来源信息一起输出
def format_docs_with_source(docs):
    return "\n\n".join(
        f"[来源{i+1}] {doc.page_content}" for i, doc in enumerate(docs)
    )

# 输出时同时返回文档和答案
rag_with_sources = RunnableParallel(
    answer=(
        {"context": retriever | format_docs_with_source, "question": RunnablePassthrough()}
        | prompt | model | StrOutputParser()
    ),
    sources=retriever,
)

result = rag_with_sources.invoke("Python 用在哪里？")
print("回答:", result["answer"])
print("来源文档数:", len(result["sources"]))
```

---

## Tool（工具）

Tool 是**让模型能够执行实际操作**的接口。模型本身只会「说话」，但通过 Tool，它可以查天气、搜网页、查数据库、运行代码、调用 API。

### 为什么需要 Tool？

```
没有 Tool:
  用户: 今天北京天气怎么样？
  AI:   我无法获取实时天气信息...（只会说，不会做）

有 Tool:
  用户: 今天北京天气怎么样？
  AI:   [调用天气 API] → 北京今天 28°C，晴
       北京今天晴，气温 28°C。（能做事了！）
```

### 创建自定义 Tool

#### 方式一：@tool 装饰器（推荐）

```python
from langchain.tools import tool

@tool
def add(a: int, b: int) -> int:
    """两个整数相加"""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """两个整数相乘"""
    return a * b

@tool
def get_word_length(word: str) -> int:
    """获取单词的字符长度"""
    return len(word)

# 直接调用（普通函数一样用）
print(add.invoke({"a": 3, "b": 5}))  # 8
print(get_word_length.invoke({"word": "hello"}))  # 5
```

> **关键点**：函数的**docstring 就是给模型看的工具说明**。模型通过 docstring 决定什么时候用这个工具、怎么用。所以 docstring 一定要写清楚！

#### 方式二：StructuredTool（更灵活）

```python
from langchain.tools import StructuredTool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")
    limit: int = Field(default=5, description="返回结果数量")

def search_func(query: str, limit: int = 5) -> str:
    return f"搜索 '{query}'，返回 {limit} 条结果"

search_tool = StructuredTool.from_function(
    func=search_func,
    name="search",
    description="搜索互联网获取信息",
    args_schema=SearchInput,
)
```

### Tool 的属性

```python
print(add.name)           # "add"
print(add.description)    # "两个整数相加"
print(add.args_schema)    # 参数的 schema（模型据此生成参数）
print(add.args)           # {'a': {'type': 'integer'}, 'b': {'type': 'integer'}}
```

### 内置工具

LangChain 提供了一些现成的工具：

```python
# Python 代码执行器
from langchain_experimental.tools import PythonREPLTool
python_tool = PythonREPLTool()

# DuckDuckGo 搜索
from langchain_community.tools import DuckDuckGoSearchRun
search_tool = DuckDuckGoSearchRun()
```

### Tool 的运行流程

```
1. 模型收到问题和可用工具列表
2. 模型决定: "我需要用 add 工具，参数 a=3, b=5"
3. LangChain 执行 add(3, 5) = 8
4. 把结果 8 返回给模型
5. 模型基于结果生成最终回答: "3 + 5 = 8"
```

---

## Agent（智能体）

Agent 是 LangChain 中**最高级的组件**。它能**自主决策**：面对一个问题，它自己决定该用什么工具、该不该搜索、该不该计算，像一个能独立思考的助手。

### Agent vs Chain 的区别

这是理解 Agent 的关键：

| 特性 | Chain（链） | Agent（智能体） |
|------|------------|----------------|
| 执行流程 | **固定**，预定义好步骤 | **动态**，模型自己决定下一步 |
| 谁来控制 | 开发者写死流程 | 模型自主决策 |
| 灵活性 | 低 | 高 |
| 可预测性 | 高 | 较低 |
| 类比 | 流水线工人 | 独立思考的助手 |

```
Chain:  输入 → 步骤A → 步骤B → 步骤C → 输出（固定路线）

Agent:  输入 → 模型思考"该干嘛？"
                    │
                    ├─ 需要搜索？ → 调搜索工具 → 拿到结果
                    │       │
                    │       └─ 再思考"还需要干嘛？"
                    │              │
                    │              ├─ 需要计算？ → 调计算工具 → 拿到结果
                    │              │       │
                    │              │       └─ 再思考"信息够了？" → 输出答案
                    │              └─ 信息够了 → 输出答案
                    └─ 信息够了 → 直接输出答案
```

### ReAct 模式

Agent 最常用的推理模式是 **ReAct（Reasoning + Acting）**：

```
Thought:  用户问 25 的平方根，我需要计算
Action:   使用 calculator 工具
Action Input: sqrt(25)
Observation: 5.0
Thought:  得到结果 5.0，可以回答了
Final Answer: 25 的平方根是 5
```

### 创建 Agent

v1.0 推荐使用 `create_agent`——它是构建 Agent 的标准方式，基于 LangGraph 构建，自带持久化、流式、人机协作：

```python
from langchain.agents import create_agent
from langchain.tools import tool
from langchain_openai import ChatOpenAI

# 1. 准备工具
@tool
def add(a: int, b: int) -> int:
    """两个整数相加"""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """两个整数相乘"""
    return a * b

@tool
def power(base: int, exp: int) -> int:
    """计算 base 的 exp 次方"""
    return base ** exp

# 2. 一行创建 Agent（不需要手动拼 prompt、不需要 AgentExecutor）
agent = create_agent(
    model="gpt-4o-mini",                          # 模型（字符串或对象都行）
    tools=[add, multiply, power],                  # 工具列表
    system_prompt="你是一个数学助手，可以使用提供的工具来计算",
)

# 3. 运行
result = agent.invoke({
    "messages": [{"role": "user", "content": "3 的 4 次方是多少？然后再乘以 2"}]
})
print(result["messages"][-1].content)
# 输出: 3 的 4 次方是 81，再乘以 2 等于 162。
```

> 对比旧方式（`create_tool_calling_agent` + `AgentExecutor` + 手动拼 prompt），`create_agent` 把这些都简化了——不需要 `agent_scratchpad`、不需要 `AgentExecutor`、不需要手动拼 prompt 模板。

### Agent 执行过程

```
Agent 收到: "3 的 4 次方是多少？然后再乘以 2"

第1轮思考: 我需要先计算 3 的 4 次方
  → 调用工具: power(base=3, exp=4)
  → 结果: 81

第2轮思考: 现在把 81 乘以 2
  → 调用工具: multiply(a=81, b=2)
  → 结果: 162

第3轮思考: 计算完成，不需要更多工具了
  → 最终回答: 3 的 4 次方是 81，再乘以 2 等于 162。
```

### Middleware：Agent 的中间件系统（v1.0 核心特性）

Middleware 是 `create_agent` 的杀手锏——它让你在 Agent 执行的每一步**插入自定义逻辑**，不用改 Agent 本身：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    SummarizationMiddleware,       # 自动总结长对话
    HumanInTheLoopMiddleware,      # 敏感操作需人工确认
    PIIMiddleware,                 # 自动脱敏敏感信息
)

agent = create_agent(
    model="gpt-4o-mini",
    tools=[read_email, send_email],
    middleware=[
        # 自动脱敏邮箱和手机号
        PIIMiddleware("email", strategy="redact"),
        PIIMiddleware("phone_number", strategy="block"),

        # 对话超过 500 token 时自动总结历史
        SummarizationMiddleware(
            model="gpt-4o-mini",
            trigger={"tokens": 500}
        ),

        # 发邮件前必须人工确认
        HumanInTheLoopMiddleware(
            interrupt_on={"send_email": {"allowed_decisions": ["approve", "reject"]}}
        ),
    ]
)
```

**Middleware 的钩子点：**

| 钩子 | 执行时机 | 典型用途 |
|------|---------|---------|
| `before_agent` | Agent 开始前 | 加载记忆、校验输入 |
| `before_model` | 每次调模型前 | 更新 prompt、裁剪消息 |
| `wrap_model_call` | 包裹模型调用 | 拦截/修改请求和响应 |
| `wrap_tool_call` | 包裹工具调用 | 拦截/修改工具执行 |
| `after_model` | 模型响应后 | 校验输出、加护栏 |
| `after_agent` | Agent 结束后 | 保存结果、清理 |

> Middleware 相当于旧版 `AgentExecutor` 的 `max_iterations`、`handle_parsing_errors` 等参数的升级版——更灵活、更可组合。

### 带记忆的 Agent

`create_agent` 基于 LangGraph，**内置 checkpoint 支持**，不需要额外的 Memory 类：

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import MemorySaver

agent = create_agent(
    model="gpt-4o-mini",
    tools=[add, multiply],
    system_prompt="你是一个数学助手",
    checkpointer=MemorySaver(),  # ← 一行加上记忆
)

# 用 thread_id 标识会话
config = {"configurable": {"thread_id": "session-1"}}

# 第一轮
agent.invoke(
    {"messages": [{"role": "user", "content": "3 + 5 等于多少？"}]},
    config=config
)
# → 8

# 第二轮（记得上一轮的结果）
agent.invoke(
    {"messages": [{"role": "user", "content": "再乘以 3 呢？"}]},
    config=config
)
# → 24（记得上面是 8，8 × 3 = 24）
```

### 结构化输出

v1.0 的 `create_agent` 支持直接输出结构化数据，不需要额外的 Output Parser：

```python
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy
from pydantic import BaseModel

class Weather(BaseModel):
    temperature: float
    condition: str

def weather_tool(city: str) -> str:
    """获取城市天气"""
    return f"{city} 晴天 25度"

agent = create_agent(
    "gpt-4o-mini",
    tools=[weather_tool],
    response_format=ToolStrategy(Weather),  # ← 直接输出 Pydantic 对象
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "北京天气怎么样？"}]
})
print(result["structured_response"])
# Weather(temperature=25.0, condition='晴')
```

### v1.0 vs 旧版 Agent 对比

| 特性 | 旧版（0.x） | v1.0 |
|------|------------|------|
| 创建方式 | `create_tool_calling_agent` + `AgentExecutor` | `create_agent`（一步搞定） |
| Prompt | 手动拼，需含 `agent_scratchpad` | 自动生成，只需 `system_prompt` |
| 记忆 | `ConversationBufferMemory`（已弃用） | 内置 LangGraph checkpoint |
| 扩展机制 | `AgentExecutor` 参数 | Middleware 中间件系统 |
| 人机协作 | 需手动实现 | `HumanInTheLoopMiddleware` |
| 结构化输出 | 需额外 Output Parser | `response_format=ToolStrategy()` |
| 持久化 | 无 | 内置 checkpoint |
| 流式 | 需配置 | 内置支持 |

> **旧版迁移**：如果你的代码用了 `create_tool_calling_agent` + `AgentExecutor`，安装 `langchain-classic` 可以继续用，但建议迁移到 `create_agent`。

---

## Callback（回调）

Callback 是**在链执行过程中插入自定义逻辑**的钩子。比如记录日志、显示进度、收集统计信息，都不用修改链本身。

### 工作原理

```
链执行:  prompt → model → parser
              │        │       │
              ▼        ▼       ▼
           onStart  onStart  onStart   ← 回调触发
              │        │       │
              ▼        ▼       ▼
           onEnd    onEnd    onEnd     ← 回调触发
```

### 内置回调：控制台输出

```python
from langchain.globals import set_verbose, set_debug

# verbose: 显示高层流程
set_verbose(True)

# debug: 显示最详细的信息（包括完整提示词）
set_debug(True)

chain.invoke({"topic": "AI"})
# 会打印每一步的详细信息
```

### 自定义回调处理器

```python
from langchain.callbacks import BaseCallbackHandler

class MyCallbackHandler(BaseCallbackHandler):
    def on_llm_start(self, serialized, prompts, **kwargs):
        print(f"[开始] 调用模型: {serialized['name']}")

    def on_llm_end(self, response, **kwargs):
        # 统计 token 使用
        if response.llm_output and "token_usage" in response.llm_output:
            usage = response.llm_output["token_usage"]
            print(f"[结束] 消耗 tokens: {usage}")

    def on_llm_new_token(self, token, **kwargs):
        # 流式输出时，每个 token 都会触发
        print(token, end="", flush=True)

    def on_chain_start(self, serialized, inputs, **kwargs):
        print(f"[链开始] 输入: {str(inputs)[:50]}...")

    def on_chain_end(self, outputs, **kwargs):
        print(f"[链结束] 输出: {str(outputs)[:50]}...")

    def on_tool_start(self, serialized, input_str, **kwargs):
        print(f"[工具开始] {serialized['name']}: {input_str}")

    def on_tool_end(self, output, **kwargs):
        print(f"[工具结束] 结果: {output}")

# 使用
chain.invoke({"topic": "AI"}, config={"callbacks": [MyCallbackHandler()]})
```

### 回调的常用场景

| 场景 | 回调方法 | 用途 |
|------|---------|------|
| 记录日志 | `on_chain_*`, `on_llm_*` | 追踪执行过程 |
| 统计 token | `on_llm_end` | 计算成本 |
| 流式输出 | `on_llm_new_token` | 打字机效果 |
| 错误处理 | `on_chain_error` | 异常告警 |
| 性能监控 | `on_*_start` + `on_*_end` | 计算耗时 |

---

## LangGraph（图式工作流）

LangGraph 是 LangChain 团队推出的**用于构建复杂 Agent 工作流的框架**。它把应用建模为一张「图」——节点是处理步骤，边是步骤之间的流转关系，支持循环、条件分支、状态管理。

### 为什么需要 LangGraph？

普通的 Chain（LCEL 管道）是**单向直线**的，数据从左流到右就结束了。但真实的 Agent 任务往往需要**循环、回退、条件分支**：

```
Chain（直线，不能回头）:  A → B → C → 结束

Agent 需要的（有循环和分支）:
        ┌──────────────────────────┐
        ▼                          │
  开始 → 思考 → 需要工具？─是→ 调用工具 ─┘
                │
                否
                │
                ▼
              输出答案 → 结束
```

LangGraph 就是为了解决这类**有状态、有循环的复杂流程**而设计的。

### LangGraph vs Chain vs Agent

| 特性 | Chain（LCEL） | Agent（create_agent） | LangGraph |
|------|--------------|----------------------|-----------|
| 流程 | 固定直线 | 模型自主决策（黑盒） | **开发者定义图结构** |
| 循环 | 不支持 | 内部有循环（不可控） | **显式支持循环** |
| 状态 | 无 | 隐式 | **显式 State 管理** |
| 可控性 | 高（但不够灵活） | 低（黑盒） | **最高（每一步可控）** |
| 适用场景 | 简单流水线 | 简单工具调用 | **复杂多步工作流** |

### 核心概念

#### 1. State（状态）

State 是在图中流转的**共享数据**，每个节点都可以读取和修改它。通常用 `TypedDict` 或 Pydantic 定义：

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

# 定义状态：整个图共享的数据结构
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # 消息列表，自动追加
    question: str                             # 用户问题
    answer: str                               # 最终答案
```

> `Annotated[list, add_messages]` 表示这个字段用 `add_messages` 函数来更新——新消息会**追加**到列表，而不是覆盖。

#### 2. Node（节点）

节点是图中的**处理单元**，每个节点是一个函数，接收 State、返回 State 的更新：

```python
def think_node(state: AgentState) -> dict:
    """思考节点：调用模型分析问题"""
    response = model.invoke(state["messages"])
    return {"messages": [response]}  # 返回要更新的字段

def answer_node(state: AgentState) -> dict:
    """回答节点：整理最终答案"""
    last_msg = state["messages"][-1].content
    return {"answer": last_msg}
```

#### 3. Edge（边）

边定义节点之间的**流转关系**：

```python
# 普通边：think 之后一定执行 answer
graph.add_edge("think", "answer")

# 条件边：根据状态决定下一步去哪
def should_use_tool(state: AgentState) -> str:
    last_msg = state["messages"][-1]
    if last_msg.tool_calls:
        return "tools"     # 模型要调工具 → 去工具节点
    return "answer"        # 不需要工具 → 直接回答

graph.add_conditional_edges(
    "think",           # 从哪个节点出发
    should_use_tool,   # 路由函数
    {
        "tools": "tool_node",   # 返回 "tools" → 去 tool_node
        "answer": "answer_node" # 返回 "answer" → 去 answer_node
    }
)
```

#### 4. 图的编译与运行

```python
from langgraph.graph import StateGraph, START, END

# 创建图
graph = StateGraph(AgentState)

# 添加节点
graph.add_node("think", think_node)
graph.add_node("tools", tool_node)
graph.add_node("answer", answer_node)

# 添加边
graph.add_edge(START, "think")              # 入口 → think
graph.add_conditional_edges("think", should_use_tool, {
    "tools": "tools",
    "answer": "answer",
})
graph.add_edge("tools", "think")            # 工具执行完 → 回到 think（循环！）
graph.add_edge("answer", END)               # answer → 结束

# 编译成可执行的应用
app = graph.compile()

# 运行
result = app.invoke({
    "messages": [HumanMessage(content="3的平方加5是多少？")],
    "question": "3的平方加5是多少？",
})
print(result["answer"])
```

### 完整图结构示意

```
START → think ──┬── 需要工具? ──是──→ tools ──┐
                │                              │
                │ 否                           │（循环回到 think）
                │                              │
                ▼                              │
              answer → END                    │
                                   ◄──────────┘
```

### LangGraph 内置工具节点

```python
from langgraph.prebuilt import ToolNode, tools_condition

# ToolNode 是现成的工具执行节点
tool_node = ToolNode(tools)

# tools_condition 是现成的路由函数
# 模型有 tool_calls → "tools"
# 没有 → END
graph.add_conditional_edges("think", tools_condition)
```

### Memory / Checkpoint（状态持久化）

LangGraph 自带状态持久化，可以实现「断点续跑」和「人机协作」：

```python
from langgraph.checkpoint.memory import MemorySaver

# 编译时传入 checkpointer
app = graph.compile(checkpointer=MemorySaver())

# 用 thread_id 标识会话，支持多轮对话
config = {"configurable": {"thread_id": "conversation-1"}}

# 第一轮
app.invoke({"messages": [HumanMessage("我叫小明")]}, config=config)

# 第二轮（能记住之前的对话）
app.invoke({"messages": [HumanMessage("我叫什么？")]}, config=config)
# → "你叫小明"
```

### Human-in-the-Loop（人机协作）

LangGraph 可以在关键节点**暂停等待人工确认**：

```python
# 编译时设置在某个节点前中断
app = graph.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["answer"],  # 执行 answer 前暂停
)

# 第一次调用：会停在 answer 之前
result = app.invoke(inputs, config=config)
# 此时可以检查状态，人工审核

# 人工确认后继续执行
result = app.invoke(None, config=config)  # 传 None 表示继续
```

### 预构建的 Graph

LangGraph 提供了开箱即用的预构建图。v1.0 的 `create_agent` 就是基于 LangGraph 构建的预构建图，底层自动实现了 Agent 循环：

```python
from langchain.agents import create_agent

# 一行创建 Agent（底层就是 LangGraph 图，自动实现了 ReAct 循环）
agent = create_agent(
    model="gpt-4o-mini",
    tools=tools,
    system_prompt="你是一个有用的助手",
)

result = agent.invoke({"messages": [{"role": "user", "content": "北京今天天气？"}]})
```

> `create_agent` 是 v1.0 的标准 Agent 构建方式，底层就是 LangGraph。旧版的 `langgraph.prebuilt.create_react_agent` 已移至 `langchain-classic`，新项目直接用 `create_agent` 即可，自带持久化、流式、人机协作等能力。

---

## LangSmith（观测与评估）

LangSmith 是 LangChain 团队的**LLM 应用观测和评估平台**。如果说 LangChain 是「造车」，LangSmith 就是「仪表盘+质检车间」——帮你看到应用内部发生了什么，并评估它做得好不好。

### 为什么需要 LangSmith？

```
没有 LangSmith:
  chain.invoke("问题")
  → 结果不对
  → 为什么不对？哪一步出错了？prompt 有问题还是模型的问题？
  → 一头雾水，只能猜

有 LangSmith:
  chain.invoke("问题")
  → LangSmith 自动记录每一步的输入输出
  → 在网页上看到完整的调用链：prompt → model → parser
  → 一眼看出是 prompt 写得不好，还是模型理解错了
```

### 核心功能

| 功能 | 说明 | 解决的问题 |
|------|------|-----------|
| **Tracing（追踪）** | 记录每次调用的完整链路 | 看不到内部发生了什么 |
| **Evaluation（评估）** | 自动评估应用质量 | 不知道答案好不好 |
| **Datasets（数据集）** | 管理测试用例 | 没有标准测试集 |
| **Playground（测试场）** | 在线调试 prompt | 改 prompt 要反复跑代码 |
| **Monitoring（监控）** | 生产环境监控 | 线上出问题不知道 |

### 快速接入

#### 1. 安装与配置

```bash
pip install langsmith
```

```python
# 在 .env 文件中配置（或设置环境变量）
# LANGSMITH_API_KEY=你的key
# LANGSMITH_TRACING=true
# LANGSMITH_PROJECT=我的项目

import os
from dotenv import load_dotenv
load_dotenv()

# 只要设置了以上环境变量，LangChain 的所有调用会自动被追踪
# 不需要改任何业务代码！
```

#### 2. 自动追踪

配置好环境变量后，**所有 LangChain 调用自动被追踪**，无需改代码：

```python
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.output_parsers import StrOutputParser

model = ChatOpenAI(model="gpt-4o-mini")
prompt = ChatPromptTemplate.from_template("解释什么是{topic}")
chain = prompt | model | StrOutputParser()

# 这次调用会自动出现在 LangSmith 的追踪面板中
result = chain.invoke({"topic": "量子纠缠"})
```

在 LangSmith 网页上你会看到：

```
Chain.invoke({"topic": "量子纠缠"})
├── ChatPromptTemplate: "解释什么是量子纠缠"     ← 格式化后的 prompt
├── ChatOpenAI: 调用 GPT-4o-mini                  ← 模型调用
│   ├── 输入 tokens: 12
│   ├── 输出 tokens: 85
│   └── 耗时: 1.2s
└── StrOutputParser: "量子纠缠是..."               ← 解析后的输出
```

#### 3. 手动追踪（更细粒度）

```python
from langsmith import traceable

# 用 @traceable 装饰任意函数，也能被追踪
@traceable(name="我的自定义函数")
def my_function(question: str) -> str:
    # 这里面的所有操作都会被记录
    result = process(question)
    return result
```

### 评估（Evaluation）

评估是 LangSmith 的核心价值：用一套测试数据自动检验你的应用质量。

#### 1. 创建数据集

```python
from langsmith import Client

client = Client()

# 创建数据集
dataset = client.create_dataset("RAG问答测试集", description="测试文档问答质量")

# 添加测试用例（输入 + 期望输出）
client.create_example(
    inputs={"question": "Python是谁创建的？"},
    outputs={"answer": "Guido van Rossum"},
    dataset_id=dataset.id,
)

client.create_example(
    inputs={"question": "LCEL用什么符号串联组件？"},
    outputs={"answer": "管道符 |"},
    dataset_id=dataset.id,
)
```

#### 2. 运行评估

```python
from langsmith.evaluation import evaluate

# 定义评估函数：判断模型回答是否正确
def correctness_evaluator(run, example) -> dict:
    """检查答案是否包含期望的关键词"""
    expected = example.outputs["answer"]
    actual = run.outputs["output"]

    # 用另一个 LLM 来判断答案是否正确（LLM-as-judge）
    is_correct = expected.lower() in actual.lower()

    return {
        "key": "correctness",
        "score": 1.0 if is_correct else 0.0,
        "comment": f"期望: {expected}, 实际: {actual}",
    }

# 运行评估
results = evaluate(
    lambda inputs: rag_chain.invoke(inputs["question"]),  # 被测函数
    data="RAG问答测试集",           # 数据集名称
    evaluators=[correctness_evaluator],  # 评估器
    experiment_name="RAG-v1",      # 实验名称
)
# 结果会显示在 LangSmith 网页上，每个用例的得分一目了然
```

#### 3. 评估指标

| 评估器类型 | 说明 | 示例 |
|-----------|------|------|
| 关键词匹配 | 答案是否包含关键词 | 期望"Guido"，实际包含→✓ |
| LLM-as-judge | 用另一个 LLM 评判 | "这个回答准确吗？打 1-5 分" |
| 相似度 | 语义相似度 | cos similarity > 0.8 |
| 自定义 | 你写的任何逻辑 | 正则匹配、长度检查等 |

### Playground（在线调试）

在 LangSmith 网页上，你可以：

```
1. 选中某次追踪记录
2. 点击 "Open in Playground"
3. 在网页上直接修改 prompt / 模型参数
4. 实时看到修改后的输出
5. 对比修改前后的差异
```

> 不用反复跑代码改 prompt，在网页上直接调试，效率大幅提升。

### 监控生产环境

```python
# 生产环境自动收集以下指标:
# - 每次调用的延迟
# - token 使用量和成本
# - 错误率
# - 用户反馈（thumbs up/down）

# 配合 LangSmith 的 Dashboard 实时监控
# 设置告警：延迟超过 5s 或错误率超过 10% 时通知
```

### LangSmith 与 Callback 的关系

```
Callback:   代码层面的钩子，自己写逻辑处理事件
LangSmith:  平台级方案，自动收集+可视化+评估，不用写代码

二者可以配合: LangSmith 底层也用 Callback 机制收集数据
```

| 对比 | Callback | LangSmith |
|------|---------|-----------|
| 实现方式 | 自己写代码 | 配置即可，零代码 |
| 数据存储 | 自己处理 | 平台自动存储 |
| 可视化 | 自己做 | 自带 Web 界面 |
| 评估能力 | 无 | 内置评估系统 |
| 适用阶段 | 简单日志 | 开发+测试+生产全覆盖 |

---

## 实战案例

### 案例 1：智能文档问答机器人（RAG 完整版）

把一篇文档喂给模型，然后自由提问：

```python
from langchain_chroma import Chroma
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain.prompts import ChatPromptTemplate
from langchain.runnables import RunnablePassthrough
from langchain.output_parsers import StrOutputParser

# ===== 准备知识库 =====
knowledge = """
LangChain 是一个用于开发大语言模型（LLM）应用的开源框架，由 Harrison Chase 于 2022 年 10 月创建。
它使用 Python 和 JavaScript 两种语言编写。
LangChain 的核心组件包括 Model、Prompt、Chain、Memory、Agent 和 Tool。
Model 是与 LLM 交互的统一接口，支持 OpenAI、Anthropic 等多种模型。
Prompt 是提示词模板，用于结构化地组织模型输入。
Chain 把多个组件串联成一个处理流程。
Memory 用于保存对话历史，让模型记住上下文。
Agent 是能自主决策的智能体，可以调用工具完成任务。
Tool 是让模型执行实际操作的接口，如搜索、计算、查数据库。
LCEL 是 LangChain 的表达式语言，用管道符 | 串联组件。
RAG 是检索增强生成，先检索相关文档再让模型回答。
Embedding 把文本转换成向量，用于语义搜索。
Vector Store 是存储和检索向量的数据库，如 Chroma、FAISS。
"""

# 切片
splitter = RecursiveCharacterTextSplitter(chunk_size=150, chunk_overlap=30)
chunks = splitter.create_documents([knowledge])

# 存入向量数据库
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(chunks, embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# ===== 构建 RAG 链 =====
model = ChatOpenAI(model="gpt-4o-mini", temperature=0)

prompt = ChatPromptTemplate.from_template("""
你是 LangChain 知识助手。根据以下上下文回答问题。
如果上下文中没有答案，请说"文档中没有相关信息"。

上下文:
{context}

问题: {question}
""")

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)

# ===== 提问 =====
questions = [
    "LangChain 是谁创建的？",
    "Agent 和 Chain 有什么区别？",
    "LCEL 是什么？",
    "Java 有什么特性？",  # 文档中没有
]

for q in questions:
    print(f"问: {q}")
    print(f"答: {rag_chain.invoke(q)}\n")
```

输出：

```
问: LangChain 是谁创建的？
答: LangChain 由 Harrison Chase 于 2022 年 10 月创建。

问: Agent 和 Chain 有什么区别？
答: Chain 把多个组件串联成固定的处理流程，而 Agent 是能自主决策的智能体，可以调用工具完成任务。

问: LCEL 是什么？
答: LCEL 是 LangChain 的表达式语言，用管道符 | 串联组件。

问: Java 有什么特性？
答: 文档中没有相关信息。
```

### 案例 2：多工具 Agent（计算 + 搜索）

```python
from langchain.agents import create_agent
from langchain.tools import tool
import datetime

# 自定义工具
@tool
def calculate(expression: str) -> str:
    """计算数学表达式，如 '3 * 4 + 5'。只支持加减乘除和括号。"""
    try:
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"计算失败: {e}"

@tool
def get_current_time() -> str:
    """获取当前日期和时间"""
    return datetime.datetime.now().strftime("%Y年%m月%d日 %H:%M:%S")

@tool
def string_reverse(text: str) -> str:
    """反转字符串"""
    return text[::-1]

# 一行创建 Agent（v1.0 方式，不需要手动拼 prompt）
agent = create_agent(
    model="gpt-4o-mini",
    tools=[calculate, get_current_time, string_reverse],
    system_prompt="你是一个多功能助手，可以使用工具帮助用户。",
)

# Agent 会自己决定用哪个工具
r1 = agent.invoke({"messages": [{"role": "user", "content": "现在是几点？"}]})
print(r1["messages"][-1].content)

r2 = agent.invoke({"messages": [{"role": "user", "content": "帮我算一下 (15 + 27) * 3"}]})
print(r2["messages"][-1].content)

r3 = agent.invoke({"messages": [{"role": "user", "content": "把 'Hello World' 反转"}]})
print(r3["messages"][-1].content)
```

### 案例 3：流式对话机器人

```python
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.output_parsers import StrOutputParser
from langchain.messages import SystemMessage, HumanMessage, AIMessage

model = ChatOpenAI(model="gpt-4o-mini")
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个幽默的程序员朋友，回答简洁有趣"),
    ("placeholder", "{history}"),
    ("human", "{input}"),
])

history = []

def chat(user_input: str):
    chain = prompt | model | StrOutputParser()

    print("AI: ", end="")
    full_response = ""
    for chunk in chain.stream({"history": history, "input": user_input}):
        print(chunk, end="", flush=True)
        full_response += chunk
    print()  # 换行

    # 更新历史
    history.append(HumanMessage(content=user_input))
    history.append(AIMessage(content=full_response))

# 交互
chat("Python 和 Java 哪个更好？")
chat("那如果我刚入门呢？")
```

---

## 最佳实践

### 1. 模型选择

```python
# 简单任务用便宜模型，复杂推理用强模型
fast_model = ChatOpenAI(model="gpt-4o-mini")     # 便宜快速
smart_model = ChatOpenAI(model="gpt-4o")          # 强但贵

# 路由策略：简单问题用 fast，复杂问题用 smart
```

### 2. Prompt 工程

```python
# ✅ 好的 prompt：角色明确 + 任务清晰 + 格式要求
good_prompt = ChatPromptTemplate.from_template("""
你是一个资深 {language} 开发工程师。
请审查以下代码，按 JSON 格式返回:
{{
  "issues": ["问题1", "问题2"],
  "score": 1-10的评分,
  "suggestion": "改进建议"
}}

代码:
{code}
""")

# ❌ 差的 prompt：模糊、无约束
bad_prompt = ChatPromptTemplate.from_template("看看这段代码: {code}")
```

### 3. 控制 token 成本

```python
from langchain.messages import trim_messages, SystemMessage, HumanMessage, AIMessage
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini", max_tokens=500)

# v1.0 方式：用 trim_messages 裁剪历史消息，控制 token 用量
history = [
    SystemMessage(content="你是一个助手"),
    HumanMessage(content="你好"),
    AIMessage(content="你好！有什么可以帮你的？"),
    # ... 更多历史消息
]

# 只保留最近的消息，按 token 数裁剪
trimmed = trim_messages(
    history,
    max_tokens=1000,             # 最多保留 1000 token
    strategy="last",             # 保留最新的消息
    token_counter=model,         # 用模型来计算 token 数
    include_system=True,         # 始终保留 SystemMessage
    start_on="human",            # 确保从 HumanMessage 开始
)
```

### 4. 错误处理

```python
from langchain.agents import create_agent
from langchain.agents.middleware import AgentMiddleware

# v1.0 方式：用 create_agent + 自定义 Middleware 处理错误
class ErrorHandlingMiddleware(AgentMiddleware):
    def wrap_tool_call(self, request, handler):
        try:
            return handler(request)
        except Exception as e:
            # 工具出错时返回错误信息，而不是崩溃
            return f"工具执行失败: {e}，请重试或换一种方式"

agent = create_agent(
    model="gpt-4o-mini",
    tools=tools,
    middleware=[ErrorHandlingMiddleware()],
)

# create_agent 底层基于 LangGraph，自带循环次数控制
# 也可以通过 middleware 的 before_model 钩子手动限制迭代次数
```

### 5. RAG 优化

```python
# 切片大小要合适：太小丢语义，太大浪费 token
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,        # 经验值 200-1000
    chunk_overlap=50,      # 10-20% 的 overlap
)

# 用 MMR 检索兼顾相关性和多样性
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 4, "fetch_k": 20},
)
```

### 6. 优先使用 LCEL

```python
# ✅ 推荐：用 LCEL 构建链
chain = prompt | model | parser

# ❌ 不推荐：用旧的 Chain 类
# chain = LLMChain(llm=model, prompt=prompt)
```

### 7. 环境变量管理

```python
import os
# 不要把 API key 写在代码里
# 用 .env 文件 + python-dotenv
from dotenv import load_dotenv
load_dotenv()  # 自动加载 .env 中的变量

# .env 文件内容:
# OPENAI_API_KEY=sk-xxx
# 其他配置...
```

---

## 总结

### 核心概念速查表

| 概念 | 一句话解释 | 类比 |
|------|-----------|------|
| **Model** | 与 LLM 交互的统一入口（`init_chat_model` 统一初始化） | 翻译官 |
| **Prompt** | 结构化的输入模板 | 填空题模板 |
| **Output Parser** | 把模型输出解析成结构化数据 | 阅卷老师 |
| **LCEL** | 用 `\|` 串联组件的语法 | 水管连接 |
| **Chain** | 用 LCEL 管道符组合多步骤（旧 Chain 类在 `langchain-classic`） | 流水线 |
| **Memory** | 手动管理消息列表 + `trim_messages` + LangGraph checkpoint | 聊天记录本 |
| **Embedding** | 把文本变成向量 | 文字翻译成坐标 |
| **Vector Store** | 存储和检索向量 | 语义图书馆 |
| **Retriever** | 检索相关文档的接口 | 图书管理员 |
| **RAG** | 检索+生成，让模型用你的数据 | 开卷考试 |
| **Tool** | 让模型执行实际操作（`@tool` 装饰器） | 手和脚 |
| **Agent** | 用 `create_agent` 构建的自主智能体 | 独立思考的助手 |
| **Middleware** | Agent 执行过程中的可组合拦截器（v1.0 新特性） | 流水线质检员 |
| **Callback** | 执行过程中的钩子 | 监工 |
| **LangGraph** | 用图构建有状态、可循环的工作流（`create_agent` 底层） | 流程图引擎 |
| **LangSmith** | 追踪、评估、监控 LLM 应用 | 仪表盘+质检车间 |

### 组件关系图

```
                    ┌─────────┐
                    │  Prompt │ 提示词模板
                    └────┬────┘
                         │
┌────────────────┐  ┌────▼──────────────┐    ┌──────────────┐
│ 消息历史管理    │─▶│       Model       │───▶│ Output Parser│
│ trim_messages  │  │   init_chat_model │    │   输出解析   │
│ LangGraph ckpt │  └────────┬──────────┘    └──────┬───────┘
└────────────────┘           │                       │
                             └───────┬───────────────┘
                                     │  用 LCEL 的 | 串联
                                     ▼
                               ┌───────────┐
                               │   Chain   │ 链（LCEL 管道）
                               └─────┬─────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  │                  │                  │
           ┌──────▼──────┐   ┌───────▼───────┐  ┌──────▼──────┐
           │    RAG      │   │     Tool      │  │ Middleware  │
           │ 检索增强    │   │   @tool 装饰器 │  │  拦截器/守卫 │
           └──────┬──────┘   └───────┬───────┘  └─────────────┘
                  │                  │
         ┌────────┴────────┐         │
         │                 │         │
  ┌──────▼─────┐   ┌──────▼──────┐  │
  │ Embedding  │   │ Vector Store│  │
  │  词嵌入    │   │  向量存储   │  │
  └────────────┘   └─────────────┘  │
                                    │
                             ┌──────▼──────┐
                             │   Agent     │ create_agent()
                             │   智能体    │ 自主决策+工具调用
                             └──────┬──────┘
                                    │
                        ┌───────────┴───────────┐
                        │                       │
                 ┌──────▼──────┐         ┌──────▼──────┐
                 │  LangGraph  │         │  LangSmith  │
                 │  图式工作流 │         │  观测与评估 │
                 │ (create_agent│         │ (追踪/测试/│
                 │  的底层引擎)│         │   监控)     │
                 └─────────────┘         └─────────────┘
                 构建层                    观测层
```

### 学习路径建议

```
入门:  Model（init_chat_model）→ Prompt → Output Parser → LCEL
       （掌握基础组件和管道语法）

进阶:  Chain（LCEL 管道）→ Memory（手动管理消息）→ Retriever → RAG
       （构建实用应用：对话机器人、文档问答）

高级:  Tool（@tool 装饰器）→ Agent（create_agent）→ Middleware → Callback
       （自主智能体、中间件拦截、可观测性、生产级应用）

专家:  LangGraph → LangSmith
       （复杂图式工作流 + 全生命周期观测评估）
```

### 环境安装

```bash
# 基础安装（v1.0+，langchain-core 会自动作为依赖安装）
pip install -U langchain

# 按需安装集成包（不用全装）
pip install langchain-openai          # OpenAI 集成
pip install langchain-anthropic       # Anthropic 集成
pip install langchain-google-genai    # Google Gemini 集成
pip install langchain-ollama          # Ollama 本地模型集成
pip install langchain-chroma          # Chroma 向量数据库
pip install langchain-community       # 社区集成（各种工具）
pip install langchain-text-splitters  # 文本切片

# 旧版 Chain/Memory 类（如果需要兼容旧代码）
pip install langchain-classic

# LangGraph（图式工作流，create_agent 的底层引擎）
pip install langgraph

# LangSmith（观测与评估）
pip install langsmith

# 或一步到位安装所有常用包
pip install langchain[all]
```

> **v1.0 包结构说明**：
> - `langchain`：核心包，包含 `create_agent`、`init_chat_model`、Middleware 等
> - `langchain-core`：底层基础（自动安装），提供 Runnable 接口等
> - `langchain-classic`：旧版组件（`LLMChain`、`ConversationBufferMemory`、`AgentExecutor` 等）
> - `langchain-xxx`：各厂商/工具的集成包，按需安装
