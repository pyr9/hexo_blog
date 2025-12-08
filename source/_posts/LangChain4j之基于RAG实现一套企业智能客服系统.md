---
title: LangChain4j之基于RAG实现一套企业智能客服系统
date: 2025-08-03 21:55:24
tags:
categories: AI应用开发
---

# 1. 前期准备

首先，我们需要把现有的常见文件整理成文档，可以是txt、pdf、xlsx、markdown等格式都可以，我们这里将问题和答案转成txt文件，文件为：外卖常见问题.txt

```txt
Q：在线支付取消订单后钱怎么返还？
订单取消后，款项会在一个工作日内，直接返还到您的美团账户余额。

Q：怎么查看退款是否成功？
退款会在一个工作日之内到美团账户余额，可在“账号管理——我的账号”中查看是否到账。

Q：美团账户里的余额怎么提现？
余额可到美团网（meituan.com）——“我的美团→美团余额”里提取到您的银行卡或者支付宝账号，另外，余额也可直接用于支付外卖订单（限支持在线支付的商家）。

Q：余额提现到账时间是多久？
2-5个工作日内可退回您的支付账户。由于银行处理可能有延迟，具体以账户的到账时间为准。

Q：申请退款后，商家拒绝了怎么办？
申请退款后，如果商家拒绝，此时回到订单页面点击“退款申诉”，美团客服介入处理。
```



# 2. 代码实现

## 2.1. 引入依赖

直接创建一个普通的Maven工程就可以了，然后引入langchain4j的依赖和你选择的大模型依赖，我这里使用open-ai：

```json
<dependency>
  <groupId>dev.langchain4j</groupId>
  <artifactId>langchain4j</artifactId>
  <version>0.27.1</version>
</dependency>

<dependency>
  <groupId>dev.langchain4j</groupId>
  <artifactId>langchain4j-open-ai</artifactId>
  <version>0.27.1</version>
</dependency>
```

## 2.2. 导入知识库

### 2.2.1. 使用DocumentParser加载并解析文件

```java
import dev.langchain4j.data.document.Document;
import dev.langchain4j.data.document.DocumentParser;
import dev.langchain4j.data.document.loader.FileSystemDocumentLoader;
import dev.langchain4j.data.document.parser.TextDocumentParser;

import java.net.URISyntaxException;
import java.nio.file.Path;
import java.nio.file.Paths;

public class DocumentUtil {
    // 加载并解析文件
    public static Document loadDocument(String fileName) {
        Document document;
        try {
            Path documentPath = Paths.get(CustomerServiceAgent.class.getClassLoader().getResource(fileName).toURI());
            DocumentParser documentParser = new TextDocumentParser();
            document = FileSystemDocumentLoader.loadDocument(documentPath, documentParser);
        } catch (URISyntaxException e) {
            throw new RuntimeException(e);
        }
        return document;
    }
}

```

使用FileSystemDocumentLoader来加载本地文件，利用TextDocumentParser来解析txt文件，最终得到文件所对应的Document对象。

### 2.2.2. 使用DocumentSplitter切分文件成问答对

```java
import dev.langchain4j.data.document.Document;
import dev.langchain4j.data.document.DocumentSplitter;
import dev.langchain4j.data.segment.TextSegment;

import java.util.ArrayList;
import java.util.List;

public class CustomerServiceDocumentSplitter implements DocumentSplitter {

    @Override
    public List<TextSegment> split(Document document) {

        List<TextSegment> segments = new ArrayList<>();

        String[] parts = split(document.text());
        for (String part : parts) {
            segments.add(TextSegment.from(part));
        }

        return segments;
    }

    public String[] split(String text) {
        return text.split("\\s*\\R\\s*\\R\\s*");
    }
}

```

### 2.2.3. 基于Embedding+redis文本向量化并存储

