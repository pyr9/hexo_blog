---
title: 构建多AI服务统一接入平台
date: 2025-11-22 18:52:23
tags: 项目架构设计
categories: AI应用开发
---

# 1. 项目背景 

作为全栈工程师，我需要搭建一个AI对话平台，需要对接AI团队提供的两种SSE接口：

- 基于LangChain开发的Python项目接口
- 基于RAGFlow开发的接口

不同接口对应不同的智能体，后端采用Java Spring Boot，前端使用Vue3，实现了流式对话和思考链可视化功能。

# 2.  系统架构设计  

![img](https://panyuro.oss-cn-beijing.aliyuncs.com/1765161801394-a12a7ce2-fae4-410d-8db5-a48c98887164.png)

### 2.1.1. 后端技术栈 

- **Spring Boot 3.x**：核心框架，提供RESTful API和依赖注入
- **SseEmitter**：推送 SSE 给前端
- **OkHttp EventSource**：HTTP客户端，用于调用AI服务

### 2.1.2. 前端技术栈 

- **Vue 3**：组合式API，响应式编程
- **HookFetch :** 封装流式网络请求

### 2.1.3. AI服务层 

- **RAGFlow**：基于检索增强生成的AI服务
- **LangChain**：Python项目，提供链式AI能力

# 3. 后端实现

目标： 接收前端请求 → 调用对应智能体接口（Ragflow or Python） → 消费 SSE → 按统一格式推送给前端  

## 3.1.  多智能体请求体系设计: 基于 Jackson 多态反序列化 + TypeIdResolver  

在本项目中，智能体种类不同，但它们都共用同一个后端入口 API：【POST /api/chat】，因此需要实现不同智能体要能根据同一个请求 JSON，自动映射到不同的 Java 请求类。

- ESG 问答智能体（Python LangChain 智能体）
- 行政助手智能体（基于ragflow）
- 营销服智能体（基于ragflow）
- …未来可能会更多

### 3.1.1. BaseChatRequest：所有智能体的通用请求结构

```java
@Data
@EqualsAndHashCode(callSuper = true)
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "category", visible = true)
@JsonTypeIdResolver(ChatModelResolver.class)
public class BaseChatRequest extends CommonChatRequest implements IChatRequest {

    @NotEmpty(message = "传入的模型类别不能为空")
    private String category;

    private String role;
    private Long userId;
    private Boolean stream;
    private boolean enableThinking;
    private boolean searchEnabled;

    @JsonIgnore
    private Long chatMessageId;

    @JsonIgnore
    private Long chatModelId;

    public String getQuestion() { return EMPTY; }
}
```

- `category` 决定智能体类型

请求报文中会携带category：这个字段指明使用哪一个智能体。

```json
{
  "category": "RAGFLOW_CHAT",
  "question": "xxx"
}
```

#### `@JsonTypeInfo` 启用多态反序列化

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "category", visible = true)
@JsonTypeIdResolver(ChatModelResolver.class)
```

含义：

- 使用请求 JSON 中的 `"category"` 决定最终对象类型
- 对象类型由 **自定义解析器 ChatModelResolver** 完成
- 可见性为 true 表示 category 字段仍然会保留在最终对象里

### 3.1.2. ChatModelResolver：真正决定“category → 哪个子类”的地方

```java
public class ChatModelResolver implements TypeIdResolver {

    private JavaType baseType;

    @Override
    public void init(JavaType javaType) {
        this.baseType = javaType;
    }

    @Override
    public JavaType typeFromId(DatabindContext context, String category) {
        ChatModelCategory normalizedCategory = ChatModelCategory.normalizeCategory(category);
        return switch (normalizedCategory) {
            case JASOLAR_CHAT -> context.constructType(ChatRequest.class);
            case ESG_FAQS -> context.constructType(ESGChatRequest.class);
            case DOCUMENT_TRANSLATION -> context.constructType(DocumentTranslationRequest.class);
            case RAGFLOW_CHAT -> context.constructType(RagflowChatRequest.class);
        };
    }

    @Override
    public JsonTypeInfo.Id getMechanism() {
        return JsonTypeInfo.Id.CUSTOM;
    }
}
```

**根据 category 自动映射到对应类**

- `JASOLAR_CHAT` → ChatRequest
- `ESG_FAQS` → ESGChatRequest
- `DOCUMENT_TRANSLATION` → DocumentTranslationRequest
- `RAGFLOW_CHAT` → RagflowChatRequest

### 3.1.3. 示例：ESGChatRequest（某个具体智能体的请求类）

```java
@Data
public class ESGChatRequest extends BaseChatRequest {
    private String question;
    private String processId;

