---
title: AI技术前沿：Function Calling、RAG与MCP的深度解析与应用实践
date: 2025-04-29 12:28:00
top_group_index: 1
categories: AI
tags: [AI, Function Calling, RAG, MCP]
cover: https://sky-take-out-csuft.oss-cn-beijing.aliyuncs.com/img2.jpg
---
# AI技术前沿：Function Calling、RAG与MCP的深度解析与应用实践

随着人工智能技术的迅猛发展，大型语言模型(LLM)正逐步从单纯的文本生成工具进化为能够与外部世界交互的智能系统。在这一演进过程中，Function Calling、RAG(检索增强生成)和MCP(模型上下文协议)三大技术扮演着关键角色。本文将深入探讨这三种技术的核心原理、应用场景以及它们如何共同推动AI应用进入新阶段，同时提供实用的代码示例和行业趋势分析，帮助开发者把握AI技术的最新发展方向。

## 引言：AI技术发展的新阶段

2025年，人工智能技术迎来了前所未有的发展浪潮。从AIGC(人工智能生成内容)到智能体(Agent)，从AI+RPA到推理模型优化，AI的应用场景不断拓展和深化2。在这一背景下，如何让大型语言模型突破自身局限，实现与外部世界的高效交互，成为技术发展的关键挑战。

传统的大型语言模型虽然具备强大的文本理解和生成能力，但仍存在明显局限：无法获取实时信息、缺乏垂直领域专业知识、不能直接操作系统或工具完成任务。为解决这些问题，行业先后发展出了三种关键技术方案：

- **Function Calling**：让模型能够调用预定义的外部函数或API
- **RAG(检索增强生成)**：通过检索外部知识库增强模型的生成能力
- **MCP(模型上下文协议)**：提供标准化的模型与外部工具交互协议

这三种技术各有侧重又相互补充，共同构成了现代AI应用的基础设施。Function Calling解决了"如何调用"的问题，RAG解决了"知识不足"的问题，而MCP则解决了"如何标准化调用"的问题。它们的出现使得AI系统从封闭的问答机器人逐步进化为能够自主完成复杂任务的智能助手。

本文将系统性地介绍这三种技术的核心概念、工作原理、典型应用场景以及未来发展趋势，并通过实际代码示例展示如何将它们应用于真实业务场景中。无论您是AI领域的初学者还是经验丰富的开发者，都能从本文中获得有价值的见解和实践指导。

## Function Calling：让大模型学会"动手"

Function Calling(函数调用)是特定大模型(如OpenAI的GPT-4、Qwen2等)提供的一种机制，使模型能够主动生成结构化输出，以调用外部系统中预定义的函数或API3。这项技术最初由OpenAI在2023年6月推出，最初在GPT-3.5和GPT-4模型上实现，如今已成为各大模型厂商的标配功能。

### Function Calling的工作原理

Function Calling的执行流程可以分解为以下步骤3：

1. **注册外部函数**：开发者预先向大模型注册可用的外部函数接口(建议不超过20个)
2. **用户发起请求**：用户通过自然语言提出需求，AI程序(Agent)接收请求
3. **模型解析评估**：Agent将请求提交给大模型，模型解析语义并评估是否需要调用外部工具
4. **生成调用指令**：如需调用函数，模型生成包含工具ID和输入参数的调用指令并返回
5. **执行函数调用**：Agent程序接收调用指令并执行对应的工具函数
6. **返回处理结果**：工具函数执行后将结果返回给Agent程序
7. **模型二次处理**：Agent将函数返回结果和自定义提示词一起反馈给大模型
8. **生成最终响应**：大模型融合工具返回数据与原始上下文，生成最终结果
9. **呈现给用户**：Agent程序将最终结果输出给终端用户

这一流程的核心在于大模型能够理解何时需要调用外部函数，并正确生成结构化调用指令，而非直接执行函数调用。实际执行仍由外部程序完成，确保了安全性和可控性。

### Function Calling的代码示例

以下是一个完整的天气查询Function Calling示例，展示了从配置工具到二次处理的完整流程：

