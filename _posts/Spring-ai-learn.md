# Spring AI 学习总结：从基础到实战的完整探索

## 前言

在这段时间里，我深入学习了 Spring AI 框架，通过四个不同的演示项目逐步掌握了 Spring AI 的核心概念和实际应用。本文将对我的学习过程进行系统性总结，分享各个模块的技术要点和实践经验。

## 项目概览

整个学习项目采用 Maven 多模块结构，包含以下四个核心模块：

1. **spring-ai-demo**: Spring AI 基础功能演示（OpenAI/DeepSeek）
2. **spring-ollama-demo**: Ollama 本地模型集成
3. **spring-alibaba-demo**: 阿里通义千问集成
4. **spring-chat-bot**: 完整的聊天机器人应用

技术栈：
- Java 17
- Spring Boot 3.5.3
- Spring AI 1.0.0-M6
- Maven 多模块管理

## 一、Spring AI 基础概念

### 1.1 核心组件

Spring AI 提供了统一的 API 来访问不同的大语言模型，主要组件包括：

- **ChatClient**: 高级客户端，提供链式调用 API
- **ChatModel**: 底层聊天模型接口
- **Advisor**: 增强器，用于添加日志、记忆等功能
- **ChatMemory**: 对话记忆管理

### 1.2 依赖配置

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0-M6</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 二、模块详解

### 2.1 spring-ai-demo：OpenAI/DeepSeek 集成

#### 核心功能

**1. 基础聊天功能**

```java
@RestController
@RequestMapping("/chat")
public class ChatController {
    
    @Autowired
    public ChatClient chatClient;

    @GetMapping("/call")
    public String generation(String userInput){
        return this.chatClient.prompt()
                .user(userInput)
                .advisors(new SimpleLoggerAdvisor())
                .call()
                .content();
    }
```

**代码详解：**
- `prompt()`: 创建提示词构建器
- `.user()`: 设置用户消息
- `.advisors()`: 添加增强器
- `.call()`: 执行同步调用
- `.content()`: 提取响应文本

**2. 结构化输出**

```java
record Recipe(String dish, List<String> ingredients){}

@GetMapping("/entity")
public String entity(String userInput){
    Recipe recipe = this.chatClient.prompt()
            .user(String.format("请帮我生成%s的食谱", userInput))
            .call()
            .entity(Recipe.class);
    
    return recipe.toString();
}
```

**代码详解：**
- `record`: Java 14+ 引入的记录类型
- `.entity(Recipe.class)`: Spring AI 会自动要求 AI 以 JSON 格式输出，然后反序列化

**3. 流式响应**

```java
@GetMapping(value = "/stream", produces = "text/html;charset=utf-8")
public Flux<String> stream(String userInput){
    return this.chatClient.prompt()
            .user(userInput)
            .stream()
            .content();
}
```

**代码详解：**
- `Flux<String>`: Reactor 响应式编程中的异步序列
- `.stream()`: 切换到流式模式，逐块返回
- `produces = "text/html;charset=utf-8"`: 设置响应头，防止中文乱码

#### 配置示例

```yaml
spring:
  application:
    name: spring-ai-demo        # 应用名称
  ai:
    openai:
      api-key: sk-f72bad07461c4f7297ade171301538be  # DeepSeek API 密钥
      base-url: https://api.deepseek.com           # DeepSeek API 地址（兼容 OpenAI 接口）
      chat:
        options:
          kind: chat                               # 聊天类型
          model: deepseek-chat                     # 使用的模型名称
          temperature: 0.7                         # 温度参数：0-1之间，越高越随机，越低越确定
logging:
  level:
    org.springframework.ai.chat.client.advisor: debug  # 开启 Advisor 的调试日志
```

**关键点：**
- **API 兼容性**: DeepSeek 提供了与 OpenAI 兼容的 API 接口，所以可以使用 `spring-ai-openai-spring-boot-starter` 依赖
- **同步 vs 流式**: `.call()` 是同步阻塞调用，`.stream()` 是异步流式调用
- **结构化输出**: `.entity()` 方法让 AI 返回结构化数据，适合需要解析的场景
- **Temperature 参数**: 控制输出的随机性，0.7 是平衡创造性和一致性的常用值

