# #LangChain多数据源集成   
下面我给你一份 **LangChain 多数据源集成（Multi-Source RAG / Agent）完整工程方案**，从架构 → 模块拆解 → 多源检索 → 多源合并 → 最佳实践，一套讲透，让你能在企业级项目里真正落地。  
内容直说重点，不玩虚的。  
  
## 🚀** 一、你要实现的能力是什么？**  
用户问一句：  
“给我总结下我们 2024 销售报告的关键趋势，并结合行业公开数据分析原因。”  
系统要能自动从多个数据源取资料：  
* 📄 企业文档（PDF/Word/Markdown）  
* 📊 数据库（SQL）  
* 🌐 网页（API/爬虫）  
* 🧠 内置知识库（知识点/规则）  
* 🔎 传统搜索（Bing/Google）  
* 🏛️ 第三方结构化数据（如 Wikipedia / Finance API）  
并自动：  
1. 检索 → 过滤  
2. 排序 → rerank  
3. 压缩 → 拼接上下文  
4. 生成分析  
这就是 LangChain 的 ****多数据源整合能力（Multi-Source RAG / Agent）****。  
  
## 🧱** 二、整体架构（关键要理解）**  
```
用户 query
       ↓
Query Router（路由）
       ↓
每个数据源对应一个 Retriever / Tool
       ↓
合并所有结果（聚合）
       ↓
Rerank + Compress（精排+压缩）
       ↓
上下文 Orchestrator（上下文工程）
       ↓
LLM 生成最终回答

```
核心理念：  
****数据源要拆分，检索要分别做，上下文要统一 orchestrate。****  
****数据源要拆分，检索要分别做，上下文要统一 orchestrate。****  
  
## 🧩** 三、代码结构（最佳模板）**  
```
multi_source_rag/
 ├── retrievers/
 │    ├── vector_retriever.py   # 文档向量库
 │    ├── sql_retriever.py      # SQL 数据库
 │    ├── web_retriever.py      # 网络数据/API
 │    └── wiki_retriever.py     # Wikipedia
 ├── router/
 │    └── query_router.py       # Query Router
 ├── aggregator/
 │    └── aggregator.py         # 合并多个检索结果
 ├── orchestrator/
 │    └── context_manager.py    # 上下文工程
 ├── main.py                    # 主调用入口
 └── config.py

```
这是企业级结构。  
  
## 🔍** 四、构建各个数据源 Retriever（核心）**  
## **1. 文档向量库 Retriever**  
```
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings

def build_vector_retriever():
    embed = OpenAIEmbeddings(model="text-embedding-3-large")
    store = Chroma(persist_directory="./db", embedding_function=embed)
    return store.as_retriever(search_kwargs={"k": 5})

```
  
## **2. SQL 数据库 Retriever**  
（查询结果返回文本 Document）  
```
from langchain import SQLDatabase
from langchain.chains import SQLDatabaseChain

def build_sql_retriever(llm):
    db = SQLDatabase.from_uri("sqlite:///sales.db")
    chain = SQLDatabaseChain(llm=llm, database=db)

    class SQLRetriever:
        def get_relevant_documents(self, query):
            answer = chain.run(query)
            return [Document(page_content=answer, metadata={"source": "sql"})]
    return SQLRetriever()

```
  
## **3. 网络/API Retriever（实时数据）**  
```
import requests

class WebRetriever:
    def get_relevant_documents(self, query):
        response = requests.get(
            "https://api.duckduckgo.com/",
            params={"q": query, "format": "json"}
        ).json()
        text = response.get("Abstract", "")
        return [Document(page_content=text, metadata={"source": "web"})]

```
  
## **4. Wikipedia Retriever**  
```
from langchain.document_loaders import WikipediaLoader

class WikiRetriever:
    def get_relevant_documents(self, query):
        docs = WikipediaLoader(query=query, load_max_docs=2).load()
        return docs

```
  
