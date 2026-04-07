# Advanced Patterns

**Supported Languages:** TypeScript, Python, Java

Advanced techniques and patterns for sophisticated durable function workflows.

## Advanced GenAI Agent Patterns

### Agent with Reasoning and Dynamic Step Naming

**TypeScript:**

```typescript
export const handler = withDurableExecution(async (event, context: DurableContext) => {
  context.logger.info('Starting AI agent', { prompt: event.prompt });
  const messages = [{ role: 'user', content: event.prompt }];

  while (true) {
    // Invoke AI model with reasoning
    const { response, reasoning, tool } = await context.step(
      'invoke-model',
      async (stepCtx) => {
        stepCtx.logger.info('Invoking AI model', {
          messageCount: messages.length
        });
        return await invokeAIModel(messages);
      }
    );

    // Log AI's reasoning
    if (reasoning) {
      context.logger.debug('AI reasoning', { reasoning });
    }

    // If no tool needed, return response
    if (tool == null) {
      context.logger.info('AI agent completed - no tool needed');
      return response;
    }

    // Execute tool with dynamic step naming
    const toolResult = await context.step(
      `execute-tool-${tool.name}`,  // Dynamic step name
      async (stepCtx) => {
        stepCtx.logger.info('Executing tool', {
          toolName: tool.name,
          toolParams: tool.parameters
        });
        return await executeTool(tool, response);
      }
    );

    // Add result to conversation
    messages.push({
      role: 'assistant',
      content: toolResult,
    });

    context.logger.debug('Tool result added', {
      toolName: tool.name,
      resultLength: toolResult.length
    });
  }
});
```

**Python:**

```python
# Note: invoke_ai_model and execute_tool are decorated with @durable_step
@durable_execution
def handler(event: dict, context: DurableContext) -> str:
    context.logger.info('Starting AI agent', extra={'prompt': event['prompt']})
    messages = [{'role': 'user', 'content': event['prompt']}]

    while True:
        # Invoke AI model
        result = context.step(invoke_ai_model(messages))

        response = result['response']
        reasoning = result.get('reasoning')
        tool = result.get('tool')

        if reasoning:
            context.logger.debug('AI reasoning', extra={'reasoning': reasoning})

        if tool is None:
            context.logger.info('AI agent completed')
            return response

        # Execute tool with dynamic step naming
        tool_result = context.step(
            func=execute_tool(tool, response),
            name=f"execute-tool-{tool['name']}"
        )

        messages.append({'role': 'assistant', 'content': tool_result})
        context.logger.debug('Tool result added', extra={'tool': tool['name']})
```

**Java:**

```java
public class AgentHandler extends DurableHandler<AgentRequest, String> {
    @Override
    public String handleRequest(AgentRequest event, DurableContext ctx) {
        ctx.getLogger().info("Starting AI agent with prompt: {}", event.getPrompt());
        var messages = new ArrayList<>(List.of(Map.of("role", "user", "content", event.getPrompt())));
        
        while (true) {
            var result = ctx.step("invoke-model", ModelResult.class,
                stepCtx -> {
                    stepCtx.getLogger().info("Invoking AI model. Message count: {}", messages.size());
                    return invokeAIModel(messages);
                });
            
            if (result.getReasoning() != null) {
                ctx.getLogger().debug("AI reasoning: {}", result.getReasoning());
            }
            
            if (result.getTool() == null) {
                ctx.getLogger().info("AI agent completed - no tool needed");
                return result.getResponse();
            }
            
            // Execute tool with dynamic step naming
            var toolResult = ctx.step("execute-tool-" + result.getTool().getName(),
                String.class,
                stepCtx -> {
                    stepCtx.getLogger().info("Executing tool: {}", result.getTool().getName());
                    return executeTool(result.getTool(), result.getResponse());
                });
            
            messages.add(Map.of("role", "assistant", "content", toolResult));
            ctx.getLogger().debug("Tool result added. Tool: {}", result.getTool().getName());
        }
    }
}
```

## Step Semantics Deep Dive

### AtMostOncePerRetry vs AtLeastOncePerRetry

**TypeScript:**

