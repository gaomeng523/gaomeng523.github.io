# Spring AI 学习总结：从入门到实战应用

## 📚 前言

最近几天，我系统地学习了 Spring AI 框架，并通过实际编码掌握了如何将大语言模型（LLM）集成到企业级 Java 应用中。本文是我这段时间的学习总结和实践经验分享，希望能帮助同样对 Spring AI 感兴趣的开发者快速上手。

---

## 🎯 一、什么是 Spring AI？

Spring AI 是 Spring 生态系统中的一个新项目，旨在简化在 Java 应用中集成人工智能功能的过程。它提供了统一的 API 来访问各种 AI 模型提供商（如阿里云通义千问、百度文心一言等），让开发者能够专注于业务逻辑而非底层技术细节。

### 核心优势

- **统一抽象**：屏蔽不同 AI 平台的差异
- **Spring 生态集成**：与 Spring Boot 无缝衔接
- **企业级特性**：支持工具调用、RAG、Agent 等高级功能
- **快速迭代**：紧跟 AI 技术发展步伐

---

## 🏗️ 二、项目架构概览

在学习过程中，我创建了两个示例项目：

1. **spring-alibaba-demo**：基于阿里云通义千问的完整示例
2. **spring-qianfan-demo**：基于百度文心一言的基础示例

### 项目结构

```
Spring-ai-project2/
├── spring-alibaba-demo/          # 阿里云通义千问示例
│   ├── src/main/java/com/example/demo/
│   │   ├── AlibabaApplicationDemo.java      # 启动类
│   │   ├── controller/
│   │   │   ├── ChatController.java          # 聊天控制器
│   │   │   └── ChatModelController.java     # 模型控制器
│   │   ├── tool/
│   │   │   ├── DateTimeTools.java           # 日期时间工具
│   │   │   └── WeatherTools.java            # 天气查询工具
│   │   └── config/
│   │       └── ToolConfig.java              # 工具配置
│   └── src/main/resources/
│       └── application.yml                  # 配置文件
│
└── spring-qianfan-demo/            # 百度文心一言示例
    ├── src/main/java/com/exemple/demo/
    │   ├── QianfanApplication.java          # 启动类
    │   └── controller/
    │       └── ChatController.java          # 聊天控制器
    └── src/main/resources/
        └── application.yml                  # 配置文件
```

---

## 🔧 三、基础配置与实践

### 3.1 依赖管理

在 `pom.xml` 中引入 Spring AI 相关依赖：

```xml
<!-- Spring AI Alibaba Starter -->
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
    <version>1.0.0-M5.1</version>
</dependency>

<!-- Spring AI OpenAI Starter (用于百度千帆) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    <version>1.0.0-M5</version>
</dependency>
```

### 3.2 配置文件详解

#### 阿里云通义千问配置 (`application.yml`)

```yaml
spring:
  application:
    name: spring-alibaba-demo
  ai:
    dashscope:
      api-key: sk-3696800b571e4f2ab6dbc100bf19ccb0
      chat:
        options:
          model: qwen-turbo          # 指定使用的模型
          temperature: 0.7           # 控制回复的随机性（0-1）
```

**关键参数说明：**
- `api-key`：阿里云 DashScope API 密钥
- `model`：选择具体的模型版本（qwen-turbo、qwen-plus 等）
- `temperature`：温度参数，控制输出的创造性（0 最确定，1 最随机）

#### 百度文心一言配置

```yaml
spring:
  ai:
    openai:
      # 格式：AK|SK 中间竖线隔开
      api-key: "bce-v3/ALTAK-LYTDXb47v6rSn5Dxu9310/2378a24f1337d76794b|257bec5975ee28b833971"
      base-url: https://qianfan.baidubce.com
      chat:
        options:
          model: ernie-tiny-8k
          temperature: 0.7
```

**注意：** 百度千帆使用 OpenAI 兼容接口，因此可以使用 `spring-ai-openai` starter。

---

## 💬 四、基础聊天功能实现

### 4.1 最简单的聊天实现

在 `spring-qianfan-demo` 中，我们实现了最基础的聊天功能：

```java
@RestController
@RequestMapping("/chat")
public class ChatController {

    @Autowired
    private ChatModel chatModel;

    @GetMapping("/generate")
    public String generate(@RequestParam String message){
        return chatModel.call(message);
    }
}
```

**访问方式：**
```
GET http://localhost:8080/chat/generate?message=你好
```

这是最简单的用法，直接调用 `ChatModel` 的 `call()` 方法即可获取 AI 回复。

### 4.2 使用 ChatClient（推荐方式）

在 `spring-alibaba-demo` 中，我们使用了更强大的 `ChatClient`：

```java
@RestController
@RequestMapping("/chat")
public class ChatController {

    public ChatClient chatClient;

    public ChatController(DashScopeChatModel chatModel) {
        this.chatClient = ChatClient
                .builder(chatModel)
                .defaultTools(new DateTimeTools())
                .build();
    }

    @RequestMapping("/call")
    public String call(String message) {
        return chatClient.prompt()
                .user(message)
                .tools(new DateTimeTools())
                .call()
                .content();
    }
}
```