### 2.2 spring-ollama-demo：本地模型集成

#### 核心优势

Ollama 是一个开源工具，允许在本地运行大语言模型：
- **数据隐私保护**: 所有数据都在本地处理，不会上传到云端
- **无网络依赖**: 不需要互联网连接即可使用
- **低成本部署**: 无需支付 API 调用费用
- **模型选择自由**: 可以运行各种开源模型（Llama、Mistral、DeepSeek 等）

#### 实现代码

```java
@RestController
@RequestMapping("/ollama")
public class OllamaController {
    
    @Autowired
    private OllamaChatModel ollamaChatModel;

    @RequestMapping("/chat")
    public String chat(String message){
        return ollamaChatModel.call(message);
    }

    @RequestMapping(value = "/stream", produces = "text/html;charset=utf-8")
    public Flux<String> stream(String message){
        return ollamaChatModel.stream(message);
    }
}
```

**代码详解：**
- `OllamaChatModel`: Spring AI 为 Ollama 提供的专用模型实现
- `.call(message)`: 简化的同步调用方式
- `.stream(message)`: 简化的流式调用方式

#### 配置

```yaml
server:
  port: 8081                      # 服务端口号，避免与其他模块冲突

spring:
  application:
    name: spring-ollama-demo      # 应用名称
  ai:
    ollama:
      base-url: http://localhost:11434  # Ollama 服务的地址（默认端口 11434）
      chat:
        model: deepseek-r1:1.5b   # 使用的模型名称和版本
                                  # deepseek-r1 是模型名，1.5b 表示 15 亿参数版本

logging:
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    # 自定义日志格式：时间 [线程] 级别 类名 - 消息
```

**使用前准备：**
1. 安装 Ollama: 从 https://ollama.ai 下载安装
2. 拉取模型: 在终端运行 `ollama pull deepseek-r1:1.5b`
3. 启动 Ollama 服务: 运行 `ollama serve`
4. 验证服务: 访问 http://localhost:11434 应该能看到 "Ollama is running"

**学习收获：**
- **本地 vs 云端**: 理解了本地模型与云端 API 的差异（延迟、成本、隐私）
- **Ollama 管理**: 掌握了 Ollama 的安装、模型拉取、服务启动等操作流程
- **模型切换**: 学会了如何通过配置快速切换不同的模型提供商
- **简化 API**: Ollama 可以直接使用 ChatModel，比 ChatClient 更简洁

### 2.3 spring-alibaba-demo：通义千问集成

#### 特色功能

阿里云 DashScope（灵积）提供了丰富的中文模型能力，特别适合中文场景：

```java
@RestController
@RequestMapping("/ali")
public class AliController {
    
    private static final String DEFAULT_PROMPT = "你是⼀个博学的智能聊天助⼿, 请根据⽤⼾提问回答！";
    
    private final ChatClient dashScopeChatClient;

    public AliController(ChatClient.Builder chatClientBuilder){
        this.dashScopeChatClient = chatClientBuilder
                .defaultSystem(DEFAULT_PROMPT)
                .defaultAdvisors(new SimpleLoggerAdvisor())
                .build();
    }

    @GetMapping("/chat")
    public String chat(String message){
        return dashScopeChatClient.prompt(message)
                .call()
                .content();
    }

    public Flux<String> flux(String message){
        Flux<String> output = dashScopeChatClient.prompt()
                .user(message)
                .stream()
                .content();
        return output;
    }
}
```

**代码详解：**
- **构造函数注入**: 相比字段注入，构造函数注入保证依赖不可变性，便于单元测试
- **ChatClient.Builder**: 建造者模式，允许链式配置
- **defaultSystem**: 设置系统级提示词，影响 AI 的所有回复风格
- **defaultAdvisors**: 设置默认增强器，对所有请求生效

#### 配置

```yaml
server:
  port: 8082                      # 服务端口号，避免冲突

spring:
  application:
    name: spring-alibaba-demo     # 应用名称
  ai:
    dashscope:                    # 阿里云 DashScope 配置
      api-key: sk-3696800b571e4f2ab6dbc100bf19ccb0  # API 密钥
      chat:
        model: qwen-turbo         # 使用的模型：通义千问 Turbo 版本
                                  # 其他可选：qwen-plus, qwen-max 等

logging:
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
```

