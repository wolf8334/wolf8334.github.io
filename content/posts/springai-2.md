+++
date = '2025-12-10T08:01:06+08:00'
draft = false
ShowToc = true
TocOpen = true
title = 'Spring AI全景指南(二) 大模型接入与“黑话”翻译器的构建'
+++

# Spring AI全景指南(二) 大模型接入与“黑话”翻译器的构建

在生成式AI技术栈逐渐标准化的今天，构建一个企业级AI应用已不再是Python及其生态的专利。对于广大的Java开发者而言，利用现有的Spring生态无缝接入大模型能力，是实现业务智能化的最优路径。

本系列文章将剥离繁杂的概念包装，从工程落地的角度出发，系统性地拆解基于Java和Spring的AI开发流程。我们将沿着“基础对话->知识增强->智能体工作流”的技术进阶路线，逐步构建复杂的AI系统。

**基础接入**：从最简单的“一问一答”开始，通过“办公室黑话翻译器”案例，掌握SpringAI的核心抽象与Prompt结构设计。

**RAG架构**：引入**向量数据库(Vector Store)**，通过SQL语义化检索示例，实现隐性知识的显性化与企业私有数据的挂载。

**智能体工作流(AgenticWorkflow)**：探索神经符号AI(Neuro-Symbolic AI)，通过工具调用(Tool Calling)与语义逻辑，实现模型的自修正与反思迭代，解决复杂SQL生成等高难度任务。

本文作为系列首篇，将聚焦于SpringAI的核心架构与基础对话能力的构建，默认读者具备JDBC、Spring、接口调用、Spring Boot等现代化Java开发的基础知识。

在传统的AI开发中，接入OpenAI、DeepSeek或通义千问(Qwen)往往意味着需要维护多套异构的HTTP Client代码。这种硬编码方式不仅导致系统与特定厂商强耦合，也增加了后续的模型迁移成本。

Spring AI沿袭了Spring框架一贯的“可移植服务抽象”(Portable Service Abstraction)设计哲学。它定义了一套通用的顶层接口(如ChatClient、EmbeddingClient)，将不同模型厂商的API差异屏蔽在框架实现层之下。

对于开发者而言，这不仅意味着代码层面的解耦，更意味着一种“配置驱动”的开发模式，只需更改配置文件中的参数，即可在不同模型之间无缝切换，从而在成本、性能与效果之间找到最佳平衡点。

Spring AI的核心交互组件是ChatClient。与简单的HTTP请求不同，大模型的交互依赖于结构化的提示词工程(Prompt Engineering)。在Spring AI中，这一结构被封装为Prompt对象，通常由两个关键角色构成：

**System Message(系统预设)**：用于设定AI的“人设”、行为边界与回复风格。它是模型的底层指令，权重最高。

**User Message(用户输入)**：来自终端用户的具体指令或问题。

为了演示基础对话能力的构建，我们摒弃枯燥的“HelloWorld”，以一个具备实际业务语义的场景——“办公室黑话翻译器”为例。

在该场景中，我们需要利用LLM的自然语言理解与生成能力，将用户输入的直白语言(如“我不想做这个需求”)转化为委婉、专业的职场术语(如“该需求与当前各条线的优先级对齐存在困难”)。

## 1.系统指令设计(System Prompt)

要实现这一功能，核心在于System Message的定义。我们需要告诉模型，它不再是一个通用的助手，而是一位“资深职场沟通专家”。我们需要在System Prompt中明确：

角色定义：职场沟通专家。

任务目标：将口语化表达转化为专业术语。

输出限制：仅输出翻译后的文本，不解释，不添加多余寒暄。

![提示词](/images/springai2-1.png)

## 2.代码实现逻辑

在 Spring AI 中，这一交互过程可以通过直观的接口调用来实现。开发者无需关注底层的HTTP请求与JSON拼接细节，只需构建包含 SystemMessage 和 UserMessage 的结构化 Prompt 对象，并将其直接传递给模型客户端(Chat Model)，即可获取生成的响应结果。

```yaml
custom:  
  coder-name: Qwen/Qwen2.5-Coder-7B-Instruct
  qwen-name: Qwen/Qwen3-8B
```