```typescript
import { StepSemantics } from '@aws/durable-execution-sdk-js';

// AtMostOncePerRetry (DEFAULT) - For idempotent operations
// Step executes at most once per retry attempt
// If step fails partway through, it won't re-execute the same attempt
await context.step(
  'update-database',
  async () => {
    // This is idempotent - safe to retry
    return await updateUserRecord(userId, data);
  },
  { semantics: StepSemantics.AtMostOncePerRetry }
);

// AtLeastOncePerRetry - For operations that can execute multiple times
// Step may execute multiple times per retry attempt
// Use when idempotency is handled externally
await context.step(
  'send-notification',
  async () => {
    // External system handles deduplication
    return await sendEmail(email, message);
  },
  { semantics: StepSemantics.AtLeastOncePerRetry }
);
```

**Python:**

```python
from aws_durable_execution_sdk_python.config import StepConfig, StepSemantics

# AtMostOncePerRetry (DEFAULT) - For idempotent operations
# Step executes at most once per retry attempt
result = context.step(
    func=update_user_record(user_id, data),
    config=StepConfig(semantics=StepSemantics.AT_MOST_ONCE_PER_RETRY),
    name='update-database'
)

# AtLeastOncePerRetry - For operations that can execute multiple times
# Use when idempotency is handled externally
result = context.step(
    func=send_email(email, message),
    config=StepConfig(semantics=StepSemantics.AT_LEAST_ONCE_PER_RETRY),
    name='send-notification'
)
```

**Java:**

```java
import software.amazon.lambda.durable.config.StepSemantics;

// AT_MOST_ONCE_PER_RETRY (for non-idempotent operations)
var payment = ctx.step("charge", PaymentResult.class, 
    s -> chargeCard(amount),
    StepConfig.builder().stepSemantics(StepSemantics.AT_MOST_ONCE_PER_RETRY).build());

// AT_LEAST_ONCE (default - for idempotent operations)
var result = ctx.step("update-db", Result.class, 
    s -> updateRecord(data));
```

**When to use each:**

| Semantic                | Use When                      | Example Operations                                |
| ----------------------- | ----------------------------- | ------------------------------------------------- |
| **AtMostOncePerRetry**  | Operation is idempotent       | Database updates, API calls with idempotency keys |
| **AtLeastOncePerRetry** | External deduplication exists | Queuing systems, event streams                    |

## Async Operations with DurableFuture

Run operations asynchronously and await results later:

**Java:**

```java
// Run concurrent async steps and await all
var f1 = ctx.stepAsync("fetch-user", User.class, s -> fetchUser(userId));
var f2 = ctx.stepAsync("fetch-orders", new TypeToken<List<Order>>() {}, s -> fetchOrders(userId));
var f3 = ctx.stepAsync("fetch-prefs", Preferences.class, s -> fetchPreferences(userId));

// Wait for all to complete
DurableFuture.allOf(f1, f2, f3);

var user = f1.get();
var orders = f2.get();
var prefs = f3.get();
```

**Async Child Context:**

```java
var childFuture = ctx.runInChildContextAsync("process-batch", BatchResult.class, child -> {
    var validated = child.step("validate", ValidatedData.class, s -> validateBatch(data));
    child.wait("batch-delay", Duration.ofSeconds(1));
    return child.step("process", BatchResult.class, s -> processBatch(validated));
});
// ... do other work ...
var batchResult = childFuture.get();
```

**Async Map:**

```java
var future = ctx.mapAsync("process-items", items, ProcessResult.class,
    (item, index, childCtx) -> {
        return childCtx.step("process-" + index, Result.class,
            stepCtx -> processItem(item));
    },
    MapConfig.builder().maxConcurrency(5).build());
// ... do other work ...
var results = future.get();
```

## Completion Policies - Interaction and Combination

### Combining Multiple Constraints

Completion policies can be combined, and execution **stops when the first constraint is met**:

**TypeScript:**

```typescript
const results = await context.map(
  'process-items',
  items,
  processFunc,
  {
    completionConfig: {
      minSuccessful: 8,              // Need at least 8 successes
      toleratedFailureCount: 2,       // OR can tolerate 2 failures
      toleratedFailurePercentage: 20, // OR can tolerate 20% failures
    }
  }
);

// Execution stops when ANY of these conditions is met:
// 1. 8 successful items (minSuccessful reached)
// 2. 2 failures occur (toleratedFailureCount reached)
// 3. 20% of items fail (toleratedFailurePercentage reached)
```

**Python:**