**关键知识点：**
- **构造函数注入 Builder**: 使用 `ChatClient.Builder` 而非直接注入 `ChatClient`，可以在构建时进行定制
- **系统提示词**: `defaultSystem()` 设置的提示词会在每次对话开始时发送给 AI，用于设定角色和行为准则
- **厂商差异**: 不同厂商的配置前缀不同（openai、ollama、dashscope），但使用方式类似
- **依赖不同**: Alibaba 使用 `spring-ai-alibaba-starter`，而不是 `spring-ai-openai-spring-boot-starter`

### 2.4 spring-chat-bot：完整聊天机器人

这是最复杂的模块，实现了完整的聊天机器人后端应用。

#### 架构设计

```
┌─────────────────────────────────────────┐
│     Controller Layer (ChatController)   │  ← 接收 HTTP 请求，处理业务逻辑
└──────────────┬──────────────────────────┘
               │ 调用
               ↓
┌─────────────────────────────────────────┐
│  Service Layer (ChatClient + Advisors)  │  ← AI 对话核心，包含记忆管理
└──────────────┬──────────────────────────┘
               │ 保存/读取
               ↓
┌─────────────────────────────────────────┐
│ Repository Layer (MemoryChatHistoryRepo)│  ← 会话历史存储（内存）
└─────────────────────────────────────────┘
```

#### 核心功能实现

**1. 配置类：Bean 的定义和依赖注入**

```java
@Configuration                    // 标记这是一个配置类，Spring 会扫描其中的 @Bean 方法
public class CommonConfiguration {
    
    @Bean
    public ChatClient ollamaChatClient(OllamaChatModel model){
        return ChatClient
                .builder(model)
                .defaultSystem("你叫小萌，拥有十年的Java开发经验，是一个十分出色的开发工程师，专注于解决大学生对于Java项目的疑惑~")
                .defaultAdvisors(
                    new SimpleLoggerAdvisor(),
                    new MessageChatMemoryAdvisor(chatMemory())
                )
                .build();
    }

    @Bean
    public ChatClient deepseekChatClient(OpenAiChatModel chatModel, ChatMemory chatMemory){
        return ChatClient.builder(chatModel)
                .defaultAdvisors(
                    new SimpleLoggerAdvisor(),
                    new MessageChatMemoryAdvisor(chatMemory())
                )
                .defaultSystem("你叫小萌，拥有十年的Java开发经验，是一个十分出色的开发工程师，专注于解决大学生对于Java项目的疑惑~")
                .build();
    }

    @Bean
    public ChatMemory chatMemory(){
        return new InMemoryChatMemory();
    }
}
```

**代码详解：**
- `@Configuration`: 告诉 Spring 这个类包含 Bean 定义
- `@Bean`: 将方法返回值注册到 Spring 容器中，其他组件可以注入使用
- **依赖注入**: Spring 会自动识别方法参数类型，从容器中注入相应的 Bean
- **MessageChatMemoryAdvisor**: 关键的增强器，它会自动：
  - 在发送请求前，从 ChatMemory 中获取历史对话
  - 将历史对话添加到提示词中
  - 在收到响应后，将新的对话保存到 ChatMemory
- **为什么需要两个 ChatClient**: 可以同时支持本地 Ollama 和云端 DeepSeek，根据需要选择

**2. 会话历史存储：Repository 层**

```java
@Repository                     // 标记这是一个数据访问层组件，Spring 会自动扫描并注册为 Bean
public class MemoryChatHistoryRepository implements ChatHistoryRepository{
    
    private final Map<String, String> chatHistory = new LinkedHashMap<>();
    
    @Override
    public void save(String prompt, String chatId) {
        chatHistory.put(chatId, prompt);
    }

    @Override
    public List<ChatInfo> getChats() {
        return chatHistory.entrySet().stream()
                .map(entry -> new ChatInfo(entry.getValue(), entry.getKey()))
                .collect(Collectors.toList());
    }

    @Override
    public void clearChatId(String chatId) {
        chatHistory.remove(chatId);
    }
}
```

