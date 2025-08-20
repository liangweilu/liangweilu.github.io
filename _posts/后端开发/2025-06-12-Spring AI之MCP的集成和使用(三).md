---
layout: post
title: Spring AI之MCP的集成和使用(三)
date: 2025-06-12
tags: SpringAI
categories:
  - 后端开发
top: 4
---

## 1. 什么是MCP
简单来说，MCP ([Model Context Protocol](https://modelcontextprotocol.io/introduction)就是一个使大模型更好的使用外部工具能力的协议，
它为应用程序向 LLM 提供上下文的方式进行了标准化。它就像一个标准的USB-C接口连接各种外设一样，将我们的AI应用程序和各种数据源与工具进行标准化连接。
MCP 为 AI 模型连接各种数据源和工具提供了标准化的接口，让AI可以轻松操作外部工具，完成更加复杂的任务，从而发挥真正的“工具调用”能力。

MCP是一个客户端-服务器架构，主机可以连接到多个MCP服务器，如下图所示：
![](/images/20250612/mcp-1.png)
MCP架构中的几个角色说明如下：
- **MCP Hosts**: 如 Claude Desktop、IDE 或 AI 工具，希望通过 MCP 访问数据的程序
- **MCP Clients**: 维护与服务器一对一连接的协议客户端
- **MCP Servers**: 轻量级程序，通过标准的 Model Context Protocol 提供特定能力
- **本地数据源**: MCP 服务器可安全访问的计算机文件、数据库和服务
- **远程服务**: MCP 服务器可连接的互联网上的外部系统（如通过 APIs）

看到这里明白MCP是什么了吗，下面这个图是我理解的MCP：
![](/images/20250612/mcp-2.png)
为了加深理解，请看下文的实操例子进行理解MCP。

## 2. 理解MCP客户端和服务端
为了更好的理解MCP客户端和服务端，这里用现有的模型和MCP服务进行测试和演示，参考官网的文件系统MCP-Server，我们进行一下实际操作。在进行测试前，需要
做如下准备：
- 本地已经安装好了[Claude Desktop](https://claude.ai/download)；
- 启动一个[文件系统MCP-Server](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem)；

关于Claude的介绍以及Claude Desktop的安装参考官网，Claude Desktop就是一个桌面客户端版本的AI聊天应用，和我们在网页上使用ChatGPT一样，只是这
个客户端还提供了连接到MCP-Server的能力。

### 2.1 安装Claude Desktop
[官网下载](https://claude.ai/download)安装文件后直接安装即可，安装好Claude Desktop之后，我们需要注册并登录Claude才能使用AI模型，打开程序后界面如下：
![](/images/20250612/mcp-3.png)

### 2.2 启动文件系统服务
此时我们和Claude进行聊天，比如我们想让他访问我们本地计算机上的文件系统，实际上它是不具备这个能力和权限的，这个时候就需要我们的文件系统MCP-Server了。

直接参考github上的说明，使用如下命令在本机启动一个文件系统MCP-Server：
```text
# 后面的目录是代表可以访问的目录
npx -y @modelcontextprotocol/server-filesystem D:\\tmp
```
启动成功后界面如下：
![](/images/20250612/mcp-4.png)
> 后来测试发现，可以不用手动启动，Claude Desktop会自动启动  
> 但是我们也可以自己手动启动一个mcp server，用来为后边自己编写mcp client测试使用。

### 2.3 Claude配置连接MCP server
此时我们的Claude Desktop就是一个AI应用，MCP客户端是这个应用的一部分，我们需要将MCP客户端连接到MCP服务端，点击Claude Desktop中
的`File-->Setting-->Developer`，然后点击`Edit Config`编辑MCP server配置，配置存在一个`claude_desktop_config.json`文件中。配置如下：
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "D:\\tmp"
      ]
    }
  }
}
```
配置完成后重启ClaudeDesktop然后点击如下图标就可以看到我们的文件系统了
![](/images/20250612/mcp-5.png)
此时我们向Claude发送一个要求总结文件并输出的对话，然后它就会调用我们的文件系统MCP-Server提供的工具，并且每次访问工具都会询问是否同意，效果如下：
![](/images/20250612/mcp-6.png)
总结的内容如下图，符合我们的预期。
![](/images/20250612/mcp-7.png)
在这个例子中，底层的执行流程如下：
![](/images/20250612/mcp-8.png)
1. mcp客户端连接到mcp服务端，并获取服务端的能力；
2. 用户向AI应用程序发送了一段对话；
3. AI应用程序将用户发送的对话和mcp服务端的能力一起发送给大模型；
4. 大模型判断用户的目的，决定使用哪些能力；
5. 大模型使用这些能力，通过mcp客户端调用服务端，并将最终以自然语言反馈给用户；

## 3. 实现一个MCP客户端
MCP官方提供了很多SDK支持，这里我们使用[Spring AI MCP](https://doc.spring4all.com/spring-ai/reference/api/mcp/mcp-client-boot-starter-docs.html)
提供的starter来实现一个客户端，Spring AI中的客户端starter是基于[MCP Java SDK](https://mcp-docs.cn/sdk/java/mcp-overview)实现的。
我们基于前文的RAG集成中的工程进行改造，实现一个集成filesystem和高德地图的的AI应用。MCP客户端、服务端，以及AI模型交互如下图所示：
![](/images/20250612/mcp-9.png)

### 3.1 配置依赖
此处仅列出关键的依赖，就是openai模型依赖starter以及mcp客户端默认starter，完整代码见[github](https://github.com/liangweilu/springAI/tree/master/spring-ai-mcp-client)仓库。
```xml
<dependencies>
        <!-- openai 模型依赖 -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-model-openai</artifactId>
        </dependency>

        <!-- mcp client依赖 -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-mcp-client</artifactId>
        </dependency>
    </dependencies>
