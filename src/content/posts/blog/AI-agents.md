---
title: AI-agents
published: 2025-09-18
tags:
  - 教程
  - AI工具
  - 推荐
lang: zh
abbrlink:  ai-agents-study
---

### 1. 学习AI-agents
1. [huggingface](https://huggingface.co/learn/agents-course/en/unit2/smolagents/tools)

2. [阿里云 ACP](https://edu.aliyun.com/certification/acp26?spm=a2cwt.28380597.J_1564692210.19.42943487psvDVN): 可以白嫖一份3个月的服务器资源 

3. [Microsoft Learn AI-agents](https://github.com/microsoft/ai-agents-for-beginners)

### 2. 定义工具（Tools）
 MCP : https://modelcontextprotocol.io/docs/getting-started/intro <br>
**Python 实现**
 ```python
 from smolagents import CodeAgent, LiteLLMModel, DuckDuckGoSearchTool,tool

# 配置 Ollama 模型
model = LiteLLMModel(
    model_id="ollama_chat/deepseek-r1:14b", # 格式是ollama_chat/xxx
    base_url="http://localhost:11434",
    api_key=None, # 默认为 None
    num_ctx=8192
)

@tool
def duckduckgo_search(query: str) -> str:
    """DuckDuckGo 搜索工具"""
    url = f"https://duckduckgo.com/?q={query}"
    return url

# 添加工具
tools = [DuckDuckGoSearchTool(), duckduckgo_search]  # 添加 DuckDuckGo 搜索工具

# 初始化 SmolAgents 的 CodeAgent
agent = CodeAgent(tools=tools, model=model, add_base_tools=True)

# 测试智能体
output = agent.run("What is the latest news about AI?")
print("Final output:")
print(output)
 ```

**Java 实现**
```xml
<!-- 引入依赖 spring-ai-mcp-server-webmvc-spring-boot-starter -->
<!-- 此方法通信方式为 sse, 其他方式请查看官方文档以及注意JDK版本 -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-mcp-server-webmvc-spring-boot-starter</artifactId>
        </dependency>
```

 ```java
 // 定义 工具
 @Service
public class WetherMcpServer {
    //定义了一个查询天气的HTTP客户端
    private final RestClient restClient;

    public WetherMcpServer() {
        this.restClient = RestClient.builder()
                .baseUrl("https://api.weather.gov")
                .defaultHeader("Accept", "application/geo+json")
                .defaultHeader("User-Agent", "WeatherApiClient/1.0 (your@email.com)")
                .build();
    }

     //定义 RestClient 接口对象参数，这里record是高版本Java特性，可以省略getters和setters方法
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Points(@JsonProperty("properties") Props properties) {
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Props(@JsonProperty("forecast") String forecast) {
        }
    }
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Forecast(@JsonProperty("properties") Props properties) {
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Props(@JsonProperty("periods") List<Period> periods) {
        }

        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Period(@JsonProperty("number") Integer number, @JsonProperty("name") String name,
                             @JsonProperty("startTime") String startTime, @JsonProperty("endTime") String endTime,
                             @JsonProperty("isDaytime") Boolean isDayTime, @JsonProperty("temperature") Integer temperature,
                             @JsonProperty("temperatureUnit") String temperatureUnit,
                             @JsonProperty("temperatureTrend") String temperatureTrend,
                             @JsonProperty("probabilityOfPrecipitation") Map probabilityOfPrecipitation,
                             @JsonProperty("windSpeed") String windSpeed, @JsonProperty("windDirection") String windDirection,
                             @JsonProperty("icon") String icon, @JsonProperty("shortForecast") String shortForecast,
                             @JsonProperty("detailedForecast") String detailedForecast) {
        }
    }

    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Alert(@JsonProperty("features") List<Feature> features) {

        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Feature(@JsonProperty("properties") Properties properties) {
        }

        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Properties(@JsonProperty("event") String event, @JsonProperty("areaDesc") String areaDesc,
                                 @JsonProperty("severity") String severity, @JsonProperty("description") String description,
                                 @JsonProperty("instruction") String instruction) {
        }
    }


    // 获取天气信息，并格式化输出，简单来说就是通过restClient去做获取天气信息的操作，将结果输出给大模型。
    @Tool(description = "Get weather forecast for a specific latitude/longitude")
    public String getWeatherForecastByLocation(
            double latitude,   // Latitude coordinate
            double longitude   // Longitude coordinate
    ) {
        var points = restClient.get()
                .uri("/points/{latitude},{longitude}", latitude, longitude)
                .retrieve()
                .body(Points.class);
        
        var forecast = restClient.get().uri(points.properties().forecast()).retrieve().body(Forecast.class);

        String forecastText = forecast.properties().periods().stream().map(p -> String.format("""
                %s:
                Temperature: %s %s
                Wind: %s %s
                Forecast: %s
                """, p.name(), p.temperature(), p.temperatureUnit(), p.windSpeed(), p.windDirection(),
                p.detailedForecast())).collect(Collectors.joining());

        return forecastText;
    }

    @Tool(description = "Get weather alerts for a US state")
    public String getAlerts(
            @ToolParam(description = "Two-letter US state code (e.g. CA, NY") String state)
    {
        Alert alert = restClient.get().uri("/alerts/active/area/{state}", state).retrieve().body(Alert.class);

        return alert.features()
                .stream()
                .map(f -> String.format("""
					Event: %s
					Area: %s
					Severity: %s
					Description: %s
					Instructions: %s
					""", f.properties().event(), f.properties.areaDesc(), f.properties.severity(),
                        f.properties.description(), f.properties.instruction()))
                .collect(Collectors.joining("\n"));
    }
}

    // 注册工具
    @Bean
    public ToolCallbackProvider weatherTools(WetherMcpServer weatherService) {
        return  MethodToolCallbackProvider.builder().toolObjects(weatherService).build();
    }
 ```
 ### 3. 模型（Models、Brain）
 1. 本地部署 [Ollama](https://ollama.com/) 、[vLLM](https://docs.vllm.com.cn/en/latest/index.html)
 > Ollama 安装建议查看GitHub安装教程，修改安装目录；vLLM 安装性能比Ollama好。
 2. 使用各个大模型厂商提供的API

 ### 4. 测试MCP
 1. 配置MCP服务，以Cherry Studio为例
 ![image](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920233729526-2061079896.png)
 ![image](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920233912778-1981479759.png)
 2. 将MCP服务绑定大模型，再使用对应模型的助手工具进行测试
 ![image](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250920234132738-763700399.png)

 ### 5. RAG（Retrieval Augmented Generation）
 > 通过指定数据源以增强LLM的生成能力，提高准确性。

 ### 6. 模型微调（Fine-tuning）