**代码详解：**
- `@Repository`: 专门用于数据访问层的注解，便于异常转换和事务管理
- `LinkedHashMap`: 相比普通 HashMap，它能保持插入顺序
- **为什么只存第一条消息**: 作为会话标题，方便用户在侧边栏识别不同的对话
- **Stream API**: Java 8+ 的函数式编程特性，代码更简洁易读
- **接口实现**: 实现 `ChatHistoryRepository` 接口，方便未来替换为数据库实现

**3. 控制器层：处理 HTTP 请求**

```java
@Slf4j                        // Lombok 注解，自动生成 logger 对象（等同于 private static final Logger log = ...）
@RequestMapping("/chat")      // 基础路径：/chat
@RestController               // REST 控制器，所有方法返回值自动序列化为 JSON
public class ChatController {
    
    private final ChatClient chatClient;

    public ChatController(ChatClient deepseekChatClient) {
        this.chatClient = deepseekChatClient;
    }

    @Autowired
    private MemoryChatHistoryRepository memoryChatHistoryRepository;
    
    @Autowired
    private ChatMemory chatMemory;

    @RequestMapping(value = "/stream", produces = "text/html;charset=utf-8")
    public Flux<String> stream(String prompt, String chatId){
        log.info("prompt:{} , chatId:{}", prompt, chatId);
        
        memoryChatHistoryRepository.save(prompt, chatId);
        
        return chatClient.prompt()
                .user(prompt)
                .advisors(spec -> spec.param(
                    AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY,
                    chatId
                ))
                .stream()
                .content();
    }

    @RequestMapping("/getChatIds")
    public List<ChatInfo> getChatIds(){
        return memoryChatHistoryRepository.getChats();
    }

    @RequestMapping("/getChatHistory")
    public List<MessageVO> getChatHistory(String chatId){
        log.info("获取会话记录，chatId:{}", chatId);
        
        List<Message> messages = chatMemory.get(chatId, 20);
        
        return messages.stream()
                .map(MessageVO::new)
                .collect(Collectors.toList());
    }

    @RequestMapping("/deleteChat")
    public Boolean deleteChat(String chatId){
        log.info("删除会话，chatId:{}", chatId);
        try{
            memoryChatHistoryRepository.clearChatId(chatId);
            chatMemory.clear(chatId);
        }catch (Exception e){
            log.info("删除会话失败，chatId:{}", chatId);
            return false;
        }
        return true;
    }
}
```

**代码详解：**
- **@Slf4j**: Lombok 提供的注解，自动生成 SLF4J Logger
- **构造函数注入**: 比字段注入更推荐，保证不可变性和可测试性
- **AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY**: 常量，值为 "chat_memory_conversation_id"，用于指定会话 ID
- **两次删除**: 
  - `memoryChatHistoryRepository.clearChatId()`: 删除会话元数据（标题）
  - `chatMemory.clear()`: 删除实际的对话历史

**4. 实体类：数据传输对象**

```java
@Data
public class ChatInfo {
    private String title;
    private String chatId;
    
    public ChatInfo(String title, String chatId) {
        this.title = title == null ? "无标题" : 
                     title.length() > 15 ? title.substring(0, 15) : title;
        this.chatId = chatId;
    }
}
```

```java
@Data
public class MessageVO {
    String role;
    String content;

    public MessageVO(Message message){
        switch(message.getMessageType()){
            case USER -> { this.role = "user"; break; }
            case ASSISTANT -> { this.role = "assistant"; break; }
            case SYSTEM -> { this.role = "system"; break; }
            case TOOL -> { this.role = "tool"; break; }
        }
        this.content = message.getText();
    }
}
```

**代码详解：**
- **@Data**: Lombok 注解，自动生成 getter、setter、toString 等方法
- **三元运算符**: `condition ? value1 : value2`，if-else 的简洁写法
- **substring(0, 15)**: 截取前 15 个字符，防止标题过长
- **switch 表达式**: Java 14+ 新特性
- **getMessageType()**: 返回 MessageType 枚举，表示消息角色

## 三、核心技术要点总结

### 3.1 ChatClient vs ChatModel

