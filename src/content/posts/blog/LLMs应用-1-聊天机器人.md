---
title: LLMs-应用1-聊天机器人
published: 2025-10-28
tags:
  - LLMs
  - AI
lang: zh
abbrlink: llms-usage-chatbot
---
### 1. LLMs 实际运用案例

> 大模型实现聊天机器人。

[Python 实现聊天机器人](https://graphacademy.neo4j.com/courses/llm-chatbot-python/3-tools/1-vector-tool/)



<iframe src="https://excalidraw.com/#json=POKGHBJTn95znC4HEGZWV,2_zeJoJB17oF-yNMW_8OWw" width="100%" height="600px" frameborder="0" ></iframe

### 2. LLMs 选择

模型分类：

```py
from langchain_ollama import OllamaLLM
from langchain_ollama import OllamaEmbeddings

# Create the LLM
llm = OllamaLLM(
    model="deepseek-r1:8b", 
    base_url="http://localhost:11434",
    system= "你是一个遵循 ReAct 格式的智能助手，请严格按照 Thought/Action/Action Input 或 Final Answer 的格式输出。"
    )


# Create the Embedding model
# 目的是将 "xxxxx" 字符 转变为 向量表示 （x1,y2,.....） 通常维度 1-768 、1536
embedding_model = OllamaEmbeddings(model="embeddinggemma:300m", base_url="http://localhost:11434")
```


| 隐私                             | 能力支持                | 限制                                              |
| -------------------------------- | ----------------------- | ------------------------------------------------- |
| 本地模型，例如Ollama             | 是否支持ReAct、电脑性能、可视化模型 | abliterated模型和普通模型（非法问题提醒不予回答） |
| 大模型厂商，例如OpenAI、DeepSeek | token支付，月租选择     | 国内IP封禁，信用卡支付，注册难（hugging face）    |


### 3. Agent 模板

> 具体模板 建议查看官网提示、GitHub、AI 提示生成（Copilot等）

```py
from langchain_core.prompts import PromptTemplate

# Define the agent prompt
agent_prompt = PromptTemplate.from_template("""
你是一个智能助手，擅长使用工具解决问题。

你可以使用以下工具：
{tool_names}

工具详细信息如下：
{tools}

请根据用户的问题进行推理，并决定是否需要使用工具。如果需要，请严格按照以下格式输出：

Question: 用户的问题
Thought: 你对问题的思考
Action: 工具名称
Action Input: 工具的输入内容

或者：

Thought: 你对问题的思考
Final Answer: 最终答案

以下是几个示例：

Question: 你最喜欢的电影是什么？
Thought: 这是一个关于电影偏好的问题，我可以使用 General Chat 工具来回答。
Action: General Chat
Action Input: "你最喜欢的电影是什么？"

Question: 谁是《盗梦空间》的导演？
Thought: 这是一个关于电影信息的问题，我可以直接回答。
Final Answer: 《盗梦空间》的导演是克里斯托弗·诺兰。

现在请回答下面的问题：

Question: {input}
{agent_scratchpad}
""")
```

### 4. TOOLS 

:::fold[get_movie_plot 工具实现]
```py
# 这里不需要理解具体实现，只关注get_movie_plot如何转变为工具即可，如需了解可以查看Neo4j官网的向量索引、RAG基础教程部分。
# 提供向量搜索， neo4j中数据存在一个向量索引绑定着维度属性[],
# embedding模型将用户的搜索、提问转换为维度，retrieval检索利用相似算法找到最相似的数据集；
# 再将数据集转换为文本，通过模板生成回复。

"""
 {
    name: "hello",
    description: "Says hello to the user",
    embeddings:[x1,y2,z3,....]

 }

query = "你好" ===> embedding model===> [x1,y2,z3,....] ===> retrieve ==compares with===> [x1,y2,z3,....]  ===> {name: "hello"}

"""

import streamlit as st
from llm import llm, embeddings
from graph import graph

# tag::import_vector[]
from langchain_neo4j import Neo4jVector
# end::import_vector[]
# tag::import_chain[]
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain.chains import create_retrieval_chain
# end::import_chain[]

# tag::import_chat_prompt[]
from langchain_core.prompts import ChatPromptTemplate
# end::import_chat_prompt[]


# tag::vector[]
neo4jvector = Neo4jVector.from_existing_index(
    embeddings,                              # <1>
    graph=graph,                             # <2>
    index_name="moviePlots",                 # <3>
    node_label="Movie",                      # <4>
    text_node_property="plot",               # <5>
    embedding_node_property="plotEmbedding", # <6>
    retrieval_query="""
RETURN
    node.plot AS text,
    score,
    {
        title: node.title,
        directors: [ (person)-[:DIRECTED]->(node) | person.name ],
        actors: [ (person)-[r:ACTED_IN]->(node) | [person.name, r.role] ],
        tmdbId: node.tmdbId,
        source: 'https://www.themoviedb.org/movie/'+ node.tmdbId
    } AS metadata
"""
)
# end::vector[]

# tag::retriever[]
retriever = neo4jvector.as_retriever()
# end::retriever[]

# tag::prompt[]
instructions = (
    "Use the given context to answer the question."
    "If you don't know the answer, say you don't know."
    "Context: {context}"
)

prompt = ChatPromptTemplate.from_messages(
    [
        ("system", instructions),
        ("human", "{input}"),
    ]
)
# end::prompt[]

# tag::chain[]
question_answer_chain = create_stuff_documents_chain(llm, prompt)
plot_retriever = create_retrieval_chain(
    retriever, 
    question_answer_chain
)
# end::chain[]

# tag::get_movie_plot[]
def get_movie_plot(input):
    return plot_retriever.invoke({"input": input})
# end::get_movie_plot[]

```

**相关性算法**：
[向量数据库](https://guangzhengli.com/blog/zh/vector-database#%E7%9B%B8%E4%BC%BC%E6%80%A7%E6%B5%8B%E9%87%8F-similarity-measurement)

> 理解可以基于二维平面理解，归纳法拓展到n维空间。<br>
- 基于两点间距离（欧几里得距离）： n 维坐标系中，两点的距离  （Neo4j可选择）
- 基于余弦相似度：n 维向量空间中，两个向量的夹角的余弦值     （Neo4j可选择）
- 基于点积

:::

```py
# Create a set of tools
tools = [
        Tool.from_function(
        name="General Chat",
        description="For general movie chat not covered by other tools",
        func=movie_chat.invoke,
        ),
        # Add more tools here...
        Tool.from_function(
        name="Movie Plot Search",  
        description="For when you need to find information about movies based on a plot",
        func=get_movie_plot, 
        )
]
# Create chat history callback
def get_memory(session_id):
    return 
# Create the agent
agent_prompt = PromptTemplate.from_template()

agent = create_react_agent(llm, tools, agent_prompt)
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    handle_parsing_errors=True,
    )

chat_agent = RunnableWithMessageHistory(
    agent_executor,
    get_memory,
    input_messages_key="input",
    history_messages_key="chat_history",
)

# Create a handler to call the agent
def generate_response(user_input):
    """
    Create a handler that calls the Conversational agent
    and returns a response to be rendered in the UI
    """

    response = chat_agent.invoke(
        {"input": user_input},
        {"configurable": {"session_id": get_session_id()}},)

    return response['output']
```

### 5. 问题

* 1. 模型切换
> Neo4j 教程提供的是 OpenAI的模型，但是OpenAI免费额度取消，需要国外账户或者信用卡充值；<br>
> 切换Ollama 本地模型，
**注意点：** 
- vpn 代理可能导致模型连接失败；
- 教程的requirement没有安装langchain_ollama;
- 向量搜索错误，维度不匹配，embedding模型和向量索引维度不匹配，手动为沙盒neo4j添加768维度的向量索引；

* 2. ReAct 格式
> agent 提示模板错误，可能导致模型无法理解问题，报 `Invalid Format: Missing 'Action:' after 'Thought:'` 错误；<br>
> 模型的选择，查看是否支持 Tools 功能，是否支持 ReAct 格式；<br>
**解决方案：** 
- 调整 agent 提示模板，添加 `Action:` 字段,不同模型理解不同格式，需要调整模板；
- PromptTemplate.from_template中 {tool_names} 、{tools}都是必须参数，否则会报错；

* 3. 格式错误
> 输出内容不符和解析器要求；
> [OUTPUT_PARSING_FAILURE](https://js.langchain.com/docs/troubleshooting/errors/OUTPUT_PARSING_FAILURE/)
- handle_parsing_errors=True, 处理解析错误；
- 控制格式规范