## 🔀** 五、Query Router（决定用哪个数据源）**  
LangChain 内置 RouterChain。  
例子：  
```
from langchain.chains.router import MultiPromptChain, RouterChain
from langchain.prompts import PromptTemplate
from langchain_openai import ChatOpenAI

def build_router(llm):
    destinations = {
        "docs": "与内部文档、公司政策、PDF 相关的内容",
        "sql": "与销售、财务、数量统计、数据分析相关的问题",
        "web": "涉及实时数据、外部信息、新闻的查询",
        "wiki": "需要百科知识、背景资料的查询"
    }

    router_prompt = """
用户问题: {input}

目标:
判断用户的意图属于以下哪个数据源：
{destinations}

只返回一个 key。
"""

    router_chain = LLMChain(
        llm=llm,
        prompt=PromptTemplate(
            input_variables=["input", "destinations"],
            template=router_prompt
        )
    )
    return router_chain

```
  
## 🔗** 六、聚合多个数据源（可并行）**  
```
def aggregate_results(results_list):
    all_docs = []
    for docs in results_list:
        all_docs.extend(docs)
    return all_docs

```
如果你希望并行：  
```
import asyncio

async def multi_query(query, retrievers):
    tasks = [r.get_relevant_documents(query) for r in retrievers]
    results = await asyncio.gather(*tasks)
    return aggregate_results(results)

```
  
## 🎛** 七、上下文工程（rerank + 压缩）**  
```
from langchain.retrievers.context_compressor import ContextualCompressionRetriever
from langchain_community.document_transformers import BGERerank

def compress(docs, question):
    reranker = BGERerank(top_n=3)
    compressor = ContextualCompressionRetriever(
        base_compressor=reranker,
        base_retriever=None
    )
    return compressor.compress_documents(docs, query=question)

```
  
## 🧠** 八、最终大模型生成回答（DeepSeek/Qwen/OpenAI）**  
```
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="deepseek-chat", temperature=0)

def final_answer(question, docs):
    context = "\n\n".join([d.page_content for d in docs])

    prompt = f"""
你是一名专业知识融合 AI。

根据下面来自多个数据源的内容回答用户问题。

用户问题：
{question}

相关内容：
{context}

请给出：
1. 明确结论  
2. 各数据源的依据  
3. 推理过程  
4. 最终建议（如适用）
"""

    return llm.invoke(prompt).content

```
  
## 🔥** 九、多数据源集成主函数**  
```
def rag_multi_source(question):
    retrievers = [
        vector_retriever,
        sql_retriever,
        web_retriever,
        wiki_retriever,
    ]

    # 多源检索
    results = []
    for r in retrievers:
        docs = r.get_relevant_documents(question)
        results.append(docs)

    # 合并
    all_docs = aggregate_results(results)

    # 精排与压缩
    selected_docs = compress(all_docs, question)

    # 生成最终回答
    return final_answer(question, selected_docs)

```
调用：  
```
print(rag_multi_source("我们 2024 年销售下降的原因？结合行业数据分析"))

```
  
## 🏆** 十、企业级建议（非常关键）**  
1. **多源检索必须精排，不然上下文爆炸**  
2. **SQL 必须做安全过滤（禁止 DROP/DELETE）**  
3. **网络检索不要直接信任，必须 rerank**  
4. **Router 精准度很重要，建议用 Qwen2.5/DeepSeek R1 来路由**  
5. **不同数据源 metadata 必须保留（用于溯源）**  
6. **强制压缩上下文，避免 token 过长**  
7. **一定要做缓存（同样 query 不要重复查）**  
  
## 📌** 最终总结一句话**  
****多数据源 RAG = 多检索器 + Query Router + 聚合 + 精排压缩 + LLM Orchestrator。 真正的难点不是检索，是“如何整合、过滤、压缩并调度上下文”。****  
  
如果你愿意，我可以再给你：  
## ✔** 完整项目脚手架（含目录 + 所有代码）**  
## ✔** DeepSeek + Multi-Source RAG 最优架构图**  
## ✔** 企业级多数据源中台（RAG 3.0）设计文档**  
## ✔** Streamlit / FastAPI 可视化版本**  
你想要哪一个？  
