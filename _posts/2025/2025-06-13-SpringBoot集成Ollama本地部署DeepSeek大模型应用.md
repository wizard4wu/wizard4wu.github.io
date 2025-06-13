---
title: 'SpringBoot集成Ollama本地部署DeepSeek大模型应用'
key: key-2026-06-13-SpringBoot+Ollama+DeepSeek
date: 2025-06-03 14:10:26
tags: ["SpringBoot", "AI"]
comment: true
footer: true
show_edit_on_github: false
pageview: true
lightbox: true
aside:
toc: true
show_subscribe: false
---
本文通过ollama本地部署`deepseek-r1:1.5b`模型，然后基于Langchain开源框架集成到SpringBoot项目中，完成本地大模型对话项目。

Langchain是一个开源框架，用于开发由语言模型驱动的应用程序框架。

它通过三个核心组件实现增强：

• 首先是 Compents“组件”，LLMs提供接口封装模板提示和信息检索索引；

• 其次是 Chains“链”，它将不同的组件组合起来解决特定的任务，比如在大量文本中查找信息；

• 最后是 Agents“代理”，它们使得LLMs能够与外部环境进行交互，例如通过API请求执行操作。

### 1.安装ollama

ollama[官网](https://ollama.com)

Ollama 是专门为 macOS/Linux/Windows 本地部署大模型设计的工具，支持 Apple Silicon（M1/M2/M3）原生加速。

```shell
curl -fsSL https://ollama.com/install.sh | sh

上面的命令在mac上运行提示ERROR: This script is intended to run on Linux only.
```

使用homebrew安装遇到了一些问题，然后乖乖的卸载后重新在官网下载安装包傻瓜式的安装。

安装后一定要使用`ollama server`命令启动，下面的截图是表示安装成功，然后运行大模型进行控制台的交互方式

下述使用`ollama run llama3`这个是ollama的模型，我使用的是deepseek模型，所以运行该命令`ollama run deepseek-r1:1.5b`, 下载后会自动部署(很傻瓜式的)。

![image-20250513131903570]({{"/assets/picture/2025/jun/image-20250513131903570.png" | absolute_url}})
或者使用`http://1ocalhost:11434`访问会出现`Ollama is running`的返回结果。

还可以在控制台中执行命令：

```java
curl http://localhost:11434/api/generate -d '{
  "model": "llama3",
  "prompt": "用 Java 写一个快速排序",
  "stream": false
}'
```
![image-20250513133441834]({{"/assets/picture/2025/jun/image-20250513133441834.png" | absolute_url}})

### 2.基于langchain4j集成SpringBoot

我的项目地址：[点此进入](https://github.com/wizard4wu/wizard-2.0/tree/main/MyLangChain4j)

LangChain4j 的目标是简化与 Java 应用程序集成大模型，LangChain4j的[官网](https://docs.langchain4j.dev/)

引入依赖：

```java
<dependencies>
    <!-- LangChain4J 核心库 -->
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j</artifactId>
        <version>1.0.0-rc1</version>
    </dependency>

    <!-- OpenAI 接入支持 OpenAI和Deepseek协议完全一致 -->
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-open-ai</artifactId>
        <version>1.0.0-rc1</version>
    </dependency>

    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-ollama</artifactId>
        <version>1.0.0-beta5</version>
    </dependency>
</dependencies>
```

```yaml
ollama:
  chat:
    base-url: http://localhost:11434   //我的是本地部署的
    model-name: deepseek-r1:1.5b       //我选择的deepseek模型，比较小巧
```

#### 2.1对话响应

```java

    //配置文本输出model对象
    @Bean
    public ChatModel chatModel() {
        ChatModel model = OllamaChatModel
                .builder()
                .baseUrl(ollamaChatProperties.getBaseUrl())
                .modelName(ollamaChatProperties.getModelName())
                .build();
        return model;
    }

    @RequestMapping("/chat/ollama/string")
    public String ollamaChatModel(@RequestParam("message") String inputValue) {
        final String resultString = chatModel.chat(inputValue);
        return resultString;
    }
```

很简单，这样就可以调用API进行访问了，在响应的数据中会有大模型的回答。

#### 2.2流式响应

```java
    //配置流式输出model对象
    @Bean
    public StreamingChatModel streamChatModel() {
        StreamingChatModel streamingChatModel = OllamaStreamingChatModel
                .builder()
                .baseUrl(ollamaChatProperties.getBaseUrl())
                .modelName(ollamaChatProperties.getModelName())
                .build();
        return streamingChatModel;
    }

   @RequestMapping(value = "/chat/ollama/stream", produces = "text/stream;charset=utf-8") //这里要配置utf-8，不然流式输出会乱码
    public Flux<String> streamOllamaChatModel(@RequestParam("message") String inputValue) {
        return Flux.create(sink -> {
            streamingChatModel.chat(inputValue, new StreamingChatResponseHandler() {
                @Override
                public void onPartialResponse(String partialResponse) {
                    sink.next(partialResponse);
                }

                @Override
                public void onCompleteResponse(ChatResponse completeResponse) {
                    sink.complete();
                    log.info("complete response");
                }
                @Override
                public void onError(Throwable error) {
                    sink.error(error);
                    log.error("error: ", error);
                }
            });
        });
    }
```

#### 2.3 对话记忆

对话记忆就是在使用大模型时，让大模型感知用户之前输入的内容。比如我们在使用chatGPT时对话的窗口会出现上下文的信息关联，主要原理就是将用户的数据存储，然后通过动态代理的方式，将数据一起喂给大模型，然后返回有信息关联的结果。

```java
//定义对话的接口
public interface Assistant{
        String chat(String text);

        TokenStream stream(String text);
}

//构建助手对象，该对象是一个代理对象
@Bean
public Assistant assistant(ChatModel chatModel, StreamingChatModel streamChatModel, ToolService toolService){
        //创建chatMemory对象，这表示会存储用户最近的10条数据，默认是存在JVM内存中
        ChatMemory chatMemory = MessageWindowChatMemory.withMaxMessages(10);
        Assistant assistant = AiServices
                .builder(Assistant.class)
                .chatModel(chatModel)
                .tools(toolService)  //toolService是为了支持function call的，后面会有讲到
                .streamingChatModel(streamChatModel)
                .chatMemory(chatMemory)
                .build();
        return assistant;
    }

//controller层进行访问
@RequestMapping(value = "/chat/ollama/memory/string")
public String ollamaChatModelMemory(@RequestParam("message") String inputValue) {
    return assistant.chat(inputValue);
}
```

```Java

```
对比一下对话记忆的显著作用:
![image-20250611140411103]({{"/assets/picture/2025/jun/image-20250611140411103.png" | absolute_url}})
![image-20250611140436354]({{"/assets/picture/2025/jun/image-20250611140436354.png" | absolute_url}})

#### 2.4对话记忆隔离

```java
    //进行对话隔离,因此要配置一个id
    //@UserMessage注解是为了告诉大模型哪一个参数是来自用户的消息。当参数大于一个是就需要用到这个注解
public interface AssistantUnique{
    String chat(@MemoryId String memoryId, @UserMessage String text);

    TokenStream stream(@MemoryId String memoryId, @UserMessage String text);
}

//构建对话隔离对象
@Bean
public AssistantUnique assistantUnique(ChatModel chatModel, StreamingChatModel streamChatModel){
    AssistantUnique assistantUnique = AiServices
           .builder(AssistantUnique.class)
           .chatModel(chatModel)
           .streamingChatModel(streamChatModel)
           .chatMemoryProvider(memoryId -> MessageWindowChatMemory.builder().maxMessages(10).id(memoryId).build())  //chatMemoryProvider会将memoryId传入
           .build();
    return assistantUnique;
}

//然后访问类似上述的方式
    @RequestMapping(value = "/ollama/memory/{id}/string")
    public String ollamaChatModelMemory(@PathVariable("id") String id, @RequestParam("message") String inputValue) {
        return assistantUnique.chat(id, inputValue);
    }
```

#### 2.5对话记忆持久化

这里就是将用户之前的输入数据进行存储，然后将这些数据喂给大模型。完成持久化的主要实现`ChatMemoryStore`这个接口就可以，然后注入这个持久化对象到Assistant对象中。代码如下：

```java
//因为是三方包，所以我一般都会加一个中间接口，我这里使用的Redis的方式来持久化
//如果是Mysql的话你可以加一个DBChatMemoryStore作为中间层
public interface CacheChatMemoryStore extends ChatMemoryStore {

}
//使用Redis来对对话进行持久化
@Service
@Slf4j
public class RedisChatMemoryStore implements CacheChatMemoryStore{

    @Autowired
    private StringRedisTemplate redisTemplate;
    @Override
    public List<ChatMessage> getMessages(Object memoryId) {
        String value = redisTemplate.opsForValue().get((String) memoryId);
        if(value == null) {
            return Collections.emptyList();
        }
        List<ChatMessage> chatMessageList = ChatMessageDeserializer.messagesFromJson(value);
        return chatMessageList;
    }

    @Override
    public void updateMessages(Object memoryId, List<ChatMessage> messages) {
        if(CollectionUtils.isEmpty(messages)) {
            return;
        }
        redisTemplate.opsForValue().set((String)memoryId, ChatMessageSerializer.messagesToJson(messages));

    }
    @Override
    public void deleteMessages(Object memoryId) {
        redisTemplate.delete((String)memoryId);
    }
}

//配置持久化的model助手对象
@Bean
public AssistantUniqueWithPersistence assistantUniqueWithPersistence(ChatModel chatModel, StreamingChatModel streamingChatModel, ChatMemoryStore chatMemoryStore) {  //我们要将这个chatMemoryStore注入到助手对象中
        AssistantUniqueWithPersistence assistantUnique = AiServices
                .builder(AssistantUniqueWithPersistence.class)
                .chatModel(chatModel)
                .streamingChatModel(streamingChatModel)
                .chatMemoryProvider(memoryId -> MessageWindowChatMemory.builder().id(memoryId).maxMessages(10).chatMemoryStore(chatMemoryStore).build())
                .build();
        return assistantUnique;
    }

//定义API，用assistantUniqueWithPersistence来调用，你会发现redis中会有对应的数据，不过是乱码的，哈哈哈哈。
@RequestMapping(value = "/ollama/memory/{id}/string")
public String ollamaChatModelMemory(@PathVariable("id") String id, @RequestParam("message") String inputValue) {
        return assistantUniqueWithPersistence.chat(id, inputValue);
    }
```

Note: 在持久化的过程中，对于ChatMessage对象的序列化和反序列化我搞了好久，我一开始使用自己的定义的序列化化方式，然后你会发现ChatMessage是个接口，并且它的所有实现类都是没有个get/set方法，然后查文档，原来它内部就有对应的序列化工具`ChatMessageDeserializer/ChatMessageSerializer`。

#### 2.5 Function Call

```java
//定义function call的接口
public interface ToolService {
    @Tool("某地区的天气")
    String getWeather(@P("地区") String address, @P("天气") String weather);
}

@Service
public class ToolServiceImpl implements ToolService {
    @Override
    public String getWeather(String address, String weather){
        //todo add your business code
        System.out.println(weather + " -- " +address);
        return "浙江杭州";
    }
}

@Bean
public Assistant assistant(ChatModel chatModel, StreamingChatModel streamChatModel, ToolService toolService){
        //创建chatMemory对象，这表示会存储用户最近的10条数据，默认是存在JVM内存中
        ChatMemory chatMemory = MessageWindowChatMemory.withMaxMessages(10);
        Assistant assistant = AiServices
                .builder(Assistant.class)
                .chatModel(chatModel)
                .tools(toolService)  //toolService是为了支持function call的，将其注入到对象中
                .streamingChatModel(streamChatModel)
                .chatMemory(chatMemory)
                .build();
}
```

因为我的模型不支持function call，所以在访问的时候会报错：

`dev.langchain4j.exception.HttpException: {"error":"registry.ollama.ai/library/deepseek-r1:1.5b does not support tools"}`

### 3.RAG检索增强生成

向量数据库是用于做相似性搜索的

数据需要调用向量大模型转换成向量后存入向量数据库中

根据用户的问题通过向量大模型转换成向量再到向量数据库中获取相似性的答案，然后将答案传入大语言模型中，大语言模型结合数据进行思考

四个步骤：

- 文档的收集和检索；
- 向量转换和存储；
- 文档的过滤和检索；
- 查询增强和关联；