**ChatClient 的优势：**
- 支持流式响应
- 更好的工具调用支持
- 更灵活的提示词构建
- 支持对话历史管理

---

## 🛠️ 五、Tool Calling（工具调用）详解

Tool Calling 是 Spring AI 的核心功能之一，它允许 AI 模型调用外部工具来获取实时信息或执行操作。

### 5.1 什么是 Tool Calling？

想象一下，你问 AI："今天北京的天气怎么样？" 

如果没有工具调用，AI 只能基于训练数据回答（可能是过时的信息）。有了工具调用，AI 可以：
1. 识别出需要查询天气
2. 调用天气查询工具
3. 获取实时天气数据
4. 基于真实数据生成回答

### 5.2 创建自定义工具

#### 示例 1：日期时间工具

```java
package com.example.demo.tool;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.stereotype.Component;

@Component
public class DateTimeTools {
    
    @Tool(description = "Get the current date and time in the user's timezone", 
          returnDirect = true)
    String getCurrentDateTime() {
        return "现在时间是 2000-09-24 12:00";
    }
}
```

**关键点：**
- `@Component`：将工具类交给 Spring 管理
- `@Tool`：标记这是一个可调用的工具
  - `description`：描述工具的功能（AI 通过此描述判断何时调用）
  - `returnDirect`：是否直接返回结果

#### 示例 2：天气查询工具

```java
package com.example.demo.tool;

import org.springframework.stereotype.Component;

@Component
public class WeatherTools {
    
    public String getCurrentWeatherByCityName(String cityName) {
        switch (cityName){
            case "北京":
                return "北京今天天气: 晴空万里";
            case "上海":
                return "上海今天天气: 电闪雷鸣";
            case "广州":
                return "广州今天天气: 细雨蒙蒙";
            default:
                return "没有该城市的天气信息";
        }
    }
}
```

### 5.3 注册工具的三种方式

#### 方式 1：在构造函数中注册（默认工具）

```java
public ChatController(DashScopeChatModel chatModel) {
    this.chatClient = ChatClient
            .builder(chatModel)
            .defaultTools(new DateTimeTools())  // 默认加载的工具
            .build();
}
```

#### 方式 2：通过 Bean 注入

```java
@Autowired
public ChatController(DashScopeChatModel chatModel, ToolCallback weatherTool) {
    this.chatClient = ChatClient
            .builder(chatModel)
            .defaultTools(new DateTimeTools())
            .defaultToolCallbacks(weatherTool)  // 注入的工具
            .build();
}
```

需要在配置类中定义 Bean：

```java
@Configuration
public class ToolConfig {
    
    @Bean
    public ToolCallback weatherTool() {
        Method method = ReflectionUtils.findMethod(
            WeatherTools.class, 
            "getCurrentWeatherByCityName", 
            String.class
        );
        
        return MethodToolCallback.builder()
                .toolDefinition(ToolDefinitions
                        .builder(method)
                        .description("根据指定的城市名称，获取城市当前的天气")
                        .build())
                .toolMethod(method)
                .toolObject(new WeatherTools())
                .build();
    }
}
```

#### 方式 3：在请求时动态指定

```java
@RequestMapping("/call")
public String call(String message) {
    return chatClient.prompt()
            .user(message)
            .tools(new DateTimeTools())  // 仅本次请求使用
            .call()
            .content();
}
```

### 5.4 手动构建 ToolCallback

对于没有 `@Tool` 注解的方法，可以手动构建：

```java
@RequestMapping("/call2")
public String call2(String message) {
    Method method = ReflectionUtils.findMethod(
        WeatherTools.class, 
        "getCurrentWeatherByCityName", 
        String.class
    );
    
    ToolCallback toolCallback = MethodToolCallback.builder()
            .toolDefinition(ToolDefinitions
                    .builder(method)
                    .description("根据指定的城市名称，获取城市当前的天气")
                    .build())
            .toolMethod(method)
            .toolObject(new WeatherTools())
            .build();

    return chatClient.prompt()
            .user(message)
            .tools(new DateTimeTools())
            .toolCallbacks(toolCallback)
            .call()
            .content();
}
```

### 5.5 测试工具调用

**测试场景 1：查询时间**
```
请求：现在几点了？
AI 会自动调用 DateTimeTools.getCurrentDateTime()
返回：现在时间是 2000-09-24 12:00
```

**测试场景 2：查询天气**
```
请求：北京今天的天气怎么样？
AI 会：
1. 识别需要查询天气
2. 调用 WeatherTools.getCurrentWeatherByCityName("北京")
3. 返回：北京今天天气: 晴空万里
```

---

## 📝 六、提示词工程（Prompt Engineering）

通过学习课件，我深刻认识到：**提示词的质量直接决定了 AI 输出结果的质量**。

### 6.1 提示词的六大要素

#### 1. 目标（Objective）- 你要做什么？

明确告诉 AI 需要完成的具体任务。

```
❌ 差：写个故事
✅ 好：写一个关于太空探险的儿童故事
```

