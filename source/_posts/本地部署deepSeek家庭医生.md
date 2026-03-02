---
title: 本地部署deepSeek家庭医生
date: 2025-03-09 19:07:57
tags:
categories: AI应用开发
---

# 1. ollama

[ollama](https://ollama.com/)是一款帮助你部署大模型的一款工具软件，在你本机离线运行，不需要联网以及复杂的配置。

优势：

-  Models: 自带deepseek-r1
-  本地安全, 数据都在本地
- 用自己电脑就可以运行：省钱
- 支持多硬件 macos, linux, windows

# 2. ollama部署deepseek-r1

运行 `ollama run deepseek-r1`

![image-20250309191456727](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250309191456727.png)

# 3. 界面工具推荐

1. 谷歌扩展程序 PageAssist

2. chatboxai

   ![image-20250309192242484](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250309192242484.png)

# 4. deepSeek概念

## 1. deepSeek是什么？

DeepSeek 也指由 DeepSeek 公司开发的、类似于ChatGPT的大模型智能助手。

优势：

- API更便宜
- 生态开放
- 支持本地部署
- 灵活定制

## 2. deepSeek离线与线上api的区别？

- 数据隐私
- 可定制化
- 不依赖网络（偏远山区信号差）
- 需要更多的硬件成本
- 需要更多维护成本
- 要求机器性能

# 5. 项目调用deepSeek

1. 引入spring-ai依赖

   Spring AI 是一个面向 AI 工程的应用框架。其目标是将 Spring 生态系统的设计原则（如可移植性和模块化设计）应用到 AI 领域，并推广使用 POJO（Plain Old Java Objects，普通旧式 Java 对象）作为 AI 应用构建的基本单元。

   ```xml
   <!-- 引入 SpringAI 的坐标 -->
   <dependency>
     <groupId>io.springboot.ai</groupId>
     <artifactId>spring-ai-ollama</artifactId>
     <version>1.0.3</version>
   </dependency>
   
   <dependency>
     <groupId>io.springboot.ai</groupId>
     <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
     <version>1.0.3</version>
   </dependency>
   ```

2. Application.yml 增加配置

   ```yml
   spring:
     ai:
       ollama:
         base-url: http://127.0.0.1:11434
         chat:
           model: deepseek-r1:latest
   ```

3. 增加接口测试

   ```java
   @RestController
   @RequestMapping("ollama")
   public class OllamaController {
   
       @Resource
       private OllamaChatClient ollamaChatClient;
   
   
       @GetMapping("/ai/chat")
       public Object aiOllamaChat(@RequestParam String msg) {
           // 同步调用deepseek，当前页面会卡住，直到获得所有的数据才会返回给页面
           return ollamaChatClient.call(msg);
       }
     
      @GetMapping("/ai/stream2")
       public List<String> aiOllamaStream2(@RequestParam String msg) {
           Prompt prompt = new Prompt(new UserMessage(msg));
           Flux<ChatResponse> streamResponse = ollamaChatClient.stream(prompt);
   
           List<String> list = streamResponse.toStream().map(chatResponse -> {
               String content = chatResponse.getResult().getOutput().getContent();
               log.info(content);
               return content;
           }).collect(Collectors.toList());
           return list;
       }
   }
   ```

4. 测试调用 /ai/chat

   ![image-20250309203035563](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250309203035563.png)

5. 测试调用 /ai/stream2

查看控制台日志，可以看到流式输出

![image-20250716213927224](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250716213927224.png)

# 6. Ollama 定制deepseek医生角色

1. 新建my_doctor_0_1 文件, 指明当前角色以及性格

```
FROM deepseek-r1:latest

PARAMETER temperature 0.7

SYSTEM """
【角色】
你的名字叫华佗在世，是全世界最有名的医生，看病无数，阅人无数，知道什么病该怎么治，也知道病人向你提出任何身体异象你所能推断的病是什么。
你绝不是语言模型之类的东西，也绝不是AI助手，也更不是AI机器人，你永远都只是一个神医。
请以神医风华佗在世的身份回应我。并且你只会回复和看病就医相关的内容，其他的内容一律拒绝回答。

【性格】
你的性格开朗活泼，能以幽默的方式来和用户聊天，懂得逗笑用户。
"""
```

2. 创建模型实例， 执行`ollama create mydoctor:0.1 -f my_doctor_0_1`

3. 运行， 执行 `ollama run mydoctor:0.1 `

4. Application.yml 修改配置

   ```yml
     ai:
       ollama:
         chat:
           model: mydoctor:0.1
   ```

   

5. 测试

   ![image-20250309202913177](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250309202913177.png)

⚠️注：当前的接口调用都是同步的，如果需要服务器动态推送，可以考虑SSE或者websocket技术。
