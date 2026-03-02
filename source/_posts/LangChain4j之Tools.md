---
title: LangChain4j之Tools
date: 2025-07-23 22:11:53
tags:
categories: AI应用开发
---

# 1. **Tools是什么？**

大模型Tools（工具）可以通过结构化接口将模型能力与现实世界操作连接。例如：

- **天气查询工具**：封装天气API，接收城市参数后返回实时数据；
- **数据库操作工具**：将SQL查询转化为自然语言交互接口；
- **代码执行工具**：调用编译器或解释器完成代码调试。

> **目前大模型的不足**：大模型在解决问题时，是基于互联网上很多历史资料进行预测的，而且答案具有一定的随机性，那如果我问"今天是几月几号？"，大模型是大概率答错的，因为大模型肯定还没有来得及学习今天所产生的最新资料。

# 2. AiServices整合Tools

```java
import com.google.common.collect.Lists;
import dev.langchain4j.agent.tool.Tool;
import dev.langchain4j.memory.chat.MessageWindowChatMemory;
import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.zhipu.ZhipuAiChatModel;
import dev.langchain4j.service.AiServices;
import dev.langchain4j.service.SystemMessage;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.ToString;

import java.lang.reflect.InvocationTargetException;
import java.time.LocalDate;
import java.time.LocalDateTime;

public class Main {

    @Data
    @ToString
    @AllArgsConstructor
    @NoArgsConstructor
    static class User {
        private String name;
        private Integer age;
        private String createTime;
    }

    static class MyTools {

        @Tool("用来获取今天具体日期")
        public String dateUtil(String onUse) {
            return LocalDateTime.now().toString();
        }

        @Tool("获取指定日期的用户信息")
        public String getUserInfo(String date) {
            User user1 = new User("张三", 88, LocalDateTime.now().toLocalDate().toString());
            User user2 = new User("李四", 99, "2021-07-23");
            return Lists.newArrayList(user1, user2).stream()
                    .filter(u -> LocalDate.parse(u.createTime).equals(LocalDate.parse(date)))
                    .toList().toString();
        }

    }

    interface UserService {
        @SystemMessage("先获取当前具体的日期，然后再解决用户问题")
        String getUserInfo(String desc);
    }

    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        ChatLanguageModel model = ZhipuAiChatModel
                .builder()
                .apiKey("5b755da0b2b04a8c9ba449808b72a474.uaEUl7Tim6xXHghK")
                .build();

        UserService userService = AiServices.builder(UserService.class).chatLanguageModel(model)
                .tools(new MyTools())
                .chatMemory(MessageWindowChatMemory.withMaxMessages(10))
                .build();

        String userInfo = userService.getUserInfo("获取今天注册的用户信息");
        System.out.println(userInfo);
    }
}

```

返回结果：

![image-20250723221332643](https://panyuro.oss-cn-beijing.aliyuncs.com/image-20250723221332643.png)
