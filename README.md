# Dog Adoption Assistant

## script

- setup the `dog` DB behind the scenes using `init_db.sh` script.
- my terrible dog, Peanut
- meet Prancer
- start.spring.io (`adoptions`):  `OpenAI`, `PG VectorStore`, `Web`, `Actuator`, `MCP Client`, `Data JDBC`

- setup the dog adoption assistant. it's a controller with a single endpoint that takes an id and an inquiry: `@PostMapping("/{id}/inquire")`

- you can issue queries with `http`:

```shell
http --form POST http://localhost:8080/1/inquire question="Do you have any neurotic dogs?"
```
- it doesnt know how to help us. it thinks we want help with managing a neurotic dog. it doesn't know its meant to be an employee of our ficticious dog adoption agency
- give it a system prompt:

```text
You are an AI powered assistant to help people adopt a dog from the adoptions
agency named Pooch Palace with locations in Atlanta, Antwerp, Seoul, Tokyo, Singapore, Paris,s
Mumbai, New Delhi, Barcelona, San Francisco, and London. Information about the dogs availables
will be presented below. If there is no information, then return a polite response suggesting wes
don't have any dogs available.
```

- retry the question. it'll go better. but still, no Prancer

- introduce vector stores, RAG, `QuestionAnswerAdvisor`, advisors

- _much_ better! we have Prancer. but how would we adopt him? the very next question out of our mouths might be: "when can i pick him up?"
- unfortunately there are two problems: the next question around, it'll completely forget about us. _and_, worse, it won't be able to make an appointment for us.

- let's look at `ChatMemory`. Add a `Map<Long,PromptChatMemoryAdvisor>` to the controller.
- in the client, build a context-specific advisor:

```        
var promptChatMemoryAdvisor = this.chatMemory
        .computeIfAbsent(id, key -> PromptChatMemoryAdvisor.builder(new InMemoryChatMemory())
        .build()
);
```

- then make sure to add that advisor to the chain of advisors, last, _after_ the `QuestionAndAnswerAdvisor`.
- retry the query asking if it has neurotic dogs, and then, in a second question, ask if I could come to the location and see the dog.
- it should work, indicating that the chat is now conversational. great. but it still won't let us adopt the dog.
- give our AI some tools to do the work for us.

```java


@Component
class DogAdoptionAppointmentScheduler {

    private final ObjectMapper om;

    DogAdoptionAppointmentScheduler(ObjectMapper om) {
        this.om = om;
    }

    @Tool(description = "schedule an appointment to adopt a dog" +
            " at the Pooch Palace dog adoption agency")
    String scheduleDogAdoptionAppointment(
            @ToolParam(description = "the id of the dog") int id,
            @ToolParam(description = "the name of the dog") String name)
            throws Exception {
        var instant = Instant.now().plus(3, ChronoUnit.DAYS);
        System.out.println("confirming the appointment: " + instant);
        return om.writeValueAsString(instant);
    }
}


```

- point the `ChatClient` to those local tools by saying `ChatClientBuilder#defaultTools(scheduler)`:
- ask the model now to confirm the appointment: 

- ok, so clearly that's worked. we can adopt the dog. but, we hard wired the business logic into this service.
- let's refactor the code to factor out the appointment booking into another service, using Anthropic's Model Context Protocol (MCP):
- NB: we're not using Claude.ai, here!
- start.spring.io (`service`): `web`, `MCP Server`, etc.
- redefine the tool for scheduling an appointment there, registering a `ToolCallbackProvider`, but pointing to the scheduler.

```java

    @Bean
    ToolCallbackProvider dogAdoptionToolCallbackProvider(DogAdoptionAppointmentScheduler scheduler) {
        return MethodToolCallbackProvider.builder()
                .toolObjects(scheduler).build();
    }

```

- start the new service.
- then, back in the client, add the following: 

```shell
http --form POST http://localhost:8080/1/inquire question="fantastic. can you please confirm an appointment for Prancer?"
```

```java
    @Bean
    McpSyncClient mcpClient() {
        var mcpClient = McpClient
                .sync(new HttpClientSseClientTransport("http://localhost:8080"))
                .build();
        mcpClient.initialize();
        return mcpClient;
    } 
```

- wire in the `McpSyncClient` into the `ChatClient`, referencing the `McpSyncClient`:

```java

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        var mcpToolCallbackProvider = new SyncMcpToolCallbackProvider(mcpSyncClient);
        return builder.
                defaultTools(mcpToolCallbackProvider)
                // ... 
                .build();
    }
```

- now try everything, end to end! 