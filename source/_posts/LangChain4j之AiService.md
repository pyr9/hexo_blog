---
title: LangChain4j之AiService
date: 2025-07-20 16:35:02
tags:
categories: AI应用开发
---

# 1. AiService是什么？

`LangChain4j` 的 `AiService` 是一个非常方便的工具，它允许开发者通过定义接口和使用注解的方式快速创建基于语言模型的服务。这个机制极大地简化了与语言模型（如 Qwen、ChatGPT 等）交互的过程，使得开发者可以更专注于业务逻辑而非底层通信细节。

### 主要概念

1. **接口定义**：你首先需要定义一个接口，其中的方法代表了你希望语言模型执行的任务。
2. **注解**：在方法上添加特定的注解来指导LangChain4j如何处理这些请求，例如：
   - `@SystemMessage`：指定系统消息或角色描述。
   - `@UserMessage`：标记用户输入的消息。
   - `@Moderate`：启用内容审核。
   - `@V`：用于变量插值。
3. **动态代理**：`AiServices.builder()` 会根据你的接口定义生成一个动态代理对象，该对象会在运行时将调用转发给实际的语言模型，并返回结果。

# 2. AiService使用

- 创建ChatLanguageModel

  ```java
  import dev.langchain4j.model.chat.ChatLanguageModel;
  import dev.langchain4j.model.zhipu.ZhipuAiChatModel;
  
  public class ModelFactory {
      public static ChatLanguageModel buildChatLanguageModel() {
          String API_KEY = "xxx";
          return ZhipuAiChatModel
                  .builder()
                  .apiKey(API_KEY)
                  .build();
      }
  }
  ```

## 1. @SystemMessage指定系统消息

```java
import dev.langchain4j.service.AiServices;
import dev.langchain4j.service.SystemMessage;

public interface WriterAiService {

    @SystemMessage("请扮演一名作家，根据输入的文章题目写一篇200字以内的作文")
    String write(String title);

    static WriterAiService create() {
        return AiServices.builder(WriterAiService.class)
                .chatLanguageModel(ModelFactory.buildChatLanguageModel())
                .build();
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        WriterAiService writer = WriterAiService.create();
        String result = writer.write("我最爱的人");
        System.out.println(result);
    }
}
```

输出结果：

![image-20250720164749767](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250720164749767.png)



## 2. `@UserMessage`：标记用户输入的消息。

```java
import dev.langchain4j.service.AiServices;
import dev.langchain4j.service.SystemMessage;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.V;

public interface WriterAiService {

    @SystemMessage("请扮演一名作家，根据输入的文章题目写一篇{{count}}字以内的作文")
    String write(@UserMessage String title, @V("count") Integer count);

    static WriterAiService create() {
        return AiServices.builder(WriterAiService.class)
                .chatLanguageModel(ModelFactory.buildChatLanguageModel())
                .build();
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        WriterAiService writer = WriterAiService.create();
        String result = writer.write("我最爱的人", 30);
        System.out.println(result);
    }
}
```

输出结果：

<img src="https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250720164944981.png" alt="image-20250720164944981" style="zoom:50%;" />

## 3.  系统提示词写在文件里

```java
import dev.langchain4j.service.*;

public interface WriterAiService {

    @SystemMessage(fromResource = "/system_message.txt")
    String write(@UserMessage String title, @V("count") Integer count);

    static WriterAiService create() {
        return AiServices.builder(WriterAiService.class)
                .chatLanguageModel(ModelFactory.buildChatLanguageModel())
                .build();
    }
}
```

## 4. `@Moderate`：启用内容审核

增加@Moderate注解，同时需要增加moderationModel

```java
import dev.langchain4j.data.message.ChatMessage;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.input.Prompt;
import dev.langchain4j.model.moderation.Moderation;
import dev.langchain4j.model.moderation.ModerationModel;
import dev.langchain4j.model.output.Response;
import dev.langchain4j.service.*;

import java.util.List;

public interface WriterAiService {

    @SystemMessage("请扮演一名作家，写一篇杀人{{count}}字以内的作文")
    @Moderate
    String write(@UserMessage String title, @V("count") Integer count);

    static WriterAiService create() {
        return AiServices.builder(WriterAiService.class)
                .chatLanguageModel(ModelFactory.buildChatLanguageModel())
                .moderationModel(new ModerationModel() {
                    @Override
                    public Response<Moderation> moderate(String s) {
                        // 在这里实现具体的审核逻辑
                        return Response.from(Moderation.flagged(s));
                    }

                    @Override
                    public Response<Moderation> moderate(Prompt prompt) {
                        // 在这里实现具体的审核逻辑
                        return ModerationModel.super.moderate(prompt);
                    }

                    @Override
                    public Response<Moderation> moderate(ChatMessage message) {
                        // 在这里实现具体的审核逻辑
                        return ModerationModel.super.moderate(message);
                    }

                    @Override
                    public Response<Moderation> moderate(List<ChatMessage> list) {
                        // 在这里实现具体的审核逻辑
                        return Response.from(Moderation.flagged(list.toString()));
                    }


                    @Override
                    public Response<Moderation> moderate(TextSegment textSegment) {
                        // 在这里实现具体的审核逻辑
                        return ModerationModel.super.moderate(textSegment);
                    }
                })
                .build();
    }
```



# 3. 源码分析

```java
AiServices.builder(WriterAiService.class)
                .chatLanguageModel(ModelFactory.buildChatLanguageModel())
                .build()
```

由于AiServices是一个抽象类，源码中有一个默认的子类DefaultAiServices，核心实现源码都在DefaultAiServices中。

DefaultAiServices的build方法就是用来创建指定接口的代理对象：

```java
public T build() {

	// 验证是否配置了ChatLanguageModel
	performBasicValidation();

	// 验证接口中是否有方法上加了@Moderate，但是又没有配置ModerationModel
	for (Method method : context.aiServiceClass.getMethods()) {
		if (method.isAnnotationPresent(Moderate.class) && context.moderationModel == null) {
			throw illegalConfiguration("The @Moderate annotation is present, but the moderationModel is not set up. " +
									   "Please ensure a valid moderationModel is configured before using the @Moderate annotation.");
		}
	}

	// JDK动态代理创建代理对象
	Object proxyInstance = Proxy.newProxyInstance(
		context.aiServiceClass.getClassLoader(),
		new Class<?>[]{context.aiServiceClass},
		new InvocationHandler() {

			@Override
			public Object invoke(Object proxy, Method method, Object[] args) throws Exception {
				//...
        // 验证参数
         DefaultAiServices.validateParameters(method);
        // 解析系统消息
         Optional<SystemMessage> systemMessage = DefaultAiServices.this.prepareSystemMessage(method, args);
        // 解析用户消息
         dev.langchain4j.data.message.UserMessage userMessage = DefaultAiServices.prepareUserMessage(method, args);
        // 这里去使用chatModel去调用openAi, 生成返回的结果
       response = DefaultAiServices.this.context.chatModel.generate((List)messages, DefaultAiServices.this.context.toolSpecifications);

        // ...
			}

		});

	return (T) proxyInstance;
}
```

可以发现，其实就是用的JDK动态代理机制创建的代理对象，只不过在创建代理对象之前有两步验证：

1. 验证是否配置了ChatLanguageModel：这一步不难理解，如果代理对象没有配置ChatLanguageModel，那就利用不上大模型的能力了
2. 验证接口中是否有方法上加了@Moderate，但是又没有配置ModerationModel