| 特性 | ChatClient | ChatModel |
|------|-----------|----------|
| API 级别 | 高级（封装了更多功能） | 低级（更接近原始 API） |
| 链式调用 | ✅ 支持 `.prompt().user().call().content()` | ❌ 直接调用 `.call(message)` |
| Advisor 支持 | ✅ 完整支持日志、记忆等增强器 | ❌ 有限或不支持 |
| 灵活性 | 中等（有固定的构建流程） | 高（可以更精细地控制） |
| 推荐使用场景 | 大多数应用场景 | 需要精细控制或简单调用时 |
| 依赖注入方式 | 通常注入 Builder 进行定制 | 直接注入模型实例 |

**选择建议：**
- 如果需要对话记忆、日志记录等功能，使用 **ChatClient**
- 如果只是简单的单次调用，使用 **ChatModel** 更简洁
- ChatClient 底层也是调用 ChatModel，只是增加了额外的功能层

### 3.2 Advisor 机制

Advisor 是 Spring AI 的强大扩展机制，类似于 Spring 的拦截器或中间件。

**内置 Advisor 类型：**

1. **SimpleLoggerAdvisor**: 
   - 功能：记录完整的请求和响应日志
   - 用途：调试和监控
   - 配置：需要在 `application.yml` 中开启 debug 日志级别

2. **MessageChatMemoryAdvisor**: 
   - 功能：自动管理对话历史
   - 工作流程：
     ```
     请求前：从 ChatMemory 获取历史 → 添加到提示词
     请求后：将新的对话保存到 ChatMemory
     ```
   - 关键参数：`CHAT_MEMORY_CONVERSATION_ID_KEY` 指定会话 ID

3. **自定义 Advisor**: 
   - 可以实现 `Advisor` 接口创建自定义增强器
   - 常见用途：权限检查、速率限制、内容过滤等

**使用示例：**

```java
// 单个 Advisor
.advisors(new SimpleLoggerAdvisor())

// 多个 Advisor
.defaultAdvisors(
    new SimpleLoggerAdvisor(),              // 日志记录
    new MessageChatMemoryAdvisor(chatMemory) // 对话记忆
)

// 动态参数（用于指定会话 ID）
.advisors(spec -> spec.param(
    AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, 
    chatId  // 传入具体的会话 ID
))
```

**工作原理：**
```
用户请求
  ↓
[Advisor 1: 日志记录] ← 记录请求信息
  ↓
[Advisor 2: 对话记忆] ← 加载历史对话
  ↓
ChatModel 调用 AI
  ↓
[Advisor 2: 对话记忆] ← 保存新对话
  ↓
[Advisor 1: 日志记录] ← 记录响应信息
  ↓
返回给用户
```

### 3.3 流式响应处理

流式响应（Streaming）是一种逐步返回结果的技术，相比传统的一次性返回有明显优势。

**流式响应的优势：**
- **更好的用户体验**: 用户可以立即看到部分结果，而不是等待完整响应
- **降低感知延迟**: 即使总时间相同，逐字显示让用户感觉更快
- **适合长文本生成**: 对于长篇回答，用户可以边看边等
- **减少内存占用**: 不需要在服务器端缓存完整响应

**后端实现代码：**

```java
@GetMapping(value = "/stream", produces = "text/html;charset=utf-8")
public Flux<String> stream(String message){
    return chatClient.prompt()
            .user(message)
            .stream()      // 关键：切换到流式模式
            .content();    // 返回 Flux<String>
}
```

**关键点：**
- `Flux<String>`: Reactor 响应式编程中的异步序列，可以发射 0-N 个元素
- `.stream()`: 告诉 Spring AI 使用流式 API，AI 会逐块（token by token）返回结果
- `produces = "text/html;charset=utf-8"`: 设置响应头，确保中文正确显示
- **注意**: 前端需要使用 ReadableStream 来消费这个流，但作为后端开发，我们只需关注如何正确地返回 Flux

**对比：同步 vs 流式**

| 特性 | 同步调用 (.call()) | 流式调用 (.stream()) |
|------|-------------------|---------------------|
| 返回类型 | `String` | `Flux<String>` |
| 等待时间 | 等待完整响应 | 立即开始接收数据 |
| 用户体验 | 可能感觉慢 | 即时反馈，体验好 |
| 实现复杂度 | 简单 | 较复杂（需处理流） |
| 适用场景 | 短文本、简单查询 | 长文本、创意生成 |