```
> 这里的依赖配置，在相同的代码情况下将spring-ai-starter-mcp-client改为spring-ai-starter-mcp-client-webflux也是能运行的

### 3.2 主要代码和配置
本例是基于OpenAi模型实现的，实际使用Ollama模型来实现也是可以，只是本地由于资源限制，响应速度太慢了。首先我们需要配置ChatClient实例，代码如下：
```java
@Configuration
public class CustomChatConfig {
    @Autowired
    private OpenAiChatModel chatModel;
    @Autowired
    private ChatMemory chatMemory;

    @Autowired
    private ToolCallbackProvider toolCallbackProvider;

    @Bean
    public ChatClient chatClient() {
        return ChatClient.builder(chatModel)
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
                .defaultToolCallbacks(toolCallbackProvider)
                .build();
    }
}
```
其次是在applicaiton.yaml配置文件中配置客户端和大模型信息：
```yaml
server:
  port: 9103

spring:
  ai:
    openai:
      api-key: ${OPEN_AI_API_KEY_PROXY}  # 这里的key需要替换成自己的key
      base-url: https://api.chatanywhere.tech   # 国内免费代理的open ai
    mcp:
      client:
        toolcallback:
          # 默认开启，允许 MCP 工具作为 AI 交互的一部分使用
          enabled: true
        name: my-mcp-client
        root-change-notification: true
        stdio:
          # 配置文件这种方式仅支持 STDIO 连接类型
          servers-configuration: classpath:mcp-servers.json
```
mcp-server.json配置是Claude Desktop配置格式，还可以直接在yaml中配置command，这里采用json配置更方便，更多配置方式
见[官方配置文档](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-client-boot-starter-docs.html)。我这里配
置了两个mcp-server，一个是文件系统，一个是高德地图提供的mcp server。
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx.cmd",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "D:\\tmp",
        "F:\\git\\springAI"
      ]
    },
    "amap-maps": {
      "command": "npx.cmd",
      "args": [
        "-y",
        "@amap/amap-maps-mcp-server"
      ],
      "env": {
        "AMAP_MAPS_API_KEY": "0f99165445346cd6bd5fd92be274ccc7"
      }
    }
  }
}
```
### 3.3 测试
启动程序，就可以在web和AI聊天了，如下是我的聊天截图
![](/images/20250612/mcp-10.png)

