---
title: LangChain4j之ChatMemory
date: 2025-07-21 21:57:01
tags:
categories: AI应用开发
---

# 1. **ChatMemory**是什么？

ChatMemory是LangChain4j提供的用来存储历史对话的组件，并且还支持窗口限制、淘汰机制、持久化机制等等扩展功能。

# 2. 使用

## 1. 基本使用

```java
import dev.langchain4j.memory.ChatMemory;
import dev.langchain4j.memory.chat.MessageWindowChatMemory;
import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.zhipu.ZhipuAiChatModel;
import dev.langchain4j.service.AiServices;
import dev.langchain4j.service.UserMessage;

public class Main {

    interface NamingMaster {
        String name(@UserMessage String desc);
    }

    public static void main(String[] args) {

        ChatLanguageModel model = ZhipuAiChatModel
                .builder()
                .apiKey("xxx")
                .build();

        ChatMemory chatMemory = MessageWindowChatMemory.withMaxMessages(10);

        NamingMaster namingMaster = AiServices.builder(NamingMaster.class).chatLanguageModel(model)
                .chatMemory(chatMemory)
                .build();

        String name = namingMaster.name("给我一个姓王男孩名字，不用解释");
        System.out.println("===========================第一次返回=====================================");
        System.out.println(name);

        String name2 = namingMaster.name("换一个");
        System.out.println("===========================第二次返回=====================================");
        System.out.println(name2);
    }
}

```

返回结果：

<img src="https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250721202648342.png" alt="image-20250721202648342" style="zoom:50%;" />

## 2. 基于MemoryId

上面的代码，一个用户使用没有问题，但如果多个用户使用的是同一个ChatMemory，那就会出现这多个用户的对话记录混杂在一起的问题。

因此，需要改造一下AiServices中设置ChatMemory的方式，让每个不同的userId对应不同的ChatMemory对象。

```java
import dev.langchain4j.memory.chat.TokenWindowChatMemory;
import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.openai.OpenAiTokenizer;
import dev.langchain4j.model.zhipu.ZhipuAiChatModel;
import dev.langchain4j.service.AiServices;
import dev.langchain4j.service.MemoryId;
import dev.langchain4j.service.UserMessage;

public class Main {

    interface NamingMaster {
        String name(@MemoryId String userId, @UserMessage String desc);
    }

    public static void main(String[] args) {
        ChatLanguageModel model = ZhipuAiChatModel
                .builder()
                .apiKey("xxx")
                .build();

        NamingMaster namingMaster = AiServices.builder(NamingMaster.class).chatLanguageModel(model)
                //限制会话历史记录的最大token数为10000，并使用OpenAI的分词器进行token计数。
                .chatMemoryProvider(userId -> TokenWindowChatMemory.builder().id(userId).maxTokens(10000,new OpenAiTokenizer()).build())
                .build();

        String name = namingMaster.name("1", "给我一个姓王的男孩名字，不用解释");
        System.out.println("===========================第一次返回=====================================");
        System.out.println(name);

        String name1 = namingMaster.name("2", "给我一个最好吃的水果名， 不用解释");
        System.out.println("===========================第二次返回=====================================");
        System.out.println(name1);

        String name2 = namingMaster.name("1", "换一个");
        System.out.println("===========================第三次返回=====================================");
        System.out.println(name2);
    }
}

```

返回结果：

<img src="https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250721203025165.png" alt="image-20250721203025165" style="zoom:50%;" />

# 3. 源码实现

ChatMemory是一个接口，默认提供了两个实现类：

1. MessageWindowChatMemory
2. TokenWindowChatMemory

## 1. ChatMemoryStore

这两个实现类内部都有一个ChatMemoryStore属性，ChatMemoryStore也是一个接口，默认有一个InMemoryChatMemoryStore实现类。

本质上就是一个ConcurrentHashMap，所以原理上我们可以自定义ChatMemoryStore的实现类来实现将ChatMessage持久化到磁盘，

```java
public class InMemoryChatMemoryStore implements ChatMemoryStore {
    private final Map<Object, List<ChatMessage>> messagesByMemoryId = new ConcurrentHashMap();

    public InMemoryChatMemoryStore() {
    }

    public List<ChatMessage> getMessages(Object memoryId) {
        return (List)this.messagesByMemoryId.computeIfAbsent(memoryId, (ignored) -> {
            return new ArrayList();
        });
    }

    public void updateMessages(Object memoryId, List<ChatMessage> messages) {
        this.messagesByMemoryId.put(memoryId, messages);
    }

    public void deleteMessages(Object memoryId) {
        this.messagesByMemoryId.remove(memoryId);
    }
}
```

## 2. MessageWindowChatMemory

那么MessageWindowChatMemory除开可以存储ChatMessage之外，还有淘汰机制,可以设置maxMessages，超过maxMessages会淘汰最旧的ChatMessage，SystemMessage不会被淘汰。

```java
    public void add(ChatMessage message) {
        List<ChatMessage> messages = this.messages();
        if (message instanceof SystemMessage) {
            Optional<SystemMessage> systemMessage = findSystemMessage(messages);
          // 替换SystemMessage
            if (systemMessage.isPresent()) {
                if (((SystemMessage)systemMessage.get()).equals(message)) {
                    return;
                }
                messages.remove(systemMessage.get()); 
            }
        }

        messages.add(message);
      // 如果超过了maxMessages限制，则会淘汰List最前面的，也就是最旧的ChatMessage
	    // 注意，SystemMessage不会被淘汰
        ensureCapacity(messages, this.maxMessages);
        this.store.updateMessages(this.id, messages);
    }

```



## 3. TokenWindowChatMemory

TokenWindowChatMemory和MessageWindowChatMemory类似，区别在于计算容量的方式不一样，MessageWindowChatMemory直接取的是List<ChatMessage>的大小，而TokenWindowChatMemory会利用指定的Tokenizer对List<ChatMessage>对应的Token数进行估算，然后和设置的maxTokens进行比较，超过maxTokens也会进行淘汰，也是淘汰最旧的ChatMessage。

Tokenizer是一个接口，默认提供了OpenAiTokenizer实现类，是用来估算一条ChatMessage对应多少个Token的，很多大模型的API都是按使用的Token数来收费的，所以在对成本比较敏感时，建议使用TokenWindowChatMemory来对一个会话使用的总Token数进行控制。

```java
    public void add(ChatMessage message) {
        List<ChatMessage> messages = this.messages();
        if (message instanceof SystemMessage) {
            Optional<SystemMessage> maybeSystemMessage = findSystemMessage(messages);
            if (maybeSystemMessage.isPresent()) {
                if (((SystemMessage)maybeSystemMessage.get()).equals(message)) {
                    return;
                }

                messages.remove(maybeSystemMessage.get());
            }
        }

        messages.add(message);
       // 如果超过了maxTokens限制，则会淘汰List最前面的
        ensureCapacity(messages, this.maxTokens, this.tokenizer);
        this.store.updateMessages(this.id, messages);
    }
```

