---
sidebar_position: 6
---

# AI Services

So far, we have been covering low-level components like `ChatLanguageModel`, `ChatMessage`, `ChatMemory`, etc.
Working at this level is very flexible and gives you total freedom, but it also forces you to write a lot of boilerplate code.
Since LLM-powered applications usually require not just a single component but multiple components working together
(e.g., prompt templates, chat memory, LLMs, output parsers, RAG components: embedding models and stores)
and often involve multiple interactions, orchestrating them all becomes even more cumbersome.

We want you to focus on business logic, not on low-level implementation details.
Thus, there are currently two high-level concepts in LangChain4j that can help with that: AI Services and Chains.

## Chains (legacy)

The concept of Chains originates from Python's LangChain (before the introduction of LCEL).
The idea is to have a `Chain` for each common use case, like a chatbot, RAG, etc.
Chains combine multiple low-level components and orchestrate interactions between them.
The main problem with them is that they are too rigid if you need to customize something.
LangChain4j has only two Chains implemented (`ConversationalChain` and `ConversationalRetrievalChain`),
and we do not plan to add more at this moment.

## AI Services

We propose another solution called AI Services, tailored for Java.
The idea is to hide the complexities of interacting with LLMs and other components behind a simple API.

This approach is very similar to Spring Data JPA or Retrofit: you declaratively define an interface with the desired API,
and LangChain4j provides an object (proxy) that implements this interface.
You can think of AI Service as a component of the service layer in your application.
It provides _AI_ services. Hence the name.

AI Services handle the most common operations:
- Formatting inputs for the LLM
- Parsing outputs from the LLM

They also support more advanced features:
- Chat memory
- Tools
- RAG

AI Services can be used to build stateful chatbots that facilitate back-and-forth interactions,
as well as to automate processes where each call to the LLM is isolated.

Let's take a look at the simplest possible AI Service. After that, we will explore more complex examples.

## Simplest AI Service

First, we define an interface with a single method, `chat`, which takes a `String` as input and returns a `String`.
```java
interface Assistant {

    String chat(String userMessage);
}
```

Then, we create our low-level components. These components will be used under the hood of our AI Service.
In this case, we just need the `ChatLanguageModel`:
```java
ChatLanguageModel model = OpenAiChatModel.withApiKey("demo");
```

Finally, we can use the `AiServices` class to create an instance of our AI Service:
```java
Assistant assistant = AiServices.create(Assistant.class, model);
```
:::note
In a Quarkus or Spring Boot application, this can be a bean that you can then inject into your code
wherever you need AI Services.
:::

Now we can use `Assistant`:
```java
String answer = assistant.chat("Hello");
System.out.println(answer); // Hello, how can I help you?
```

## How does it work?

You provide the `Class` of your interface to `AiServices` along with the low-level components,
and `AiServices` creates a proxy object implementing this interface.
Currently, it uses reflection, but we are considering alternatives as well.
This proxy object handles all the conversions for inputs and outputs.
In this case, the input is a single `String`, but we are using a `ChatLanguageModel` which takes `ChatMessage` as input.
So, `AiService` will automatically convert it into a `UserMessage` and invoke `ChatLanguageModel`.
Since the output type of the `chat` method is a `String`, after `ChatLanguageModel` returns `AiMessage`,
it will be converted into a `String` before being returned from the `chat` method.