```
import openai # 1.75.0
import json

def get_weather(location):
    """模拟天气查询函数"""
    return '{"Celsius": 27, "type": "sunny"}'

def main():
    client = openai.OpenAI(
        api_key="xxxxx",
        base_url="https://api.siliconflow.cn/v1"
    )
    
    # 定义可供调用的工具
    tools = [{
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名"
                    }
                },
                "required": ["city"],
            }
        }
    }]
    
    # 第一次请求：获取模型生成的函数调用指令
    res = client.chat.completions.create(
        model="Qwen/Qwen2.5-32B-Instruct",
        messages=[
            {"role": "system", "content": "你是一个天气查询助手"},
            {"role": "user", "content": "帮我查询上海的天气"}
        ],
        tools=tools,
        tool_choice="auto"
    )
    
    print("第一次响应:", res.choices[0].message.to_dict())
    
    # 准备二次请求的上下文
    messages = [
        {"role": "system", "content": "你是一个天气查询助手"},
        {"role": "user", "content": "帮我查询上海的天气"},
        res.choices[0].message.to_dict()  # 加入第一次的模型响应
    ]
    
    # 执行函数调用
    tool_call = res.choices[0].message.tool_calls[0]
    arguments = json.loads(tool_call.function.arguments)
    messages.append({
        "role": "tool",
        "content": get_weather(arguments['city']),
        "tool_call_id": tool_call.id
    })
    
    # 第二次请求：让模型处理函数返回结果
    res = client.chat.completions.create(
        model="Qwen/Qwen2.5-32B-Instruct",
        messages=messages,
        tools=tools,
        tool_choice="auto"
    )
    
    print("最终响应:", res.choices[0].message.content)

if __name__ == "__main__":
    main()
```

执行上述代码后，输出将分为两个阶段：

1. 第一次响应中，模型识别出需要调用`get_weather`函数，并提供了参数`{"city": "上海"}`
2. 第二次响应中，模型根据模拟的天气数据生成最终的自然语言回复

使用OpenAI Java SDK实现天气查询功能：

```
import com.theokanning.openai.client.OpenAiApi;
import com.theokanning.openai.service.OpenAiService;
import com.theokanning.openai.completion.chat.*;
import retrofit2.Retrofit;
import retrofit2.adapter.rxjava2.RxJava2CallAdapterFactory;
import retrofit2.converter.jackson.JacksonConverterFactory;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;

public class FunctionCallingExample {
    
    public static String getWeather(String location) {
        // 模拟天气查询
        return "{\"temperature\": 27, \"condition\": \"sunny\"}";
    }

    public static void main(String[] args) {
        // 初始化OpenAI服务
        OpenAiService service = new OpenAiService("your-api-key", Duration.ofSeconds(30));
        
        // 定义工具列表
        List<ChatTool> tools = new ArrayList<>();
        ChatFunction function = ChatFunction.builder()
                .name("get_weather")
                .description("获取指定城市的天气")
                .executor(arguments -> getWeather(arguments.get("city").asText())) // 实际执行函数
                .build();
        tools.add(ChatTool.builder().function(function).build());
        
        // 构建聊天消息
        List<ChatMessage> messages = new ArrayList<>();
        messages.add(new ChatMessage(ChatMessageRole.SYSTEM.value(), "你是一个天气查询助手"));
        messages.add(new ChatMessage(ChatMessageRole.USER.value(), "帮我查询上海的天气"));
        
        // 第一次请求：获取函数调用指令
        ChatCompletionRequest chatRequest = ChatCompletionRequest.builder()
                .model("gpt-4")
                .messages(messages)
                .tools(tools)
                .build();
        
        ChatCompletionResult result = service.createChatCompletion(chatRequest);
        System.out.println("第一次响应: " + result.getChoices().get(0).getMessage());
        
        // 准备第二次请求的上下文
        messages.add(result.getChoices().get(0).getMessage());
        
        // 执行函数调用并添加结果到上下文
        ChatMessage assistantMessage = result.getChoices().get(0).getMessage();
        if (assistantMessage.getToolCalls() != null && !assistantMessage.getToolCalls().isEmpty()) {
            ChatToolCall toolCall = assistantMessage.getToolCalls().get(0);
            String functionResult = getWeather(toolCall.getFunction().getArguments());
            
            messages.add(new ChatMessage(
                ChatMessageRole.TOOL.value(),
                functionResult,
                toolCall.getId()
            ));
        }
        
        // 第二次请求：获取最终回答
        chatRequest = ChatCompletionRequest.builder()
                .model("gpt-4")
                .messages(messages)
                .tools(tools)
                .build();
        
        result = service.createChatCompletion(chatRequest);
        System.out.println("最终响应: " + result.getChoices().get(0).getMessage().getContent());
    }
}
```