```python
from aws_durable_execution_sdk_python.config import MapConfig, CompletionConfig

results = context.map(
    inputs=items,
    func=process_item,
    name='process-items',
    config=MapConfig(
        completion_config=CompletionConfig(
            min_successful=8,
            tolerated_failure_count=2,
            tolerated_failure_percentage=20
        )
    )
)

# Execution stops when ANY of these conditions is met:
# 1. 8 successful items (minSuccessful reached)
# 2. 2 failures occur (toleratedFailureCount reached)
# 3. 20% of items fail (toleratedFailurePercentage reached)
```

**Java:**

```java
var results = ctx.map("process-items", items, Result.class,
    (item, index, childCtx) -> childCtx.step("p-" + index, Result.class, s -> process(item)),
    MapConfig.builder()
        .completionConfig(CompletionConfig.builder()
            .minSuccessful(8)
            .toleratedFailureCount(2)
            .toleratedFailurePercentage(20)
            .build())
        .build());

// Execution stops when ANY of these conditions is met:
// 1. 8 successful items (minSuccessful reached)
// 2. 2 failures occur (toleratedFailureCount reached)
// 3. 20% of items fail (toleratedFailurePercentage reached)
```

### Understanding Stop Conditions

**Example with 10 items:**

```typescript
const items = Array.from({ length: 10 }, (_, i) => i);

const results = await context.map(
  'process',
  items,
  processFunc,
  {
    maxConcurrency: 3,
    completionConfig: {
      minSuccessful: 7,
      toleratedFailureCount: 3
    }
  }
);

// Scenario 1: 7 successes, 0 failures
// ✅ Stops after 7th success (minSuccessful reached)
// Remaining 3 items are not processed

// Scenario 2: 5 successes, 3 failures
// ❌ Stops after 3rd failure (toleratedFailureCount reached)
// Remaining 2 items are not processed
// results.throwIfError() will throw because minSuccessful not met

// Scenario 3: 7 successes, 2 failures
// ✅ Stops after 7th success (minSuccessful reached)
// 1 item not processed, but completion policy satisfied
```

**Python:**

```python
items = list(range(10))

results = context.map(
    inputs=items,
    func=process_item,
    name='process',
    config=MapConfig(
        max_concurrency=3,
        completion_config=CompletionConfig(
            min_successful=7,
            tolerated_failure_count=3
        )
    )
)

# Scenario 1: 7 successes, 0 failures
# ✅ Stops after 7th success (minSuccessful reached)
# Remaining 3 items are not processed

# Scenario 2: 5 successes, 3 failures
# ❌ Stops after 3rd failure (toleratedFailureCount reached)
# Remaining 2 items are not processed
# results.throw_if_error() will throw because minSuccessful not met

# Scenario 3: 7 successes, 2 failures
# ✅ Stops after 7th success (minSuccessful reached)
# 1 item not processed, but completion policy satisfied
```

**Java:**

```java
var items = IntStream.range(0, 10).boxed().toList();

var results = ctx.map("process", items, Result.class,
    (item, index, childCtx) -> childCtx.step("p-" + index, Result.class, s -> process(item)),
    MapConfig.builder()
        .maxConcurrency(3)
        .completionConfig(CompletionConfig.builder()
            .minSuccessful(7)
            .toleratedFailureCount(3)
            .build())
        .build());

// Scenario 1: 7 successes, 0 failures
// ✅ Stops after 7th success (minSuccessful reached)
// Remaining 3 items are not processed

// Scenario 2: 5 successes, 3 failures
// ❌ Stops after 3rd failure (toleratedFailureCount reached)
// Remaining 2 items are not processed
// results.throwIfError() will throw because minSuccessful not met

// Scenario 3: 7 successes, 2 failures
// ✅ Stops after 7th success (minSuccessful reached)
// 1 item not processed, but completion policy satisfied
```

### Early Termination Pattern

Use completion policies for early termination when searching:

**TypeScript:**

```typescript
// Stop after finding first match
const results = await context.map(
  'find-match',
  candidates,
  async (ctx, candidate) => {
    return await ctx.step(async () => checkMatch(candidate));
  },
  {
    completionConfig: {
      minSuccessful: 1  // Stop after first success
    }
  }
);

// Only one item processed (assuming first succeeds)
if (results.successCount > 0) {
  const match = results.getSucceeded()[0];
  context.logger.info('Found match', { match });
}
```

**Python:**