## Using AI Services in Quarkus Application
[LangChain4j Quarkus extension](https://docs.quarkiverse.io/quarkus-langchain4j/dev/index.html)
greatly simplifies using AI Services in Quarkus applications.

More information can be found [here](https://docs.quarkiverse.io/quarkus-langchain4j/dev/ai-services.html).

## Using AI Services in Spring Boot Application
[LangChain4j Spring Boot starter](/tutorials/spring-boot-integration)
greatly simplifies using AI Services in Spring Boot applications.

## @SystemMessage

Now, let's look at a more complicated example.
We'll force the LLM reply using slang 😉

This is usually achieved by providing instructions in the `SystemMessage`.

```java
interface Friend {

    @SystemMessage("You are a good friend of mine. Answer using slang.")
    String chat(String userMessage);
}

Friend friend = AiServices.create(Friend.class, model);

String answer = friend.chat("Hello"); // Hey! What's up?
```

In this example, we have added the `@SystemMessage` annotation with a system prompt template we want to use.
This will be converted into a `SystemMessage` behind the scenes and sent to the LLM along with the `UserMessage`.

`@SystemMessage` can also load a prompt template from resources:
`@SystemMessage(fromResource = "my-prompt-template.txt")`

### System Message Provider
System messages can also be defined dynamically with the system message provider:
```java
Friend friend = AiServices.builder(Friend.class)
    .chatLanguageModel(model)
    .systemMessageProvider(chatMemoryId -> "You are a good friend of mine. Answer using slang.")
    .build();
```
As you can see, you can provide different system messages based on a chat memory ID (user or conversation).

## @UserMessage

Now, let's assume the model we use does not support system messages,
or maybe we just want to use `UserMessage` for that purpose.
```java
interface Friend {

    @UserMessage("You are a good friend of mine. Answer using slang. {{it}}")
    String chat(String userMessage);
}

Friend friend = AiServices.create(Friend.class, model);

String answer = friend.chat("Hello"); // Hey! What's shakin'?
```
We have replaced the `@SystemMessage` annotation with `@UserMessage`
and specified a prompt template with the variable `it` to refer to the only method argument.

`@UserMessage` can also load a prompt template from resources:
`@UserMessage(fromResource = "my-prompt-template.txt")`

Additionally, it's possible to annotate the `String userMessage` with `@V`
and assign a custom name to the prompt template variable:
```java
interface Friend {

    @UserMessage("You are a good friend of mine. Answer using slang. {{message}}")
    String chat(@V("message") String userMessage);
}
```

## Output Parsing (aka Structured Outputs)
If you want to receive a structured output from the LLM,
you can change the return type of your AI Service method from `String` to something else.
Currently, AI Services support the following return types:
- `String`
- `AiMessage`
- `boolean`/`Boolean`, if you need to get "yes" or "no" answer
- `byte`/`Byte`/`short`/`Short`/`int`/`Integer`/`BigInteger`/`long`/`Long`/`float`/`Float`/`double`/`Double`/`BigDecimal`
- `Date`/`LocalDate`/`LocalTime`/`LocalDateTime`
- `List<String>`/`Set<String>`, if you want to get the answer in the form of a list of bullet points
- Any `Enum`, `List<Enum>` and `Set<Enum>`, if you want to classify text, e.g. sentiment, user intent, etc.
- Any custom POJO
- `Result<T>`, if you need to access `TokenUsage`, `FinishReason`, sources (`Content`s retrieved during RAG) and executed tools, aside from `T`, which can be of any type listed above. For example: `Result<String>`, `Result<MyCustomPojo>`

Unless the return type is `String` or `AiMessage`, the AI Service will automatically append instructions
to the end of the `UserMessage` indicating the format in which the LLM should respond.
Before the method returns, the AI Service will parse the output of the LLM into the desired type.

You can observe appended instructions by [enabling logging](/tutorials/logging).

Now let's take a look at some examples.

### `boolean` as return type

```java
interface SentimentAnalyzer {

    @UserMessage("Does {{it}} has a positive sentiment?")
    boolean isPositive(String text);

}

SentimentAnalyzer sentimentAnalyzer = AiServices.create(SentimentAnalyzer.class, model);

boolean positive = sentimentAnalyzer.isPositive("It's wonderful!");
// true
```

### `Enum` as return type
```java
enum Priority {
    
    @Description("Critical issues such as payment gateway failures or security breaches.")
    CRITICAL,
    
    @Description("High-priority issues like major feature malfunctions or widespread outages.")
    HIGH,
    
    @Description("Low-priority issues such as minor bugs or cosmetic problems.")
    LOW
}

interface PriorityAnalyzer {
    
    @UserMessage("Analyze the priority of the following issue: {{it}}")
    Priority analyzePriority(String issueDescription);
}

PriorityAnalyzer priorityAnalyzer = AiServices.create(PriorityAnalyzer.class, model);

Priority priority = priorityAnalyzer.analyzePriority("The main payment gateway is down, and customers cannot process transactions.");
// CRITICAL
```

:::note
`@Description` annotation is optional. It's suggested to be used when enum names are not self-explanatory.
:::

### POJO as a return type
```java
class Person {
    String firstName;
    String lastName;
    LocalDate birthDate;
    Address address;
}

class Address {
    String street;
    Integer streetNumber;
    String city;
}

interface PersonExtractor {

    @UserMessage("Extract information about a person from {{it}}")
    Person extractPersonFrom(String text);
}

PersonExtractor personExtractor = AiServices.create(PersonExtractor.class, model);

String text = """
            In 1968, amidst the fading echoes of Independence Day,
            a child named John arrived under the calm evening sky.
            This newborn, bearing the surname Doe, marked the start of a new journey.
            He was welcomed into the world at 345 Whispering Pines Avenue
            a quaint street nestled in the heart of Springfield
            an abode that echoed with the gentle hum of suburban dreams and aspirations.
            """;

Person person = personExtractor.extractPersonFrom(text);

System.out.println(person); // // Person { firstName = "John", lastName = "Doe", birthDate = 1968-07-04, address = Address { ... } }
```

## JSON mode

When extracting custom POJOs (actually JSON, which is then parsed into the POJO),
it is recommended to enable a "JSON mode" in the model configuration.
This way, the LLM will be forced to produce valid JSON.

:::note
Please note that JSON mode and tools/function calling are similar features
but have different APIs and are used for distinct purposes.

JSON mode is useful when you _always_ need a response from the LLM in a structured format (valid JSON).
The schema for the expected JSON can be defined in a free form inside a `SystemMessage` or `UserMessage`.
In this scenario, the LLM must _always_ output valid JSON.
Additionally, there is usually no state/memory required,
so each interaction with the LLM is independent of others.
For instance, you might want to extract information from a text, such as the list of people mentioned in this text
or convert a free-form product review into a structured form with fields like
`String productName`, `Sentiment sentiment`, `List<String> claimedProblems`, etc.

On the other hand, the use of tool/function calling is useful when enabling the LLM to call or execute tools,
but only as necessary. In this case, a list of tools is provided to the LLM, and it autonomously decides
whether to call the tool.

Function calling is often used for structured data extraction,
but now we have the JSON mode feature, which is more suitable for this purpose.
:::

Here is how to enable JSON mode:

- For OpenAI:
  - For newer models that support [Structured Outputs](https://openai.com/index/introducing-structured-outputs-in) (e.g. gpt-4o-mini, gpt-4o-2024-08-06):
    ```java
    OpenAiChatModel.builder()
        ...
        .responseFormat("json_schema")
        .strictJsonSchema(true)
        .build();
    ```
    See more details [here](/integrations/language-models/open-ai#structured-outputs).
  - For older models (e.g. gpt-3.5-turbo, gpt-4):
    ```java
    OpenAiChatModel.builder()
        ...
        .responseFormat("json_object")
        .build();
    ```

- For Azure OpenAI:
```java
AzureOpenAiChatModel.builder()
    ...
    .responseFormat(new ChatCompletionsJsonResponseFormat())
    .build();
```

- For Vertex AI Gemini:
```java
VertexAiGeminiChatModel.builder()
    ...
    .responseMimeType("application/json")
    .build();
```

- For Google AI Gemini:
```java
GoogleAiGeminiChatModel.builder()
    ...
    .responseMimeType("application/json")
    .build();
```

- For Mistral AI:
```java
MistralAiChatModel.builder()
    ...
    .responseFormat(MistralAiResponseFormatType.JSON_OBJECT)
    .build();
```

- For Ollama:
```java
OllamaChatModel.builder()
    ...
    .format("json")
    .build();
```

- For other model providers: if the underlying model provider does not support JSON mode,
prompt engineering is your best bet. Also, try lowering the `temperature` for more determinism.

[More examples](https://github.com/langchain4j/langchain4j-examples/blob/main/other-examples/src/main/java/OtherServiceExamples.java)


## Streaming

The AI Service can [stream response](/tutorials/response-streaming) token-by-token
when using the `TokenStream` return type:
```java

interface Assistant {

    TokenStream chat(String message);
}

StreamingChatLanguageModel model = OpenAiStreamingChatModel.withApiKey(System.getenv("OPENAI_API_KEY"));

Assistant assistant = AiServices.create(Assistant.class, model);

TokenStream tokenStream = assistant.chat("Tell me a joke");

tokenStream.onNext(System.out::println)
    .onComplete(System.out::println)
    .onError(Throwable::printStackTrace)
    .start();
```

[Streaming example](https://github.com/langchain4j/langchain4j-examples/blob/main/other-examples/src/main/java/ServiceWithStreamingExample.java)


## Chat Memory

The AI Service can use [chat memory](/tutorials/chat-memory) in order to "remember" previous interactions:
```java
Assistant assistant = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .chatMemory(MessageWindowChatMemory.withMaxMessages(10))
    .build();
```
In this scenario, the same `ChatMemory` instance will be used for all invocations of the AI Service.
However, this approach will not work if you have multiple users,
as each user would require their own instance of `ChatMemory` to maintain their individual conversation.

The solution to this issue is to use `ChatMemoryProvider`:
```java

interface Assistant  {
    String chat(@MemoryId int memoryId, @UserMessage String message);
}

Assistant assistant = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .chatMemoryProvider(memoryId -> MessageWindowChatMemory.withMaxMessages(10))
    .build();

String answerToKlaus = assistant.chat(1, "Hello, my name is Klaus");
String answerToFrancine = assistant.chat(2, "Hello, my name is Francine");
```
In this scenario, two distinct instances of `ChatMemory` will be provided by `ChatMemoryProvider`, one for each memory ID.

:::note
Please note that if an AI Service method does not have a parameter annotated with `@MemoryId`,
the value of `memoryId` in `ChatMemoryProvider` will default to a string `"default"`.
:::

- [Example with a single ChatMemory](https://github.com/langchain4j/langchain4j-examples/blob/main/other-examples/src/main/java/ServiceWithMemoryExample.java)
- [Example with ChatMemory for each user](https://github.com/langchain4j/langchain4j-examples/blob/main/other-examples/src/main/java/ServiceWithMemoryForEachUserExample.java)
- [Example with a single persistent ChatMemory](https://github.com/langchain4j/langchain4j-examples/blob/main/other-examples/src/main/java/ServiceWithPersistentMemoryExample.java)
- [Example with persistent ChatMemory for each user](https://github.com/langchain4j/langchain4j-examples/blob/main/other-examples/src/main/java/ServiceWithPersistentMemoryForEachUserExample.java)


## Tools (Function Calling)

AI Service can be configured with tools that LLM can use:

```java

class Tools {
    
    @Tool
    int add(int a, int b) {
        return a + b;
    }

    @Tool
    int multiply(int a, int b) {
        return a * b;
    }
}

Assistant assistant = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .tools(new Tools())
    .build();

String answer = assistant.chat("What is 1+2 and 3*4?");
```
In this scenario, LLM will execute `add(1, 2)` and `multiply(3, 4)` methods before providing an answer.

More details about tools can be found [here](/tutorials/tools).


## RAG

AI Service can be configured with a `ContentRetriever` in order to enable RAG:
```java

EmbeddingStore embeddingStore  = ...
EmbeddingModel embeddingModel = ...

ContentRetriever contentRetriever = new EmbeddingStoreContentRetriever(embeddingStore, embeddingModel);

Assistant assistant = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .contentRetriever(contentRetriever)
    .build();
```

Configuring a `RetrievalAugmentor` provides even more flexibility,
enabling advanced RAG capabilities such as query transformation, re-ranking, etc:
```java
RetrievalAugmentor retrievalAugmentor = DefaultRetrievalAugmentor.builder()
        .queryTransformer(...)
        .queryRouter(...)
        .contentAggregator(...)
        .contentInjector(...)
        .executor(...)
        .build();

Assistant assistant = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .retrievalAugmentor(retrievalAugmentor)
    .build();
```

More details about RAG can be found [here](/tutorials/rag).

More RAG examples can be found [here](https://github.com/langchain4j/langchain4j-examples/tree/main/rag-examples/src/main/java).


## Auto-Moderation
[Example](https://github.com/langchain4j/langchain4j-examples/blob/main/other-examples/src/main/java/ServiceWithAutoModerationExample.java)


## Chaining multiple AI Services
The more complex the logic of your LLM-powered application becomes,
the more crucial it is to break it down into smaller parts, as is common practice in software development.

For instance, stuffing lots of instructions into the system prompt to account for all possible scenarios
is prone to errors and inefficiency. If there are too many instructions, LLMs may overlook some.
Additionally, the sequence in which instructions are presented matters, making the process even more challenging.

This principle also applies to tools, RAG, and model parameters such as `temperature`, `maxTokens`, etc.

Your chatbot likely doesn't need to be aware of every tool you have at all times.
For example, when a user simply greets the chatbot or says goodbye,
it is costly and sometimes even dangerous to give the LLM access to the dozens or hundreds of tools
(each tool included in the LLM call consumes a significant number of tokens)
and might lead to unintended results (LLMs can hallucinate or be manipulated to invoke a tool with unintended inputs).

Regarding RAG: similarly, there are times when it's necessary to provide some context to the LLM,
but not always, as it incurs additional costs (more context = more tokens)
and increases response times (more context = higher latency).

Concerning model parameters: in certain situations, you may need LLM to be highly deterministic,
so you would set a low `temperature`. In other cases, you might opt for a higher `temperature`, and so on.

The point is, smaller and more specific components are easier and cheaper to develop, test, maintain, and understand.

Another aspect to consider involves two extremes:
- Do you prefer your application to be highly deterministic,
where the application controls the flow and the LLM is just one of the components?
- Or do you want the LLM to have complete autonomy and drive your application?

Or perhaps a mix of both, depending on the situation?
All these options are possible when you decompose your application into smaller, more manageable parts.

AI Services can be used as and combined with regular (deterministic) software components:
- You can call one AI Service after another (aka chaining).
- You can use deterministic and LLM-powered `if`/`else` statements (AI Services can return a `boolean`).
- You can use deterministic and LLM-powered `switch` statements (AI Services can return an `enum`).
- You can use deterministic and LLM-powered `for`/`while` loops (AI Services can return `int` and other numerical types).
- You can mock an AI Service (since it is an interface) in your unit tests.
- You can integration test each AI Service in isolation.
- You can evaluate and find the optimal parameters for each AI Service separately.
- etc

Let's consider a simple example.
I want to build a chatbot for my company.
If a user greets the chatbot,
I want it to respond with the pre-defined greeting without relying on an LLM to generate the greeting.
If a user asks a question, I want the LLM to generate the response using internal knowledge base of the company (aka RAG).

Here is how this task can be decomposed into 2 separate AI Services:
```java
interface GreetingExpert {

    @UserMessage("Is the following text a greeting? Text: {{it}}")
    boolean isGreeting(String text);
}

interface ChatBot {

    @SystemMessage("You are a polite chatbot of a company called Miles of Smiles.")
    String reply(String userMessage);
}

class MilesOfSmiles {

    private final GreetingExpert greetingExpert;
    private final ChatBot chatBot;
    
    ...
    
    public String handle(String userMessage) {
        if (greetingExpert.isGreeting(userMessage)) {
            return "Greetings from Miles of Smiles! How can I make your day better?";
        } else {
            return chatBot.reply(userMessage);
        }
    }
}

GreetingExpert greetingExpert = AiServices.create(GreetingExpert.class, llama2);

ChatBot chatBot = AiServices.builder(ChatBot.class)
    .chatLanguageModel(gpt4)
    .contentRetriever(milesOfSmilesContentRetriever)
    .build();

MilesOfSmiles milesOfSmiles = new MilesOfSmiles(greetingExpert, chatBot);

String greeting = milesOfSmiles.handle("Hello");
System.out.println(greeting); // Greetings from Miles of Smiles! How can I make your day better?

String answer = milesOfSmiles.handle("Which services do you provide?");
System.out.println(answer); // At Miles of Smiles, we provide a wide range of services ...
```

Notice how we used the cheaper Llama2 for the simple task of identifying whether the text is a greeting or not,
and the more expensive GPT-4 with a content retriever (RAG) for a more complex task.

This is a very simple and somewhat naive example, but hopefully, it demonstrates the idea.

Now, I can mock both `GreetingExpert` and `ChatBot` and test `MilesOfSmiles` in isolation
Also, I can integration test `GreetingExpert` and `ChatBot` separately.
I can evaluate both of them separately and find the most optimal parameters for each subtask,
or, in the long run, even fine-tune a small specialized model for each specific subtask.


## Related Tutorials
- [LangChain4j AiServices Tutorial](https://www.sivalabs.in/langchain4j-ai-services-tutorial/) by [Siva](https://www.sivalabs.in/)