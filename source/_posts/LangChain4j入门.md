---
title: LangChain4j入门
date: 2025-07-20 15:19:05
tags:
categories: AI应用开发
---

# 1. 什么是LangChain4j

它是Java版本的LangChain，提供了一个开发框架，使得开发者可以很容易的用来构建具有LLM能力的应用程序。如何将大模型能力和Java编程语言相结合，这就是LangChain4j所做的。

> LLM就是Large Language Model，也就是常说的大语言模型，简称大模型。

# 2. langchain4j集成OpenAi（Java）

## 1. 引入依赖

```xml
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j</artifactId>
            <version>${langchain4j.version}</version>
        </dependency>
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-open-ai</artifactId>
            <version>${langchain4j.version}</version>
        </dependency>
```

## 2. 普通调用

```java
import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.openai.OpenAiChatModel;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Test {
    public static final String API_KEY = "sk-peszVtFXoLnWK45bB15370Df6f344cAa9a088eF50f9c7302";
    
    public static void main(String[] args) {
        ChatLanguageModel model = OpenAiChatModel
                .builder()
                .apiKey(API_KEY) // 需要替换成自己去open AI申请的API Key
                .modelName("gpt-4o-mini")
                .build();

        String answer = model.generate("你好，你是谁？");
        System.out.println(answer);
    }
}
```

> - 默认baseUrl 是https://api.openai.com/v1 ,具体可查看源码OpenAiChatModel
>
> - open AI 查看api key的地址：https://platform.openai.com/api-keys

**运行结果**：

![image-20250720152422613](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250720152422613.png)

## 3. 流式调用

```java
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.StreamingResponseHandler;
import dev.langchain4j.model.chat.StreamingChatLanguageModel;
import dev.langchain4j.model.openai.OpenAiStreamingChatModel;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Test {
    public static final String API_KEY = "sk-peszVtFXoLnWK45bB15370Df6f344cAa9a088eF50f9c7302";

    public static void main(String[] args) {
        StreamingChatLanguageModel model = OpenAiStreamingChatModel.builder()
                .apiKey(API_KEY)
                .modelName("gpt-4o-mini")
                .build();

        model.generate("你好，你是谁？", new StreamingResponseHandler<AiMessage>() {
            @Override
            public void onNext(String token) {
                System.out.println("当前返回的内容为：" + token);
            }

            @Override
            public void onError(Throwable error) {
                System.out.println(error);
            }
        });
    }
}
```

运行结果（打字机效果）：

<img src="https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250720154157785.png" alt="image-20250720154157785" style="zoom:50%;" />

# 3. langchain4j集成OpenAi（SpringBoot）

## 1. 引入依赖

```xml
<dependency>
	<groupId>dev.langchain4j</groupId>
	<artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
	<version>0.27.1</version>
</dependency>
```

## 2. 配置api-key

```properties
langchain4j.open-ai.chat-model.api-key=demo
```

> 在底层在构造OpenAiChatModel时，会判断传入的ApiKey是否等于"demo"，如果等于会将OpenAi的原始API地址"https://api.openai.com/v1"改为"http://langchain4j.dev/demo/openai/v1"，这个地址是langchain4j专门为我们准备的一个体验地址，实际上这个地址相当于是"https://api.openai.com/v1"的代理，我们请求代理时，代理会去调用真正的OpenAi接口，只不过代理会将自己的ApiKey传过去，从而拿到结果返回给我们。 
>
> 这个key不支持流式响应，所以真正开发，还是要替换自己的Key。

## 3. 调用

```java
import dev.langchain4j.model.chat.ChatLanguageModel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @Autowired
    private ChatLanguageModel chatLanguageModel;

    @GetMapping("/hello")
    public String hello(){
        return chatLanguageModel.generate("你好啊");
    }
}
```

# 4. langchain4j内容审核之Moderation

ModerationModel能够校验输入中是否存在敏感内容。

```java
import dev.langchain4j.model.moderation.Moderation;
import dev.langchain4j.model.moderation.ModerationModel;
import dev.langchain4j.model.openai.OpenAiModerationModel;
import dev.langchain4j.model.output.Response;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TestModeration {
    public static void main(String[] args) {
        ModerationModel moderationModel = OpenAiModerationModel.withApiKey("demo");
        Response<Moderation> response = moderationModel.moderate("我要杀了你");
        System.out.println("违反OpenAI 的使用策略: " + response.content().flagged());
        System.out.println("违规内容为：" + response.content().flaggedText());
    }
}
```

返回结果：

<img src="https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250720160406631.png" alt="image-20250720160406631" style="zoom:50%;" />

# 5. langchain4j生成图片之ImageModel

默认提供的“demo”key不能用来生成图片，需要大家自己购买apiKey

```java
import dev.langchain4j.data.image.Image;
import dev.langchain4j.model.image.ImageModel;
import dev.langchain4j.model.openai.OpenAiImageModel;
import dev.langchain4j.model.output.Response;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class _01_Main {
    public static final String API_KEY = "xxx";
  
    public static void main(String[] args) {
        ImageModel imageModel = OpenAiImageModel.builder()
                .apiKey(API_KEY)
                .build();
        Response<Image> response = imageModel.generate("一辆车");
        System.out.println(response.content().url());
    }
}
```