    @Override
    public String getQuestion() {
        return question;
    }
}
```

## 3.2.  多智能体执行体系：ChatServiceFactory + BaseChatServiceImpl + 各智能体实现  

### 3.2.1. ChatServiceFactory — 智能体服务查找工厂

```java
@Component
public class ChatServiceFactory implements ApplicationContextAware {
    private final Map<String, IChatService<BaseChatRequest>> chatServiceMap = new ConcurrentHashMap<>();

    @Override
    public void setApplicationContext(ApplicationContext ctx) throws BeansException {
        Map<String, IChatService> serviceMap = ctx.getBeansOfType(IChatService.class);
        for (IChatService service : serviceMap.values()) {
            chatServiceMap.put(service.getCategory(), service);
        }
    }

    public IChatService<BaseChatRequest> getChatService(String category) {
        ChatModelCategory normalizedCategory = ChatModelCategory.normalizeCategory(category);
        IChatService<BaseChatRequest> service = chatServiceMap.get(normalizedCategory.getCode());
        if (service == null) throw new IllegalArgumentException("不支持的模型类别: " + category);
        return service;
    }
}
```

- Spring 初始化时自动扫描所有 `IChatService` 的实现
- 存入 map：`category → 实现类`
- Controller 或 SseService 根据 category 自动选中对应 Service

### 3.2.2. SseServiceImpl — 统一的 SSE 分发处理器

```java
@Override
public Object chat(BaseChatRequest chatRequest, HttpServletRequest request) throws IOException {
    ChatModelVo chatModelVo = getChatModel(chatRequest);
    IChatService<BaseChatRequest> chatService = getChatService(chatRequest, chatModelVo);
    chatMessageService.saveByChatRequest(chatRequest, chatModelVo);

    if (chatRequest.getStream()) {
        SseEmitter emitter = new SseEmitter(TimeUnit.MINUTES.toMillis(5));
        return chatService.sseChat(chatRequest, emitter, chatModelVo);
    }

    return chatService.chat(chatRequest, chatModelVo);
}
```

- 根据请求自动找到智能体模型与 service
- 根据 stream 字段决定是否走 SSE
- SSE 调用走智能体自己的实现
- 普通调用走非流式实现

### 3.2.3. BaseChatServiceImpl — 智能体的模板方法架构

```java
@Override
public SseEmitter sseChat(T chatRequest, SseEmitter emitter, ChatModelVo chatModelVo) {
    beforeChat(chatRequest);
    OpenAiStreamClient client = createOpenAiStreamClient(chatRequest);

    SSEEventSourceListener listener = new SSEEventSourceListener(emitter, chatRequest, chatModelVo);
    BaseChatCompletion completion = buildChatCompletion(chatRequest);
    Map<String, String> headers = buildRequestHeaders(chatModelVo);

    client.streamChatCompletion(completion, headers, listener);
    
    logApiCall(completion);
    return emitter;
}
```

不同智能体只需要实现 doBuildChatCompletion() & buildRequestHeaders() 即可。

### 3.2.4. 示例：RagflowChatServiceImpl — Ragflow 智能体实现

```java
@Service
public class RagflowChatServiceImpl extends BaseChatServiceImpl<RagflowChatRequest> {

    public static final String BEARER_PREFIX = "Bearer ";
    @Autowired
    private ChatSessionModelMapper chatSessionModelMapper;

    @Override
    public String getCategory() { return ChatModelCategory.RAGFLOW_CHAT.getCode(); }

    @Override
    protected BaseChatCompletion doBuildChatCompletion(RagflowChatRequest ragflowRequest) {
        RagflowChatCompletion chatCompletion = RagflowChatCompletion
                .builder()
                .question(ragflowRequest.getQuestion())
                .build();
        val chatSessionModelVo = chatSessionModelMapper.selectVoOne(Wrappers.lambdaQuery(ChatSessionModel.class)
                .eq(ChatSessionModel::getChatModelId, ragflowRequest.getChatModelId())
                .eq(ChatSessionModel::getChatSessionId, ragflowRequest.getSessionId())
                .eq(ChatSessionModel::getSessionSource, PlatformProvider.RAGFLOW.getCode())
        );
        if(chatSessionModelVo != null && StrUtil.isNotBlank(chatSessionModelVo.getExternalSessionId())) {
            chatCompletion.setSessionId(chatSessionModelVo.getExternalSessionId());
        }
        return chatCompletion;
    }