```java
import dev.langchain4j.data.document.Document;
import dev.langchain4j.data.document.DocumentSplitter;
import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.embedding.EmbeddingModel;
import dev.langchain4j.model.output.Response;
import dev.langchain4j.model.zhipu.ZhipuAiEmbeddingModel;
import dev.langchain4j.rag.content.retriever.ContentRetriever;
import dev.langchain4j.rag.content.retriever.EmbeddingStoreContentRetriever;
import dev.langchain4j.store.embedding.redis.RedisEmbeddingStore;
import redis.clients.jedis.Jedis;

import java.util.ArrayList;
import java.util.List;

public class EmbeddingStoreInitializer {

    public static ContentRetriever initContentRetriever(String fileName, String apiKey) {
        // Redis 存储向量
        RedisEmbeddingStore embeddingStore = RedisEmbeddingStore.builder()
                .host("127.0.0.1")
                .port(6379)
                .dimension(1024)
                .build();

        try (Jedis jedis = new Jedis("127.0.0.1", 6379)) {
            if (!jedis.exists("knowledge_base_initialized")) {
                System.out.println("首次初始化知识库...");
                // 1. 加载文件
                Document document = DocumentUtil.loadDocument(fileName);

                // 2. 切分文件
                DocumentSplitter splitter = new CustomerServiceDocumentSplitter();
                List<TextSegment> segments = splitter.split(document);
                System.out.println("=====================文档切分的结果为=====================");
                segments.forEach(System.out::println);
              
                // 3. 文本向量化存储
                EmbeddingModel embeddingModel = ZhipuAiEmbeddingModel.builder()
                        .apiKey(apiKey)
                        .build();
                List<Embedding> embeddings = new ArrayList<>();
                for (TextSegment segment : segments) {
                    Response<Embedding> response = embeddingModel.embed(segment);
                    embeddings.add(response.content());
                }
                embeddingStore.addAll(embeddings, segments);
                jedis.set("knowledge_base_initialized", "true");
                System.out.println("知识库初始化完成！");
            } else {
                System.out.println("知识库已存在，跳过初始化。");
            }
        }

        // 4. 构建 ContentRetriever（只负责检索）
        return EmbeddingStoreContentRetriever.builder()
                .embeddingStore(embeddingStore)
                .embeddingModel(ZhipuAiEmbeddingModel.builder().apiKey(apiKey).build())
                .maxResults(5)
                .minScore(0.6)
                .build();
    }
}
```

这里是使用redis（redis/redis-stack-server： 带有redisearch模块的redis容器）可以使用以下命令来查看：

```java
redis-cli FT.SEARCH embedding-index "*" LIMIT 0 10
```



## 2.3. 增加tools返回具体的日期

```java
public class DateCalculator {

    @Tool("计算指定天数后的具体日期")
    String date(Integer days) {
        return LocalDate.now().plusDays(days).toString();
    }
}
```



## 2.4. 定义Agent (AiService）

直接用这个Agent来充当客服回答问题

```java
import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.zhipu.ZhipuAiChatModel;
import dev.langchain4j.rag.content.retriever.ContentRetriever;
import dev.langchain4j.service.AiServices;

public interface CustomerServiceAgent {

    // 用来回答问题的方案
    String answer(String question);

    static CustomerServiceAgent create() {
        String apiKey = "xxx";

        ChatLanguageModel model = ZhipuAiChatModel
                .builder()
                .apiKey(apiKey)
                .build();

        ContentRetriever contentRetriever = EmbeddingStoreInitializer.initContentRetriever(
                "外卖常见问题.txt", apiKey
        );

        // 指定模型，创建并返回代理对象
        return AiServices.builder(CustomerServiceAgent.class)
                .chatLanguageModel(model)
                .contentRetriever(contentRetriever)
                .tools(new DateCalculator())
                .build();
    }
   // 测试Agent:
    static void main(String[] args){
        CustomerServiceAgent customerServiceAgent = CustomerServiceAgent.create();
        System.out.println("=====================测试agent======================");
        String result=customerServiceAgent.answer("今天的余额提现，哪一天能到账，要输出具体日期?");
        System.out.println(result);
    }
}
```

![image-20250805214925133](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250805214925133.png)