### 3.4 对话记忆管理

对话记忆（Chat Memory）是让 AI 能够记住上下文的关键机制，实现有状态的对话。

**为什么需要对话记忆？**

大语言模型本质上是无状态的，每次请求都是独立的。如果没有记忆：
```
用户: "我叫小明"
AI: "你好，小明！"

用户: "我多大了？"  ← AI 不知道"我"指的是谁
AI: "我不知道你多大了"
```

有了记忆：
```
用户: "我叫小明"
AI: "你好，小明！"
[系统保存: {role: "user", content: "我叫小明"}]
[系统保存: {role: "assistant", content: "你好，小明！"}]

用户: "我多大了？"
[系统加载历史] → AI 看到完整对话
AI: "你之前没有告诉我你多大了"
```

**Spring AI 提供的记忆实现：**

1. **InMemoryChatMemory**: 
   - 存储位置：JVM 内存（HashMap）
   - 优点：快速、简单、无需配置
   - 缺点：应用重启后数据丢失、不支持分布式
   - 适用场景：演示、开发、小规模应用

2. **JdbcChatMemory**: 
   - 存储位置：关系型数据库（MySQL、PostgreSQL 等）
   - 优点：持久化、支持事务
   - 缺点：需要配置数据库、性能相对较低
   - 适用场景：生产环境、需要持久化的场景

3. **RedisChatMemory**: 
   - 存储位置：Redis
   - 优点：高性能、支持分布式、可设置过期时间
   - 缺点：需要 Redis 服务
   - 适用场景：高并发、分布式部署

**使用示例：**

```java
// 1. 创建 ChatMemory Bean
@Bean
public ChatMemory chatMemory(){
    return new InMemoryChatMemory();
}

// 2. 在 ChatClient 中添加 MessageChatMemoryAdvisor
@Bean
public ChatClient chatClient(OpenAiChatModel model, ChatMemory chatMemory){
    return ChatClient.builder(model)
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(chatMemory)  // 添加记忆增强器
            )
            .build();
}

// 3. 在调用时指定会话 ID
chatClient.prompt()
    .user(message)
    .advisors(spec -> spec.param(
        AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, 
        chatId  // 每个会话有唯一的 ID
    ))
    .stream()
    .content();
```

**工作流程详解：**

```
第一次请求 (chatId = "session_001"):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. 用户输入: "我叫小明"
2. MessageChatMemoryAdvisor 检查 session_001 的记忆 → 空
3. 发送提示词: [User: 我叫小明]
4. AI 回复: "你好，小明！"
5. MessageChatMemoryAdvisor 保存:
   - User: 我叫小明
   - Assistant: 你好，小明！

第二次请求 (chatId = "session_001"):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. 用户输入: "我今年20岁"
2. MessageChatMemoryAdvisor 加载 session_001 的记忆:
   - User: 我叫小明
   - Assistant: 你好，小明！
3. 发送提示词: 
   [
     User: 我叫小明,
     Assistant: 你好，小明！,
     User: 我今年20岁
   ]
4. AI 回复: "好的，小明，你今年20岁"
5. 保存新的对话到记忆

第三次请求 (chatId = "session_002"):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. 用户输入: "我叫小红"
2. MessageChatMemoryAdvisor 检查 session_002 的记忆 → 空（新会话）
3. 发送提示词: [User: 我叫小红]
4. AI 不会知道 session_001 的内容
```

**关键参数说明：**

```java
AbstractChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY
```
- 这是一个常量，值为 `"chat_memory_conversation_id"`
- 用于告诉 Advisor 使用哪个会话的记忆
- 不同的 chatId 对应不同的对话历史
- 如果不指定，可能所有用户共享同一个记忆（错误！）

**最佳实践：**

1. **会话 ID 生成**: 使用 UUID 或时间戳 + 随机数
   ```java
   String chatId = "chat_" + UUID.randomUUID().toString();
   ```

2. **记忆清理**: 定期清理旧会话，避免内存泄漏
   ```java
   chatMemory.clear(chatId);  // 删除指定会话
   ```