#### 2. 背景（Context）- 发生在什么场景下？

提供上下文信息，避免歧义。

```
"我们是一家专注智能家居的初创公司，正在筹备新品上市"
```

#### 3. 受众（Audience）- 给谁看？

不同的受众需要不同的表达方式。

```
- 面向高中生 → 避免专业术语
- 给CEO汇报 → 突出数据与决策建议
- 发在小红书 → 轻松活泼，带 emoji
```

#### 4. 风格（Style）- 用什么方式表达？

定义文体类型：新闻体、故事化、学术风、口语化等。

#### 5. 语气（Tone）- 带着什么情绪说话？

```
- 正式严谨：商业报告、法律文书
- 轻松幽默：社交媒体、科普内容
- 激励鼓舞：演讲稿、团队动员
- 客观中立：新闻报道、数据分析
```

#### 6. 格式（Response Format）- 输出长什么样？

指定输出结构，便于后续处理。

```
- 用 Markdown 表格形式
- 输出 JSON 格式
- 不超过 300 字
- 分点列出，每点不超过 50 字
```

### 6.2 优秀提示词示例

```
请为一家智能家居初创公司撰写产品宣传文案：

【背景】我们即将推出一款智能温控器，能根据用户习惯自动调节室内温度
【受众】25-35 岁的年轻白领，注重生活品质
【风格】轻松活泼，类似小红书种草文案
【语气】亲切自然，像朋友之间的分享，不要有推销感
【格式】
- 标题（吸引人）
- 3 个核心卖点（每个不超过 30 字）
- 使用场景描述（100 字以内）
- 结尾加 2-3 个相关 emoji
```

### 6.3 在代码中应用提示词

```java
String systemPrompt = """
    你是一个专业的智能家居顾问。
    请用轻松活泼的语气回答用户问题。
    回答要简洁明了，不超过 200 字。
    """;

return chatClient.prompt()
        。system(systemPrompt)  // 系统提示词
        。user(message)         // 用户消息
        。call()
        。content();
```

---

## 🎓 七、学习心得与最佳实践

### 7.1 核心收获

1. **Spring AI 降低了 AI 集成门槛**
   - 统一的 API 抽象，无需关心底层实现
   - 与 Spring 生态完美融合，学习成本低

2. **Tool Calling 是杀手级功能**
   - 让 AI 能够获取实时信息
   - 扩展了 AI 的能力边界
   - 是实现智能助手的关键技术

3. **提示词工程至关重要**
   - 好的提示词 = 清晰的工作说明书
   - 结构化思考能显著提升输出质量
   - 需要不断迭代和优化

### 7.2 最佳实践建议

#### ✅ 推荐做法

1. **使用 ChatClient 而非直接使用 ChatModel**
   - 更灵活的功能支持
   - 更好的可扩展性

2. **工具类添加 @Component 注解**
   - 便于 Spring 管理
   - 支持依赖注入

3. **为工具方法添加详细的 description**
   - 帮助 AI 准确判断何时调用
   - 提高工具调用的准确率

4. **合理设置 temperature 参数**
   - 事实性问题：0.0-0.3（更确定）
   - 创意性问题：0.7-1.0（更随机）

5. **敏感信息不要硬编码**
   - 使用环境变量或配置中心
   - 生产环境务必做好密钥管理

#### ❌ 避免的陷阱

1. **不要在工具方法中执行耗时操作**
   - 会导致 AI 响应变慢
   - 考虑异步处理或缓存

2. **不要过度依赖 AI 的判断**
   - 关键业务逻辑仍需人工审核
   - 做好异常处理和降级方案

3. **不要忘记限流和成本控制**
   - API 调用是有成本的
   - 实现合理的限流策略

### 7.3 常见问题及解决方案

#### 问题 1：工具没有被调用

**原因：**
- 工具描述不够清晰
- AI 没有理解需要调用工具

**解决：**
```java
// 优化描述，更具体
@Tool(description = "当用户询问当前时间、日期或时区相关信息时调用此工具")
```

#### 问题 2：API 调用失败

**原因：**
- API Key 无效或过期
- 网络连接问题
- 配额用尽

**解决：**
- 检查 API Key 是否正确
- 添加重试机制
- 实现本地缓存

#### 问题 3：响应速度慢

**原因：**
- 模型本身响应慢
- 网络延迟
- 工具调用链路过长

**解决：**
- 选择更快的模型（如 qwen-turbo）
- 优化提示词，减少不必要的上下文
- 并行调用多个工具

---

## 💡 八、总结

通过这几天的学习和实践，我对 Spring AI 有了全面的认识：

1. **Spring AI 让 AI 集成变得简单**：统一的 API、完善的生态、丰富的功能
2. **Tool Calling 是核心竞争力**：让 AI 从"聊天机器人"升级为"智能助手"
3. **提示词工程是必备技能**：好的提示词能显著提升输出质量
4. **实践是最好的老师**：通过动手编码，才能真正理解概念

希望这篇总结能帮助到正在学习 Spring AI 的你！如果有任何问题或建议，欢迎交流讨论。🎉

---