### Function Calling的特点与局限

Function Calling具有以下显著特点3：

- **实时反馈**：模型生成的函数调用指令执行后，结果可再次反馈给模型，确保回应的实时性和准确性
- **实现灵活**：没有严格的通信协议标准，格式取决于具体模型厂商的实现
- **简单直接**：适用于明确定义的简单任务，如查询天气、股票价格等

然而，Function Calling也存在明显局限性：

1. **工具数量限制**：当可调用工具数量增多(如几十上百个)时，模型难以在冗长的函数列表中准确选择，提示词复杂度急剧上升9
2. **上下文膨胀**：每个工具的定义都需要放入上下文，导致token消耗快速增加
3. **生态封闭**：不同厂商的Function Calling实现各异，缺乏统一标准3
4. **单次交互**：通常采用简单的请求-响应模式，缺乏交互延续性3

这些局限性促使了更高级的交互协议——MCP的出现，我们将在后续章节详细探讨。

## RAG：增强模型的知识能力

RAG(Retrieval-Augmented Generation，检索增强生成)是解决大模型知识局限性的一项重要技术。与Function Calling关注"如何调用工具"不同，RAG主要解决"如何获取知识"的问题，特别适用于需要实时或专有领域知识的场景3。

### RAG的核心思想

RAG技术的基本原理可以概括为3：

1. **检索阶段**：当用户提出问题后，系统首先从外部知识库中检索与问题相关的文档或信息片段
2. **增强生成**：将检索到的相关内容与用户原始问题一起提供给大模型，作为生成回答的上下文
3. **生成回答**：大模型基于自身理解能力和提供的上下文信息，生成最终回答

这种"检索+生成"的双阶段模式，有效克服了大模型的几大固有局限：

- **知识实时性**：通过检索最新知识库，解决模型训练数据过时的问题
- **领域专业性**：通过检索专有知识库，弥补通用模型在垂直领域知识的不足
- **事实准确性**：通过提供参考来源，减少模型"幻觉"(编造事实)的发生
- **可解释性**：生成的回答可以关联到具体检索结果，提高可信度

### RAG的典型架构

一个完整的RAG系统通常包含以下组件：

1. **文档处理流水线**：
   - 文档采集与清洗
   - 文本分块(Chunking)
   - 向量嵌入(Embedding)
   - 向量数据库存储
2. **检索组件**：
   - 查询理解与改写
   - 向量相似度检索
   - 混合检索(结合关键词与向量)
   - 结果重排序
3. **生成组件**：
   - 提示词工程
   - 上下文窗口管理
   - 生成结果后处理
4. **评估与优化**：
   - 检索相关性评估
   - 生成质量评估
   - 端到端A/B测试

### RAG的实现示例

以下是一个简化的RAG实现示例，使用LangChain和FAISS向量数据库：

```
from langchain.document_loaders import WebBaseLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.vectorstores import FAISS
from langchain.chains import RetrievalQA
from langchain.llms import OpenAI

# 1. 加载文档
loader = WebBaseLoader(["https://example.com/ai-article"])
documents = loader.load()

# 2. 文档分块
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
texts = text_splitter.split_documents(documents)

# 3. 创建向量存储
embeddings = HuggingFaceEmbeddings(model_name="sentence-transformers/all-mpnet-base-v2")
db = FAISS.from_documents(texts, embeddings)

# 4. 创建RAG链
retriever = db.as_retriever(search_kwargs={"k": 3})
qa_chain = RetrievalQA.from_chain_type(
    llm=OpenAI(temperature=0),
    chain_type="stuff",
    retriever=retriever,
    return_source_documents=True
)

# 5. 提问
query = "什么是RAG技术？"
result = qa_chain({"query": query})

print("答案:", result["result"])
print("来源:", result["source_documents"])
```