```python
# Stop after finding first match
results = context.map(
    inputs=candidates,
    func=lambda ctx, candidate, index, items: ctx.step(
        lambda _: check_match(candidate), 
        name=f'check-{index}'
    ),
    name='find-match',
    config=MapConfig(
        completion_config=CompletionConfig(min_successful=1)
    )
)

# Only one item processed (assuming first succeeds)
if results.success_count > 0:
    match = results.get_succeeded()[0]
    context.logger.info('Found match', extra={'match': match})
```

**Java:**

```java
// Stop after finding first match
var results = ctx.map("find-match", candidates, Match.class,
    (candidate, index, childCtx) -> {
        return childCtx.step("check-" + index, Match.class, 
            s -> checkMatch(candidate));
    },
    MapConfig.builder()
        .completionConfig(CompletionConfig.minSuccessful(1))
        .build());

// Only one item processed (assuming first succeeds)
if (results.getSuccessCount() > 0) {
    var match = results.getResults().get(0);
    ctx.getLogger().info("Found match: {}", match);
}
```

## Advanced Error Handling

For timeout handling (waitForCallback, Promise.race), conditional retries, and circuit breaker patterns, see [advanced-error-handling.md](advanced-error-handling.md).

## Advanced and Retry Strategies

For conditional retry strategies and circuit breaker patterns, see [advanced-error-handling.md](advanced-error-handling.md).

## Custom Serialization Patterns

### Class with Date Fields

**TypeScript:**

```typescript
import {
  createClassSerdesWithDates
} from '@aws/durable-execution-sdk-js';

class User {
  constructor(
    public name: string,
    public email: string,
    public createdAt: Date,
    public updatedAt: Date
  ) {}
}

const result = await context.step(
  'create-user',
  async () => new User('Alice', 'alice@example.com', new Date(), new Date()),
  {
    serdes: createClassSerdesWithDates(User, ['createdAt', 'updatedAt'])
  }
);

// result is properly deserialized User instance with Date objects
console.log(result.createdAt instanceof Date); // true
```

### Complex Object Graphs

**TypeScript:**

```typescript
import { createClassSerdes } from '@aws/durable-execution-sdk-js';

class Order {
  constructor(
    public id: string,
    public items: OrderItem[],
    public customer: Customer
  ) {}
}

class OrderItem {
  constructor(public sku: string, public quantity: number) {}
}

class Customer {
  constructor(public id: string, public name: string) {}
}

// Create serdes for each class
const orderSerdes = createClassSerdes(Order);
const itemSerdes = createClassSerdes(OrderItem);
const customerSerdes = createClassSerdes(Customer);

const result = await context.step(
  'process-order',
  async () => {
    const customer = new Customer('CUST-123', 'Alice');
    const items = [
      new OrderItem('SKU-001', 2),
      new OrderItem('SKU-002', 1)
    ];
    return new Order('ORD-456', items, customer);
  },
  { serdes: orderSerdes }
);
```

**Python:**

```python
from dataclasses import dataclass
from typing import List
from aws_durable_execution_sdk_python.serde import SerDes
import json

@dataclass
class Customer:
    id: str
    name: str

@dataclass
class OrderItem:
    sku: str
    quantity: int

@dataclass
class Order:
    id: str
    items: List[OrderItem]
    customer: Customer

# Custom SerDes implementation
class OrderSerDes(SerDes[Order]):
    def serialize(self, obj: Order) -> str:
        return json.dumps({
            'id': obj.id,
            'items': [{'sku': item.sku, 'quantity': item.quantity} for item in obj.items],
            'customer': {'id': obj.customer.id, 'name': obj.customer.name}
        })
    
    def deserialize(self, data: str) -> Order:
        d = json.loads(data)
        return Order(
            id=d['id'],
            items=[OrderItem(**item) for item in d['items']],
            customer=Customer(**d['customer'])
        )

# Use custom SerDes in step
result = context.step(
    func=lambda _: Order(
        id='ORD-456',
        items=[OrderItem('SKU-001', 2), OrderItem('SKU-002', 1)],
        customer=Customer('CUST-123', 'Alice')
    ),
    config=StepConfig(serdes=OrderSerDes()),
    name='process-order'
)
```

**Java:**

```java
import software.amazon.lambda.durable.serde.SerDes;
import com.fasterxml.jackson.databind.ObjectMapper;

// Custom SerDes interface implementation
public class UserSerDes implements SerDes<User> {
    private final ObjectMapper mapper = new ObjectMapper();
    
    @Override
    public String serialize(User user) throws Exception {
        return mapper.writeValueAsString(user);
    }
    
    @Override
    public User deserialize(String data) throws Exception {
        return mapper.readValue(data, User.class);
    }
}

// Use custom SerDes in step
var user = ctx.step("create-user", User.class, 
    s -> createUser(event),
    StepConfig.builder().serDes(new UserSerDes()).build());
```