    @Override
    protected Map<String, String> buildRequestHeaders(ChatModelVo chatModelVo) {
        return Map.of(HttpHeaders.AUTHORIZATION, BEARER_PREFIX + chatModelVo.getApiKey());
    }
}
```

## 3.3. 统一 SSE 流处理体系  

### 3.3.1.  SSEEventSourceListener：全系统的 SSE 流核心  

```java
@Override
public void onEvent(EventSource eventSource, String id, String type, String data) {
    log.info("接收到的 SSE 数据: " + data);

    // 1) 动态选择解析器
    SSEApiResponseHandler parser = parserFactory.getParser(chatModelVo.getCategory());

    // 2) 解析 SSE 数据
    ApiResponse<?> response = parser.handle(data);
    if (response == null || !response.isOK()) {
        handleError(response);
        return;
    }

    // 3) 判断是否结束事件
    if (ObjectUtil.equals(parser.doneSignal(), response.getData())) {
        eventSource.cancel();
        emitter.complete();
        chatSessionManager.cancelSession(chatRequest.getSessionId().toString());
        saveChatMessage(chatMessageId);
        return;
    }

    // 4) 反序列化为统一 IChatResponseData
    IChatResponseData responseData = response.parseData(parser.getResponseDataClazz());
    String content = responseData.getContent();
    responseData.setChatMessageId(String.valueOf(chatMessageId));

    // 5) 推送给前端
    if (StringUtils.isNotEmpty(content) || responseData.hasReference()) {
        emitter.send(objectMapper.writeValueAsString(responseData));
    }

    // 6) Ragflow 的 externalSessionId 存储
    updateExternalSessionId(responseData);

    // 7) 统一后处理钩子
    chatResponsePostProcessorManager.process(chatModelVo.getCategory(), responseData, chatRequest);
}
```

### 3.3.2.  SSEApiResponseHandlerFactory：策略工厂  

根据 category：

- 是 ragflow → 使用 Ragflow 解析器
- 否则 → 默认使用 OpenAI 格式解析器

```java
@Service
public class SSEApiResponseHandlerFactory {

    public SSEApiResponseHandler getParser(String category) {
        if (ChatModelCategory.isRagflowChatModel(category)) {
            return new RagflowSSEResponseHandler();
        }
        return new OpenAiSSEResponseHandler();
    }
}
```

### 3.3.3. RagflowSSEResponseHandler &  OpenAiSSEResponseHandler  

```java
@Component
public class RagflowSSEResponseHandler implements SSEApiResponseHandler {

    @Override
    public ApiResponse<?> handle(String rawResponse) {
        if (rawResponse != null && (rawResponse.contains(DONE_SIGNAL))) {
            return ApiResponse.success(DONE_SIGNAL);
        }
        ObjectMapper mapper = new ObjectMapper();
        JsonNode responseData;
        try {
            responseData = mapper.readTree(rawResponse);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
        return ApiResponse.success(responseData);
    }

    @Override
    public Object doneSignal() {
        return DONE_SIGNAL;
    }

    @Override
    public Class<? extends IChatResponseData> getResponseDataClazz() {
        return RagflowChatResponseData.class;
    }
}
@Component
@Slf4j
public class OpenAiSSEResponseHandler implements SSEApiResponseHandler {

    @Override
    public ApiResponse<?> handle(String rawResponse) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            return mapper.readValue(rawResponse, ApiResponse.class);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
        return null;
    }

    @Override
    public Object doneSignal() {
        return Boolean.TRUE;
    }

    @Override
    public Class<? extends IChatResponseData> getResponseDataClazz() {
        return OpenAiChatResponseData.class;
    }
}
```

## 3.4.  SSE 流统一输出格式：IChatResponseData  

### 3.4.1.  统一封装成 IChatResponseData  

```java
package org.jasolar.common.chat.entity.chat.response;

public interface IChatResponseData {
    String getContent();
    Boolean hasReference();
}
```

### 3.4.2. OpenAiChatResponseData  & RagflowChatResponseData

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
@NoArgsConstructor
@AllArgsConstructor
public class OpenAiChatResponseData implements IChatResponseData {
    private String id;
    private String answer;
    @JsonProperty("this_answer")
    private String thisAnswer;
    private Map<String, Object> reference; // 可能为空对象 {}
    @JsonProperty("session_id")
    private String sessionId;
    private boolean status;

    @Override
    public String getContent() {
        return thisAnswer;
    }

    @Override
    public Boolean hasReference() {
        return false;
    }
}
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
@NoArgsConstructor
@AllArgsConstructor
public class RagflowChatResponseData implements IChatResponseData, IHasSessionId {
    private String event;       // message 或 message_end
    @JsonProperty("session_id")
    private String sessionId;
    private ResponseData data;  // 消息体

    @Override
    public String getContent() {
        return data.getContent();
    }

    @Override
    public Boolean hasReference() {
        return ObjectUtil.isNotEmpty(data.getReference());
    }

    @Override
    public String getExternalSessionId() {
        return sessionId;
    }

    @Override
    public String getSessionSource() {
        return PlatformProvider.RAGFLOW.getCode();
    }
}
```