使用Spring AI和向量数据库实现检索增强生成：

```
import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingClient;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.openai.OpenAiChatClient;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.SystemPromptTemplate;
import org.springframework.ai.chat.messages.UserMessage;

import java.util.List;
import java.util.Map;

public class RagExample {
    
    private final EmbeddingClient embeddingClient;
    private final OpenAiChatClient chatClient;
    private VectorStore vectorStore;
    
    public RagExample(EmbeddingClient embeddingClient, OpenAiChatClient chatClient) {
        this.embeddingClient = embeddingClient;
        this.chatClient = chatClient;
        initializeVectorStore();
    }
    
    private void initializeVectorStore() {
        // 初始化向量存储
        vectorStore = new SimpleVectorStore(embeddingClient);
        
        // 添加示例文档（实际应用中可以从文件或数据库加载）
        vectorStore.add(List.of(
            new Document("RAG(检索增强生成)是一种结合检索和生成的技术...", 
                Map.of("source", "AI技术手册")),
            new Document("RAG系统通常包含检索和生成两个阶段...", 
                Map.of("source", "技术白皮书"))
        ));
    }
    
    public String query(String question) {
        // 检索相关文档
        List<Document> docs = vectorStore.similaritySearch(question);
        
        // 构建系统提示
        SystemPromptTemplate systemPromptTemplate = new SystemPromptTemplate("""
            你是一个AI助手，请基于以下上下文回答问题：
            {context}
            
            如果上下文不包含答案，请回答"我不知道"。""");
        
        // 合并检索到的文档作为上下文
        String context = docs.stream()
            .map(Document::getContent)
            .reduce("", (a, b) -> a + "\n\n" + b);
        
        // 创建提示
        Prompt prompt = new Prompt(List.of(
            systemPromptTemplate.createMessage(Map.of("context", context)),
            new UserMessage(question)
        ));
        
        // 获取AI响应
        ChatResponse response = chatClient.call(prompt);
        return response.getResult().getOutput().getContent();
    }
    
    public static void main(String[] args) {
        // 实际应用中可以通过Spring依赖注入获取这些bean
        EmbeddingClient embeddingClient = new OpenAiEmbeddingClient("your-api-key");
        OpenAiChatClient chatClient = new OpenAiChatClient("your-api-key");
        
        RagExample rag = new RagExample(embeddingClient, chatClient);
        String answer = rag.query("什么是RAG技术？");
        System.out.println("答案: " + answer);
    }
}
```

### RAG的应用场景

RAG技术特别适用于以下场景3：

1. **企业知识问答**：构建基于企业内部文档、手册、邮件的智能问答系统
2. **客服助手**：整合产品文档、常见问题解答，提供精准客服支持
3. **法律与医疗**：在高度专业化的领域提供基于最新指南的回答
4. **学术研究**：链接论文数据库，辅助文献综述和研究分析
5. **新闻分析**：结合实时新闻源，提供背景分析和趋势解读

### RAG的优化方向

虽然RAG技术已经得到广泛应用，但仍面临多项挑战，主要包括：

1. **检索质量**：如何确保检索结果全面且相关
2. **上下文窗口**：处理长文档时的上下文管理问题
3. **多模态扩展**：支持图像、表格等非文本内容的检索与生成
4. **实时性保证**：知识库更新与检索结果同步的延迟问题
5. **评估体系**：建立客观全面的RAG系统评估指标

随着模型上下文窗口的不断扩大(如百万token级别的上下文)，以及向量检索技术的持续进步，RAG系统的能力和效率将进一步提升。2025年，上下文窗口限制导致的文本幻觉问题有望被彻底解决5，这将极大增强RAG系统的可靠性。

## MCP：模型交互的标准化协议

MCP(Model Context Protocol，模型上下文协议)是Anthropic公司在2024年11月推出的一种开放标准，旨在统一大型语言模型与外部数据源和工具之间的通信协议3。如果说Function Calling是"点对点"的工具调用，那么MCP则构建了一个"总线式"的工具交互生态系统。

