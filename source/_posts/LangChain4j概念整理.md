---
title: LangChain4j概念整理
date: 2026-03-03 16:52:35
tags:
categories: AI应用开发
---

# 1. 介绍下Langchain

## 1. 定义

LangChain 是一个**面向大语言模型（LLM）的应用开发框架**，用来把 LLM 和各种外部数据源、工具、流程串起来，快速构建复杂的 AI 应用，而不仅是“调一次 API 问个问题”。

------

## 2. 解决的核心痛点

直接用 LLM API 会遇到这些问题：

- **上下文有限**：大模型记不住多轮对话、长文档内容。
- **缺乏实时信息**：不懂最新数据、私有数据。
- **只会“聊天”，不会“做事”**：不能查数据库、调接口、算结果。
- **多步任务难编排**：需要多轮推理、多工具协作时，自己写控制流很麻烦。

LangChain 就是把这些“胶水逻辑”封装成易用的模块，让开发者专注业务，而不是底层拼装。

------

## 3. 核心概念（面试高频）

1. **Models（模型）**

   抽象了对 LLM 的调用，不只 OpenAI，还支持本地模型、Azure OpenAI、Anthropic 等。

   你只需要换配置，不用改业务逻辑。

2. **PromptTemplate（提示词模板）**

   把提示词做成模板，动态填充变量，方便复用、统一管理提示词版本。

3. **Chains（链）**

   把多个步骤串联成一个流程，比如：

   “检索文档 → 组装 Prompt → 调用 LLM → 解析结果”。

   常见链：`LLMChain`、`SequentialChain`、`RouterChain`。

4. **Retrievers / Vector Stores（检索器 / 向量数据库）**

   用于 RAG（检索增强生成）：先把文档向量化存入 Milvus、Pinecone、FAISS 等，再根据用户问题检索相关内容，和问题一起送给 LLM，提升准确率。

5. **Tools & Agents（工具与代理）**

   - Tools：封装外部能力，如查天气、查数据库、调用内部接口。
   - Agents：让 LLM 自己“决定”什么时候用什么工具，比如 ReAct Agent：思考 → 选工具 → 执行 → 再思考。

> 比如做一个“文档问答机器人”：
>
> 1. 用 LangChain 的 `TextLoader`加载文档，`RecursiveCharacterTextSplitter`切分成块；
>
> 2. 用 `OpenAIEmbeddings`转成向量，存到 `FAISS`；
>
> 3. 用户提问时，用 `Retriever`查出相关片段；
>
> 4. 用 `PromptTemplate`组装成“上下文 + 问题”；
>
> 5. 交给 LLM 生成答案并返回。
>
>    整个过程在 LangChain 里就是用几条链式调用就能搭出来。

## 5. 典型应用场景

- **RAG 问答系统**：企业知识库、文档助手、客服机器人。
- **Agent 自动化**：自动查资料、写报告、调用内部系统的智能助手。
- **多模态 / 多步骤工作流**：比如“分析财报 → 查行业数据 → 生成投资建议”。