### 3.4 底层交互分析
在这个例子中，我们使用Spring AI的自动配置实现了一个AI应用，并集成了MCP Client。虽然本地启动了一个MCP Server，但是我们的应用是如何与server
交互的，底层原理我们并不知道。所以我将这个交互流程改造一下，在MCP客户端和服务端之间做一个代理，将客户端和服务端的交互输入输出打印到日志中进行分析。
改造后交互如下：
![](/images/20250612/mcp-11.png)
现在我们将mcp-server.json中的配置修改一下：
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "node.exe",
      "args": [
        "F:\\git\\springAI\\spring-ai-mcp-client\\src\\main\\resources\\mcp-logger.js"
      ]
    },
    "amap-maps": {
      "command": "npx.cmd",
      "args": [
        "-y",
        "@amap/amap-maps-mcp-server"
      ],
      "env": {
        "AMAP_MAPS_API_KEY": "0f99165445346cd6bd5fd92be274ccc7"
      }
    }
  }
}
```
再次启动程序后，就可以看到日志中的交互输入和输出信息了，日志路径在mcp-logger.js文件中。然后我们向AI应用发一个问题，得到的答案如下：
![](/images/20250612/mcp-12.png)
此时来看日志中的信息，为了便于好看，我将日志格式化后如下：
```text
-------------------客户端发送-------------------------------------------
{
    "jsonrpc": "2.0",
    "method": "initialize",
    "id": "b3e7e333-0",
    "params": {
        "protocolVersion": "2024-11-05",
        "capabilities": {

        },
        "clientInfo": {
            "name": "my-mcp-client - filesystem",
            "version": "1.0.0"
        }
    }
}
---------------------服务端响应-----------------------------------------
{
    "result": {
        "protocolVersion": "2024-11-05",
        "capabilities": {
            "tools": {

            }
        },
        "serverInfo": {
            "name": "secure-filesystem-server",
            "version": "0.2.0"
        }
    },
    "jsonrpc": "2.0",
    "id": "b3e7e333-0"
}
----------------客户端发送-----------------------------------------------------
{"jsonrpc":"2.0","method":"notifications/initialized"}
-----------------客户端发送：----------------------------------------------------
{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "id": "b3e7e333-3",
    "params": {
        "name": "list_directory",
        "arguments": {
            "path": "D:\\tmp"
        }
    }
}
------------------服务端响应：---------------------------------------------------
{
    "result": {
        "content": [
            {
                "type": "text",
                "text": "[FILE] git-command.txt\n[FILE] output.md"
            }
        ]
    },
    "jsonrpc": "2.0",
    "id": "b3e7e333-3"
}
```
可以看到前面三条就是客户端和服务端连接交互信息，最后两条就是客户端向服务端请求工具，以及得到的响应结果，可以看到客户端和服务端采用的是Json rpc消息
进行数据传输和交互的。

## 4. 实现一个MCP服务端
下面这个图充分体现出了SSE和STDIO的区别，即STDIO是客户端和服务端在进程内进行通信；而sse则是在不同的进程内进行通信。SSE适用于作为独立服务部署的场
景，可以被多个客户端远程调用，具体做法和实现与 stdio 非常类似。
![](/images/20250612/mcp-13.png)

### 4.1 配置依赖
服务端采用webflux实现时，只需要添加如下依赖，如果客户端也使用webflux实现，则需要添加`spring-boot-starter-web`依赖，否则客户端启动会报错。
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webflux</artifactId>
</dependency>
```