## Nested Workflows

### Parent-Child Workflow Pattern

**TypeScript:**

```typescript
// Parent orchestrator
export const orchestrator = withDurableExecution(
  async (event, context: DurableContext) => {
    const childFunctionArn = process.env.CHILD_FUNCTION_ARN!;

    // Invoke child workflows in parallel
    const results = await context.parallel(
      'process-batches',
      [
        {
          name: 'batch-1',
          func: async (ctx) => ctx.invoke(
            'process-batch-1',
            childFunctionArn,
            { batch: event.batches[0] }
          )
        },
        {
          name: 'batch-2',
          func: async (ctx) => ctx.invoke(
            'process-batch-2',
            childFunctionArn,
            { batch: event.batches[1] }
          )
        }
      ]
    );

    return results.getResults();
  }
);

// Child worker
export const worker = withDurableExecution(
  async (event, context: DurableContext) => {
    const items = event.batch.items;

    const results = await context.map(
      'process-items',
      items,
      async (ctx, item) => {
        return await ctx.step(async () => processItem(item));
      }
    );

    return results.getResults();
  }
);
```

**Python:**

```python
import os

# Parent orchestrator
@durable_execution
def orchestrator(event: dict, context: DurableContext) -> list:
    child_function_arn = os.environ['CHILD_FUNCTION_ARN']
    
    # Note: invoke_child functions are decorated with @durable_step
    def invoke_batch_1(ctx: DurableContext):
        return ctx.invoke(
            target_function_arn=child_function_arn,
            payload={'batch': event['batches'][0]},
            name='process-batch-1'
        )
    
    def invoke_batch_2(ctx: DurableContext):
        return ctx.invoke(
            target_function_arn=child_function_arn,
            payload={'batch': event['batches'][1]},
            name='process-batch-2'
        )
    
    # Invoke child workflows in parallel
    results = context.parallel(
        [invoke_batch_1, invoke_batch_2],
        name='process-batches'
    )
    
    return results.get_results()

# Child worker
@durable_execution
def worker(event: dict, context: DurableContext) -> list:
    items = event['batch']['items']
    
    results = context.map(
        inputs=items,
        func=lambda ctx, item, index, items: ctx.step(
            lambda _: process_item(item), 
            name=f'process-{index}'
        ),
        name='process-items'
    )
    
    return results.get_results()
```

**Java:**

```java
// Parent orchestrator
public class OrchestratorHandler extends DurableHandler<BatchEvent, List<BatchResult>> {
    @Override
    public List<BatchResult> handleRequest(BatchEvent event, DurableContext ctx) {
        var childArn = System.getenv("CHILD_FUNCTION_ARN");
        
        // Invoke child workflows using invokeAsync (not stepAsync - cannot nest durable operations)
        var f1 = ctx.invokeAsync("batch-1", childArn, event.getBatches().get(0), BatchResult.class);
        var f2 = ctx.invokeAsync("batch-2", childArn, event.getBatches().get(1), BatchResult.class);
        
        // Wait for all child invocations to complete
        DurableFuture.allOf(f1, f2);
        
        return List.of(f1.get(), f2.get());
    }
}

// Child worker
public class WorkerHandler extends DurableHandler<BatchInput, List<Result>> {
    @Override
    public List<Result> handleRequest(BatchInput event, DurableContext ctx) {
        var items = event.getBatch().getItems();
        
        var results = ctx.map("process-items", items, Result.class,
            (item, index, childCtx) -> {
                return childCtx.step("process-" + index, Result.class,
                    stepCtx -> processItem(item));
            });
        
        return results.getResults();
    }
}
```

## Best Practices Summary

1. **Dynamic Step Naming**: Use template literals for dynamic operation names
2. **Structured Logging**: Log reasoning and context with each operation
3. **Error Handling**: See [advanced-error-handling.md](advanced-error-handling.md) for timeout, retry, and circuit breaker patterns
4. **Completion Policies**: Understand how combined constraints interact
5. **Custom Serialization**: Use proper serdes for complex objects
6. **Nested Workflows**: Use invoke for modular, composable architectures