```java
@Configuration
public class AIConfig {
    
    private static final Logger log = LoggerFactory.getLogger(AIConfig.class);  

    @Value("${custom.coder-name}")    
    private String coderName;
    
    @Bean("qwenClient")    
    public ChatClient.Builder chatClientBuilder(ChatModel chatModel) {
        return ChatClient.builder(chatModel);    
    }
    
    @Bean("coderClient")
    public ChatClient.Builder dsChatClient(ChatClient.Builder builder) {
        log.info("加载代码生成模型 {}", coderName);        
        return builder.defaultSystem("你是一位PostgreSQL数据库专家，只输出SQL语句，不要输出Markdown格式，不要解释。").defaultOptions(OpenAiChatOptions.builder().model(coderName).temperature(0.1).maxTokens(4096).build());    
    }
}
```

```java
@Service
public class Chater {
    
    private final ChatClient chatClient;
    
    public Chater(@Qualifier("qwenClient") ChatClient.Builder builder) {        
        this.chatClient = builder.build();    
    }
    
    public String call(String expansionPrompt,String requirement){
        return chatClient.prompt().system(expansionPrompt).user(requirement).call().content();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public classOSGService{

    private static final Logger log = LoggerFactory.getLogger(OSGService.class);
    private final Chater chater;

    public Result<CommonData> sayItBetter(String requirement) {
        //调用LLM把表达美化一下
        String expansionPrompt = """
            你是一位在国企和互联网大厂都混迹超过10年的职场生存大师，深谙"废话文学"、"黑话"和"高情商甩锅"技巧。
            你的任务是将用户输入的【大白话/心里话】翻译成【高情商/职场专业话术】。
            【翻译原则】：                
            1. 情绪稳定：把愤怒转化为"关切"，把拒绝转化为"排期问题"，把不会转化为"需进一步调研"等等。                
            2. 用词考究：多用"颗粒度"、"对齐"、"底层逻辑"、"抓手"、"赋能"、"闭环"、"协同"、"资源置换"等词汇。                
            3. 语气委婉：绝对不要直接说"不"，要说类似于"原则上是可行的，但在具体落地层面..."。                
            4. 字数膨胀：原话越短，你翻译的要越长，显得思考深刻。  
            【示例】：                
            用户：这事我做不了。                
            你：基于当前的资源配比和项目优先级，该需求在落地层面存在一定的带宽瓶颈，建议我们重新对齐一下交付预期。   
            用户：甲方又在提傻傻的需求了。
            你：提出的建议非常有建设性，为我们打开了新的视角。不过考虑到系统架构的稳态和现有排期，我们需要在可研阶段做更深度的风险评估。                                
            请直接输出润色后的内容，不要输出多余的解释。                
        """;

        if (requirement.isBlank()){
            requirement = "本来用户应该说点话让你转换的，但是什么也没说，你说几句告诉他要他先说你才能说。俏皮一点，可爱一点，撒娇卖萌也行，别说一大堆，十几二十个字就行。";        
        }

        String translated = chater.call(expansionPrompt, requirement);
        log.info("用户输入已扩展 {}", translated);

        return Result.success(requirement,translated);    
    }
}
```

## 3.运行效果与解析

通过上述配置，当用户输入直白的拒绝或抱怨时，模型会基于预设的提示词进行语义重构。这种模式展示了LLM开发中最基础也最核心的范式：通过系统指令约束模型行为，利用通用大模型解决特定领域的文本处理问题。

通过“黑话翻译器”，我们实现了基础的模型接入与文本生成。然而，当前的系统存在一个显著的局限性：它的大脑是封闭的。

模型只能基于预训练的通用知识进行回答。如果我们需要AI回答关于“公司内部2025年Q1考勤制度”等私有领域的问题，它就会产生幻觉或无法回答。

为了解决这个问题，我们需要让AI拥有“查阅资料”的能力。在下一篇文章中，我们将引入向量数据库(Vector Database)，结合SQL语句的语义化示例，深入探讨RAG(检索增强生成)架构的落地实践，实现企业隐性知识的显性化。