### 4.2 主要代码和配置
基于SSE的服务端配置参数中，base-url目前配置了之后无法访问，应该是有bug。
```yaml
server:
  port: 9014

spring:
  ai:
    mcp:
      server:
        name: custom-mcp-server
        sse-endpoint: /sse
        sse-message-endpoint: /mcp/message
#        base-url: /api/v1  # 目前版本不能配置这个，应该是有bug
        type: ASYNC
        capabilities:
          resource: true
          tool: true
          prompt: true
          completion: true

logging:
  level:
    io.modelcontextprotocol: DEBUG  # 开启日志后可以看到底层交互消息
```
在服务端实现MCP工具，这里使用[OpenMeteo](https://open-meteo.com/)实现一个天气服务，根据经纬度查询天气信息。OpenMeteo是一个开源的天气
API，为非商业用途提供免费访问，无需 API 密钥。同时还实现了一个查询本地时间的工具。

天气服务的代码来自[aliaba官方示例](https://github.com/springaialibaba/spring-ai-alibaba-examples/blob/main/spring-ai-alibaba-mcp-example/spring-ai-alibaba-mcp-starter-example/server/mcp-webflux-server-example/src/main/java/org/springframework/ai/mcp/sample/server/OpenMeteoService.java)，此处不再列出。
```java
@Service
public class LocalTimeService {

    /**
     * 该工具用于信息检索
     *
     * @return 当前时间
     */
    @Tool(description = "获取用户所在时区的当前时间")
    public String getCurrentDateTime() {
        return LocalDateTime.now()
                .atZone(LocaleContextHolder.getTimeZone().toZoneId())
                .format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    }
}
```
将工具统一放入配置类中：
```java
@Configuration
public class ToolConfiguration {

    @Autowired
    private OpenMeteoService openMeteoService;

    @Autowired
    private LocalTimeService localTimeService;

    @Bean
    public ToolCallbackProvider registerTools() {
        return MethodToolCallbackProvider.builder()
                .toolObjects(openMeteoService, localTimeService)
                .build();
    }
}
```
启动服务后就可以让客户端来进行连接了并进行测试了。

### 4.1 测试
将4.2节种的mcp服务端编译成jar包，放到服务器上启动，我这里的地址是192.168.140.7，然后我用webflux实现了一个客户端程序，让它通过sse的方式连接
该mcp服务端。代码放在[github](https://github.com/liangweilu/springAI/tree/master/spring-ai-mcp-client-webflux)上，这里不再列出服务端实现。下面是客户端测试截图。
查询天气:
![](/images/20250612/mcp-14.png)
查询当前时间：
![](/images/20250612/mcp-15.png)

## 5. Inspector
[MCP Inspector](https://github.com/modelcontextprotocol/inspector) 是一个交互式开发者工具，用于测试和调试 MCP 服务器。需要node环
境22.7.5版本以上，根据github上的说明，执行如下命令即可运行Inspector。
```
npx @modelcontextprotocol/inspector
```
成功执行后可以浏览器访问：http://localhost:6274，其UI界面如下：
![](/images/20250612/mcp-16.png)
以高德地图提供的mcp server为例进行测试，首先需要去高德地图开放平台申请api key，然后填入环境变量，然后进入[mcp.so](https://mcp.so/)网站找到高德地图提供
的server（这个网站上有很多第三方已经提供的MCP Server，我们只需要使用就可以），上面有对应的参数命令，根据标准填写到Inspector中即可。
![](/images/20250612/mcp-17.png)
填写好后就进行连接，连接成功后可以在右侧看到高德地图MCP Server提供的tools。效果如下，我们就可以在上面测试这些tools是否可用了。
![](/images/20250612/mcp-18.png)

## 参考
1. [MCP官方中文文档](https://mcp-docs.cn/introduction)
2. [国内个人免费使用Chat-GPT的API](https://github.com/chatanywhere/GPT_API_free?tab=readme-ov-file)
3. [GPT4Free](https://github.com/xtekky/gpt4free)
4. [Spring AI Alibaba文档](https://java2ai.com/blog/spring-ai-alibaba-mcp/?spm=0.29160081.0.0.7bc51b4dfnLmGG)
5. [Spring AI Alibaba官方MCP Server示例](https://github.com/springaialibaba/spring-ai-alibaba-examples/blob/main/spring-ai-alibaba-mcp-example/spring-ai-alibaba-mcp-starter-example/server/mcp-webflux-server-example/README.md)