# 4. 前端实现

目标： 统一消费流式输出  

## 4.1.  startSSE：前端 SSE 流的入口  

```tsx
const { stream, loading: isLoading, cancel } = useHookFetch({
  request: send,
  onError: (err) => {
    console.warn('错误拦截', err);
  },
});
async function startSSE(submitResult) {
  const { isValid, handler, chatContent } = validateChat(submitResult);
  if (!isValid || !handler) return;

  senderRef.value?.clear();
  addMessage(chatContent, true);     // 用户消息
  addMessage('', false);             // AI 空气泡

  const streamParams = handler.buildParams(chatContent, submitResult, route);

  if (streamParams.stream) {
    for await (const chunk of stream(streamParams)) {
      handleDataChunk(chunk.result);
    }
  } else {
    const rs = await send(streamParams);
    handleDataChunk(JSON.parse(rs));
  }
}
```

## 4.2.  categoryHandlers：前端策略模式  

```typescript
export interface CategoryHandler {
  validate?: (chatContent: string, submitResult: SubmitResult) => boolean;
  buildParams: (chatContent: string, submitResult: SubmitResult, route: ReturnType<typeof useRoute>) => SendDTO;
  afterSend?: () => void;
}

 export const categoryHandlers: Record<string, CategoryHandler> = {
   [ChatModelCategoryConstants.ESG_FAQS]: {
    validate(chatContent, submitResult) {
      const processId = submitResult.inputTags.processId[0];
      if (!isString(processId)) {
        ElMessage.error(`输入内容有误，流程编号不能为空`);
        return false;
      }
      const match = chatContent.match(/问题：(.*)$/);
      if (!match || !match[1].trim()) {
        ElMessage.error(`输入内容有误，问题不能为空`);
        return false;
      }
      return true;
    },
    buildParams(chatContent, submitResult, route) {
      const baseParams = getBaseParams(route);
      return {
        ...baseParams,
        stream: true,
        processId: submitResult.inputTags.processId[0],
        question: chatContent,
      };
    },
    afterSend: undefined, // 依赖UI层的senderRef, 先留空，UI中再注入
  },
    [ChatModelCategoryConstants.RAGFLOW_CHAT]: {
    buildParams(chatContent, _, route) {
      const baseParams = getBaseParams(route);
      const request: ChatRequest = {
        ...baseParams,
        stream: true,
        question: chatContent,
      };
      return request;
    },
  }
 }}
```

## 4.3.  handleDataChunk :  统一解析不同模型流  

```typescript
function handleDataChunk(chunk: AnyObject) {
  try {
    const reasoningChunk = chunk.choices?.[0].delta.reasoning_content;
    if (reasoningChunk) {
      // 开始思考链状态
      bubbleItems.value[bubbleItems.value.length - 1].thinkingStatus = 'thinking';
      bubbleItems.value[bubbleItems.value.length - 1].loading = true;
      bubbleItems.value[bubbleItems.value.length - 1].thinlCollapse = true;
      if (bubbleItems.value.length) {
        bubbleItems.value[bubbleItems.value.length - 1].reasoning_content += reasoningChunk;
      }
    }

    // 另一种思考中形式，content中有 <think></think> 的格式
    // 一开始匹配到 <think> 开始，匹配到 </think> 结束，并处理标签中的内容为思考内容
    const parsedChunk = chunk.choices?.[0].delta.content || chunk.data?.content;
    if (parsedChunk) {
      const thinkStart = parsedChunk.includes('<think>');
      const thinkEnd = parsedChunk.includes('</think>');
      if (thinkStart) {
        isThinking = true;
      }
      if (thinkEnd) {
        isThinking = false;
      }
      if (isThinking) {
        // 开始思考链状态
        bubbleItems.value[bubbleItems.value.length - 1].thinkingStatus = 'thinking';
        bubbleItems.value[bubbleItems.value.length - 1].loading = true;
        bubbleItems.value[bubbleItems.value.length - 1].thinlCollapse = true;
        if (bubbleItems.value.length) {
          bubbleItems.value[bubbleItems.value.length - 1].reasoning_content += parsedChunk
            .replace('<think>', '')
            .replace('</think>', '');
        }
      }
      else {
        // 结束 思考链状态
        bubbleItems.value[bubbleItems.value.length - 1].thinkingStatus = 'end';
        bubbleItems.value[bubbleItems.value.length - 1].loading = false;
        if (bubbleItems.value.length) {
          bubbleItems.value[bubbleItems.value.length - 1].content += parsedChunk;
        }
      }
    }
  }
  catch (err) {
    // 这里如果使用了中断，会有报错，可以忽略不管
    console.error('解析数据时出错:', err);
  }
}
```