### MCP的设计初衷

MCP的诞生主要为了解决以下几个核心问题36：

1. **数据孤岛问题**：AI模型因无法有效访问分散的数据源而潜力受限
2. **接口标准化**：不同工具需要单独开发接口，开发维护成本高
3. **安全与隐私**：缺乏统一的安全控制机制，数据泄露风险高
4. **生态碎片化**：各厂商实现互不兼容，难以形成规模效应

MCP的设计者将其类比为"AI领域的USB接口"，无论AI模型还是外部工具，只要符合MCP标准，就可以实现快速"即插即用"的连接，无需为每个工具单独编写接口程序14。

### MCP的核心特性

MCP协议具有以下显著特点36：

1. **开放性**：开放标准，任何开发者或服务商均可基于此协议开发API，推动生态共建
2. **标准化**：采用JSON-RPC 2.0标准通信，确保交互统一高效
3. **AI增强**：将AI应用从简单问答升级为可执行复杂任务的工具
4. **安全性**：基于标准协议的数据交互，便于控制数据流、防止泄露
5. **兼容性**：支持文件内容、数据库记录、API响应、实时数据等多种格式
6. **扩展性**：提供提示词模板、工具、采样等功能，灵活扩展交互能力

### MCP的架构组成

MCP采用客户端-服务器(Client-Server)架构，包含以下核心组件3：

1. **MCP Host**：发起请求的AI应用程序或工具，如Claude Desktop、Cursor等
2. **MCP Client**：位于Host内部，保持与MCP Server的一对一连接，负责消息路由和管理
3. **MCP Server**：提供上下文数据、工具和提示词模板的服务端组件
4. **资源与工具**：包括本地/远程数据资源及可被模型调用的功能工具

### MCP的工作流程

一个典型的MCP调用流程如下3：

1. **配置连接**：宿主程序(客户端)中配置相关的MCP Server并建立连接
2. **用户提问**：用户使用自然语言提问，宿主程序整合用户问题和MCP工具提示
3. **模型理解**：大模型理解后产生调用指令
4. **请求发送**：宿主程序将调用指令通过Client发送给MCP Server
5. **执行操作**：MCP Server解析请求并执行相应操作(如搜索、记录等)
6. **返回结果**：Server将处理结果封装成响应消息返回客户端

### MCP与Function Calling的对比

虽然MCP和Function Calling都涉及模型与外部工具的交互，但两者在多个维度上存在显著差异39：

| 对比维度     | Function Calling              | MCP                                  |
| :----------- | :---------------------------- | :----------------------------------- |
| **交互模式** | 简单的请求-响应模式，单次调用 | 交互式、持续性的上下文管理，多轮互动 |
| **定位**     | 特定模型厂商提供的扩展能力    | 开放的标准协议，定义通用通信架构     |
| **通信协议** | 无统一标准，依赖厂商实现      | 严格遵守JSON-RPC 2.0，高度标准化     |
| **生态开放** | 相对封闭，依赖特定厂商支持    | 生态开放，社区共建为主               |
| **工具规模** | 适合少量工具(20个以内)        | 适合大量工具的标准化接入             |
| **开发成本** | 快速实现简单功能              | 需要额外MCP Server基础设施           |
| **适用场景** | 简单明确定义的任务            | 复杂业务流程、多工具协同             |

### MCP的应用示例：Spring AI集成

Spring AI对MCP的支持展示了这项技术在企业级应用中的潜力6。通过MCP，Spring AI应用可以：

1. **跨数据源整合**：统一访问数据库、文件系统、API等异构数据源
2. **降低开发成本**：避免为每个新数据源单独开发接口
3. **增强协作能力**：在聊天界面直接访问GitHub、Google Drive等平台资源
4. **提升灵活性**：支持结构化/非结构化数据的统一处理

以下是一个简化的Spring AI与MCP集成的伪代码示例：

