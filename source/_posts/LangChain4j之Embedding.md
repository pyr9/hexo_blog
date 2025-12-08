---
title: LangChain4j之Embedding
date: 2025-07-23 19:34:13
tags:
categories: AI应用开发
---

# 1. 什么是向量

一个二维向量可以理解为平面坐标轴中的一个坐标点(x，y)，在编程领域，一个二维向量就是一个大小为二的float类型的数组。

# 2. 文本向量化

文本向量化是指，利用大模型可以把一个字、一个词或一段话映射为一个多维向量。这样，我们可以基于向量来判断两句话之间的相似度。

我们可以直接在LangChain4j中来调用向量模型来对一句话进行向量化体验。

```java
import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.model.output.Response;
import dev.langchain4j.model.zhipu.ZhipuAiEmbeddingModel;

public class Main {

    public static void main(String[] args) {
        ZhipuAiEmbeddingModel embeddingModel = ZhipuAiEmbeddingModel.builder()
                .apiKey("xxx")
                .build();

        Response<Embedding> embed = embeddingModel.embed("你好，我叫张三");
        System.out.println(embed.content().toString());
        System.out.println(embed.content().vector().length);

    }
}
```

![image-20250729195248503](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250729195248503.png)

从结果可以知道"你好，我叫张三"这句话经过ZhipuAiEmbeddingModel向量化之后得到的一个长度为1024的float数组。

# 3. 向量数据库

对于向量模型生成出来的向量，我们可以持久化到向量数据库，并且能利用向量数据库来计算两个向量之间的相似度，或者根据一个向量查找跟这个向量最相似的向量。

在LangChain4j中，EmbeddingStore表示向量数据库，它有3个实现类：

1. ChromaEmbeddingStore
2. InMemoryEmbeddingStore

3. RedisEmbeddingStore

我们使用Redis来演示对于向量的增删查改

## 1 使用Redis向量数据库

### 1. 启动redis

普通的Redis是不支持向量存储和查询的，需要额外的redisearch模块，这里直接使用docker来运行一个带有redisearch模块的redis容器的，命令为：

```shell
docker run -p 6379:6379 docker.1ms.run/redis/redis-stack-server
```

### 2. 添加依赖

```xml
<dependency>
	<groupId>dev.langchain4j</groupId>
	<artifactId>langchain4j-redis</artifactId>
	<version>${langchain4j.version}</version>
</dependency>
```

### 3. 代码存储

```java
RedisEmbeddingStore embeddingStore = RedisEmbeddingStore.builder()
    .host("127.0.0.1")
    .port(6379)
    .dimension(1024) 
    .build();

// 生成向量
Response<Embedding> embed = embeddingModel.embed("我是张三");

// 存储向量
embeddingStore.add(embed.content());
```

> 当你用 OpenAI、文心一言、HuggingFace 等模型生成 **文本向量（embedding）** 时，模型会输出一个 **固定长度的浮点数组**，这个数组的长度就是 **向量维度dimension**
>
> - dimension就是告诉 RedisEmbeddingStore：“我这个向量的长度是多少，存储的时候按这个维度来建索引”
> - 可以使用以下命令来清空：`redis-cli FT.DROPINDEX embedding-index DD`

# 4. 匹配向量

演示如何根据一个查询（“客服电话多少”）找到与之最相关的文本片段。

```java
import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.output.Response;
import dev.langchain4j.model.zhipu.ZhipuAiEmbeddingModel;
import dev.langchain4j.store.embedding.EmbeddingMatch;
import dev.langchain4j.store.embedding.redis.RedisEmbeddingStore;

import java.util.List;

public class Main {

    public static void main(String[] args) {
        // 创建智谱AI的嵌入模型实例，使用提供的API密钥
        ZhipuAiEmbeddingModel embeddingModel = ZhipuAiEmbeddingModel.builder()
                .apiKey("xxx")
                .build();
        // 创建一个基于Redis的向量存储实例，配置为连接到本地运行的Redis服务器，端口6379，向量维度1024
        RedisEmbeddingStore embeddingStore = RedisEmbeddingStore.builder()
                .host("127.0.0.1")
                .port(6379)
                .dimension(1024)
                .build();
        // 创建三个文本片段
        TextSegment textSegment1 = TextSegment.textSegment("客服电话是400-8558558");
        TextSegment textSegment2 = TextSegment.textSegment("客服工作时间是周一到周五");
        TextSegment textSegment3 = TextSegment.textSegment("客服投诉电话是400-8668668");
        // 使用智谱AI的嵌入模型对这三个文本片段进行编码，得到它们的向量表示
        Response<Embedding> embed1 = embeddingModel.embed(textSegment1);
        Response<Embedding> embed2 = embeddingModel.embed(textSegment2);
        Response<Embedding> embed3 = embeddingModel.embed(textSegment3);
        // 将这些向量及其对应的原始文本片段添加到Redis向量存储中
        embeddingStore.add(embed1.content(), textSegment1);
        embeddingStore.add(embed2.content(), textSegment2);
        embeddingStore.add(embed3.content(), textSegment3);
        // 对查询文本“客服电话多少”也进行编码，得到其向量表示
        Response<Embedding> embed = embeddingModel.embed("客服电话多少");
        // 在Redis向量存储中查找与查询向量最相似的前5个文本片段
        List<EmbeddingMatch<TextSegment>> result = embeddingStore.findRelevant(embed.content(), 5);
        // 输出结果：匹配的文本片段和它们的相似度分数
        for (EmbeddingMatch<TextSegment> embeddingMatch : result) {
            System.out.println(embeddingMatch.embedded().text() + ",分数为：" + embeddingMatch.score());
        }
    }
}
```

输出结果：

<img src="https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250729221221418.png" alt="image-20250729221221418" style="zoom:50%;" />