3. **限制历史长度**: 获取最近 N 条消息，避免提示词过长
   ```java
   List<Message> messages = chatMemory.get(chatId, 20);  // 最近 20 条
   ```

4. **生产环境选择**: 
   - 小型应用：InMemoryChatMemory
   - 中型应用：JdbcChatMemory
   - 大型/分布式：RedisChatMemory

## 四、遇到的问题与解决方案

### 问题 1：API Key 配置安全

**问题**：API Key 硬编码在配置文件中存在安全风险。

**解决方案**：
- 使用环境变量
- 使用 Spring Cloud Config
- 使用 Vault 等密钥管理服务

### 问题 2：流式响应的字符编码

**问题**：中文显示乱码。

**解决方案**：
```java
@RequestMapping(value = "/stream", produces = "text/html;charset=utf-8")
```

### 问题 3：对话记忆的持久化

**问题**：InMemoryChatMemory 在应用重启后丢失数据。

**解决方案**：
- 实现基于数据库的 ChatMemory
- 定期备份内存数据
- 使用 Redis 等外部存储

## 五、最佳实践

### 5.1 项目结构规范

```
src/main/java/com/example/
├── configuration/    # 配置类
├── controller/       # 控制器
├── entity/          # 实体类
├── repository/      # 数据访问
└── service/         # 业务逻辑（推荐添加）
```

### 5.2 配置管理

1. 使用 `application.yml` 而非 `properties`
2. 敏感信息使用环境变量
3. 不同环境使用不同的 profile

### 5.3 错误处理

```java
try {
    // AI 调用
} catch (Exception e) {
    log.error("AI 调用失败", e);
    return "抱歉，服务暂时不可用";
}
```

### 5.4 性能优化

1. 复用 ChatClient 实例（单例）
2. 合理设置超时时间
3. 使用连接池
4. 缓存常用响应

## 六、学习收获

### 6.1 技术层面

1. **Spring AI 核心概念**：深入理解了 ChatClient、ChatModel、Advisor 等核心组件
2. **多模型集成**：掌握了 OpenAI、Ollama、DashScope 等多种模型的集成方法
3. **响应式编程**：熟悉了 Reactor 的 Flux 和流式处理
4. **对话管理**：学会了如何实现有状态的对话系统

### 6.2 工程实践

1. **模块化设计**：通过多模块项目组织代码
2. **配置管理**：合理使用 YAML 配置和环境变量
3. **RESTful API 设计**：遵循 REST 规范设计接口
4. **代码规范**：遵循 Spring Boot 最佳实践

### 6.3 问题解决能力

1. 学会了阅读官方文档和源码
2. 掌握了调试技巧（日志、断点）
3. 能够独立解决集成问题

## 七、未来展望

### 7.1 可以深入的方向

1. **RAG（检索增强生成）**：结合向量数据库实现知识库问答
2. **Function Calling**：让 AI 调用外部工具
3. **多模态支持**：图像、音频处理
4. **Agent 系统**：构建自主代理

### 7.2 生产化改进

1. **持久化存储**：使用数据库或 Redis 存储对话历史
2. **用户认证**：添加 JWT 或 OAuth2 认证
3. **限流保护**：防止 API 滥用
4. **监控告警**：集成 Prometheus + Grafana
5. **单元测试**：提高代码质量

### 7.3 性能优化

1. **缓存策略**：缓存常见问题的答案
2. **异步处理**：使用 @Async 提高并发能力
3. **负载均衡**：多实例部署
4. **CDN 加速**：静态资源优化

## 八、总结

通过这次系统的 Spring AI 学习，我从零开始构建了一个功能完整的聊天机器人应用。整个过程涵盖了：

- ✅ Spring AI 基础 API 的使用
- ✅ 多种大语言模型的集成（OpenAI、Ollama、DashScope）
- ✅ 流式响应的后端实现
- ✅ 对话记忆管理
- ✅ RESTful API 设计
- ✅ Spring Boot 最佳实践
- ✅ 配置管理和依赖注入

**技术栈**：Spring Boot 3.5.3 + Spring AI 1.0.0-M6 + Java 17