```
// 配置MCP客户端
@Bean
public McpClient mcpClient() {
    return McpClient.builder()
            .serverUrl("https://mcp.example.com")
            .credentials("api-key")
            .build();
}

// 定义AI服务
@Service
public class AIService {
    
    private final ChatClient chatClient;
    private final McpClient mcpClient;
    
    public AIService(ChatClient chatClient, McpClient mcpClient) {
        this.chatClient = chatClient;
        this.mcpClient = mcpClient;
    }
    
    public String handleQuery(String userQuery) {
        // 通过MCP获取相关上下文
        McpResponse context = mcpClient.retrieve()
                .query(userQuery)
                .execute();
        
        // 构建增强提示
        Prompt prompt = new Prompt(
            "你是一个智能助手，请基于以下上下文回答问题：\n" +
            context.getContent() + "\n\n问题：" + userQuery
        );
        
        // 获取AI生成
        return chatClient.call(prompt).getResult();
    }
}
```

使用Spring AI集成MCP协议的示例：

```
import org.springframework.ai.mcp.McpClient;
import org.springframework.ai.mcp.McpResponse;
import org.springframework.ai.chat.ChatClient;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.stereotype.Service;

@Service
public class McpIntegrationService {
    
    private final ChatClient chatClient;
    private final McpClient mcpClient;
    
    public McpIntegrationService(ChatClient chatClient, McpClient mcpClient) {
        this.chatClient = chatClient;
        this.mcpClient = mcpClient;
    }
    
    public String processQuery(String userQuery) {
        // 通过MCP获取上下文
        McpResponse context = mcpClient.retrieve()
                .query(userQuery)
                .execute();
        
        // 构建增强提示
        Prompt prompt = new Prompt(
            "你是一个智能助手，请基于以下上下文回答问题：\n" +
            context.getContent() + "\n\n问题：" + userQuery
        );
        
        // 获取AI生成
        return chatClient.call(prompt).getResult().getOutput().getContent();
    }
    
    // 配置类示例
    @Configuration
    public static class McpConfig {
        
        @Bean
        public McpClient mcpClient() {
            return McpClient.builder()
                    .serverUrl("https://mcp.example.com")
                    .apiKey("your-mcp-api-key")
                    .build();
        }
    }
}
```



### MCP的发展前景

随着更多技术公司和开发者加入MCP生态，其应用前景广阔6：

1. **行业渗透**：医疗、金融、教育等行业通过MCP实现信息快速流动
2. **智能体互联**：支持Agent互联网协议，实现多智能体协同2
3. **边缘计算**：与轻量化AI结合，推动端侧智能发展10
4. **标准化推进**：可能成为AI工具交互的事实标准，类似REST API在Web领域的地位

## 三大技术的协同应用与案例分析

### 技术融合的威力：Function Calling + RAG + MCP

在实际应用中，这三种技术往往不是孤立使用，而是相互配合形成完整解决方案。一个典型的协同工作流程可能是：

1. **用户提问**："帮我分析最近三个月公司销售数据，总结趋势并给出改进建议"
2. **RAG检索**：系统从企业文档库检索"销售分析报告模板"和"产品定价策略"
3. **Function Calling**：模型调用数据分析API获取最新销售数据
4. **MCP交互**：通过标准化协议连接CRM系统获取客户反馈数据
5. **综合生成**：模型整合所有信息，生成结构化报告和建议

### 实际案例：智能医疗助手

**场景**：某三甲医院部署的AI医疗助手系统

**技术组成**：

- **RAG**：对接医学文献库、诊疗指南和医院病例数据库
- **Function Calling**：调用检查单生成、药品查询、预约挂号等医院HIS系统功能
- **MCP**：标准化连接各类医疗设备和实验室系统

**工作流程**：

1. 医生询问："55岁男性，高血压病史，近期空腹血糖7.8mmol/L，建议如何处理？"
2. 系统通过RAG检索最新糖尿病诊疗指南和类似病例
3. 通过Function Calling调取患者完整病历和检查记录
4. 基于MCP获取实时可用的检查设备和药品库存
5. 生成个性化建议：建议OGTT检查，考虑二甲双胍治疗，并可直接开具检查单

**效果**：

- 诊疗建议符合最新指南且考虑医院实际资源
- 减少医生50%的常规决策时间
- 提高治疗方案标准化程度
