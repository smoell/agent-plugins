# Concurrent Operations

Process arrays and run operations in parallel with concurrency control.

## Map Operations

Process arrays with automatic concurrency control and completion policies:

**TypeScript:**

```typescript
const items = [1, 2, 3, 4, 5];

const results = await context.map(
  'process-items',
  items,
  async (ctx, item, index) => {
    return await ctx.step(`process-${index}`, async () => 
      processItem(item)
    );
  },
  {
    maxConcurrency: 3,
    completionConfig: {
      minSuccessful: 4,
      toleratedFailureCount: 1
    }
  }
);

results.throwIfError();
const allResults = results.getResults();
```

**Python:**

```python
# Note: process is decorated with @durable_step
from aws_durable_execution_sdk_python.config import MapConfig, CompletionConfig

items = [1, 2, 3, 4, 5]

def process_item(ctx: DurableContext, item: int, index: int, items: list):
    return ctx.step(process(item), name=f'process-{index}')

results = context.map(
    inputs=items,
    func=process_item,
    name='process-items',
    config=MapConfig(
        max_concurrency=3,
        completion_config=CompletionConfig(
            min_successful=4,
            tolerated_failure_count=1
        )
    )
)

results.throw_if_error()
all_results = results.get_results()
```

**Java:**

```java
import software.amazon.lambda.durable.config.MapConfig;
import software.amazon.lambda.durable.config.CompletionConfig;

List<Integer> items = List.of(1, 2, 3, 4, 5);

var results = ctx.map("process-items", items, ProcessResult.class,
    (item, index, childCtx) -> {
        return childCtx.step("process-" + index, Result.class,
            stepCtx -> processItem(item));
    },
    MapConfig.builder()
        .maxConcurrency(3)
        .completionConfig(CompletionConfig.minSuccessful(4))
        .build());

results.throwIfError();
var allResults = results.getResults();
```

## Parallel Operations

Run heterogeneous operations concurrently:

**TypeScript:**

```typescript
const results = await context.parallel(
  'parallel-ops',
  [
    {
      name: 'fetch-user',
      func: async (ctx) => ctx.step(async () => fetchUser(userId))
    },
    {
      name: 'fetch-orders',
      func: async (ctx) => ctx.step(async () => fetchOrders(userId))
    },
    {
      name: 'fetch-preferences',
      func: async (ctx) => ctx.step(async () => fetchPreferences(userId))
    }
  ],
  { maxConcurrency: 3 }
);

const [user, orders, preferences] = results.getResults();
```

**Python:**

```python
# Note: fetch_user, fetch_orders, fetch_preferences are decorated with @durable_step
from aws_durable_execution_sdk_python.config import ParallelConfig

def fetch_user_data(ctx: DurableContext):
    return ctx.step(fetch_user(user_id))

def fetch_orders_data(ctx: DurableContext):
    return ctx.step(fetch_orders(user_id))

def fetch_prefs_data(ctx: DurableContext):
    return ctx.step(fetch_preferences(user_id))

results = context.parallel(
    [fetch_user_data, fetch_orders_data, fetch_prefs_data],
    name='parallel-ops',
    config=ParallelConfig(max_concurrency=3)
)

user, orders, preferences = results.get_results()
```

**Java:**

```java
import software.amazon.lambda.durable.config.ParallelConfig;

// Use ctx.parallel() for heterogeneous operations
// AutoCloseable ensures all branches complete when exiting try block
try (var parallel = ctx.parallel("parallel-ops", ParallelConfig.builder().maxConcurrency(3).build())) {
    var f1 = parallel.branch("fetch-user", User.class, branchCtx -> 
        branchCtx.step("fetch-user-step", User.class, s -> fetchUser(userId)));
    var f2 = parallel.branch("fetch-orders", new TypeToken<List<Order>>() {}, branchCtx -> 
        branchCtx.step("fetch-orders-step", new TypeToken<List<Order>>() {}, s -> fetchOrders(userId)));
    var f3 = parallel.branch("fetch-preferences", Preferences.class, branchCtx -> 
        branchCtx.step("fetch-prefs-step", Preferences.class, s -> fetchPreferences(userId)));
    
    // Branches complete automatically when try block exits (calls close() -> get())
    // Access individual results
    var user = f1.get();
    var orders = f2.get();
    var preferences = f3.get();
}
```

## Completion Policies

### Minimum Successful

Require a minimum number of successful operations:

**TypeScript:**

```typescript
const results = await context.map(
  'process-batch',
  items,
  async (ctx, item, index) => ctx.step(async () => process(item)),
  {
    completionConfig: {
      minSuccessful: 8  // Need at least 8 successes
    }
  }
);
```

**Java:**

```java
var results = ctx.map("process-batch", items, Result.class,
    (item, index, childCtx) -> childCtx.step("process-" + index, Result.class, s -> process(item)),
    MapConfig.builder()
        .completionConfig(CompletionConfig.minSuccessful(8))  // Need at least 8 successes
        .build());
```

### Tolerated Failures

Allow a specific number of failures:

**TypeScript:**

```typescript
const results = await context.map(
  'process-batch',
  items,
  async (ctx, item, index) => ctx.step(async () => process(item)),
  {
    completionConfig: {
      toleratedFailureCount: 2  // Allow up to 2 failures
    }
  }
);
```

**Python:**

```python
results = context.map(
    inputs=items,
    func=process_item,
    name='process-batch',
    config=MapConfig(
        completion_config=CompletionConfig(
            tolerated_failure_count=2  # Allow up to 2 failures
        )
    )
)
```

**Java:**

```java
var results = ctx.map("process-batch", items, Result.class,
    (item, index, childCtx) -> childCtx.step("process-" + index, Result.class, s -> process(item)),
    MapConfig.builder()
        .completionConfig(CompletionConfig.builder()
            .toleratedFailureCount(2)  // Allow up to 2 failures
            .build())
        .build());
```

### Tolerated Failure Percentage

Allow a percentage of failures:

**TypeScript:**

```typescript
const results = await context.map(
  'process-batch',
  items,
  async (ctx, item, index) => ctx.step(async () => process(item)),
  {
    completionConfig: {
      toleratedFailurePercentage: 10  // Allow up to 10% failures
    }
  }
);
```

**Python:**

```python
results = context.map(
    inputs=items,
    func=process_item,
    config=MapConfig(
        completion_config=CompletionConfig(
            tolerated_failure_percentage=10
        )
    ),
    name='process-batch'
)
```

**Java:**

```java
// Java uses factory methods for common policies
var results = ctx.map("process-batch", items, Result.class,
    (item, index, childCtx) -> childCtx.step("process-" + index, Result.class, s -> process(item)),
    MapConfig.builder()
        .completionConfig(CompletionConfig.firstSuccessful())  // At least one succeeds
        .build());
// Or: CompletionConfig.allSuccessful() for all must succeed (default)
// Or: CompletionConfig.minSuccessful(N) for exactly N successes
```

## Batch Result Handling

### Check Status

**TypeScript:**

```typescript
const results = await context.map('process', items, processFunc);

console.log(results.status);           // 'COMPLETED' | 'FAILED'
console.log(results.totalCount);       // Total items
console.log(results.startedCount);     // Items started
console.log(results.successCount);     // Successful items
console.log(results.failureCount);     // Failed items
console.log(results.hasFailure());     // Boolean
```

**Python:**

```python
results = context.map(inputs=items, func=process_item, name='process')

print(results.status)           # InvocationStatus enum
print(results.total_count)      # Total items
print(results.started_count)    # Items started
print(results.success_count)    # Successful items
print(results.failure_count)    # Failed items
print(results.has_failure())    # Boolean
```

**Java:**

```java
var results = ctx.map("process", items, Result.class,
    (item, index, childCtx) -> childCtx.step("p-" + index, Result.class, s -> process(item)));

System.out.println(results.getStatus());        // OperationStatus enum
System.out.println(results.getTotalCount());    // Total items
System.out.println(results.getSuccessCount());  // Successful items
System.out.println(results.getErrors().size()); // Failed items
```

### Get Results

**TypeScript:**

```typescript
// Get all results (throws if any failed)
const allResults = results.getResults();

// Get successful results only
const successful = results.succeeded.map(item => item.result);

// Get failed items
const failed = results.failed.map(item => ({
  index: item.index,
  error: item.error
}));

// Get all items with status
const all = results.all.map(item => ({
  index: item.index,
  status: item.status,
  result: item.result,
  error: item.error
}));
```

**Python:**

```python
# Get all results (throws if any failed)
all_results = results.get_results()

# Get successful results only
successful = [item.result for item in results.get_succeeded()]

# Get failed items
failed = [{'index': item.index, 'error': item.error} 
          for item in results.get_failed()]

# Get all items with status
all_items = results.operations  # List of all operation results
```

**Java:**

```java
// Get all results (throws if any failed)
var allResults = results.getResults();

// Get successful results only
var succeeded = results.getResults();

// Get failed items
var errors = results.getErrors();

// Individual item results with success/failure status
for (var item : results.getResultItems()) {
    if (item.isSuccess()) {
        var result = item.getResult();
        // Process successful result
    } else {
        var error = item.getError();
        ctx.getLogger().error("Item {} failed: {}", item.getIndex(), error.getMessage());
    }
}
```

### Error Handling

**TypeScript:**

```typescript
const results = await context.map('process', items, processFunc);

if (results.hasFailure()) {
  context.logger.error('Some items failed', {
    failureCount: results.failureCount,
    failures: results.failed.map(f => f.index)
  });
  
  // Retry failed items
  const failedItems = results.failed.map(f => items[f.index]);
  await context.map('retry-failed', failedItems, processFunc);
}
```

**Python:**

```python
results = context.map(inputs=items, func=process_item, name='process')

if results.has_failure():
    context.logger.error('Some items failed', extra={
        'failure_count': results.failure_count,
        'failures': [f.index for f in results.get_failed()]
    })
    
    # Retry failed items
    failed_items = [items[f.index] for f in results.get_failed()]
    context.map(inputs=failed_items, func=process_item, name='retry-failed')
```

**Java:**

```java
var results = ctx.map("process", items, Result.class,
    (item, index, childCtx) -> childCtx.step("process-" + index, Result.class, s -> process(item)),
    MapConfig.builder().completionConfig(CompletionConfig.minSuccessful(3)).build());

// Errors in individual map items don't fail the entire map - check explicitly
if (!results.getErrors().isEmpty()) {
    ctx.getLogger().error("Some items failed. Error count: {}", results.getErrors().size());
    
    // Get failed indices for retry
    var failedIndices = new ArrayList<Integer>();
    for (var item : results.getResultItems()) {
        if (!item.isSuccess()) {
            failedIndices.add(item.getIndex());
        }
    }
    
    // Retry failed items
    var failedItems = failedIndices.stream().map(items::get).toList();
    ctx.map("retry-failed", failedItems, Result.class, 
        (item, index, childCtx) -> childCtx.step("retry-" + index, Result.class, s -> process(item)));
}
```

## Concurrency Control

### Fixed Concurrency

**TypeScript:**

```typescript
const results = await context.map(
  'process',
  items,
  processFunc,
  { maxConcurrency: 5 }  // Process 5 items at a time
);
```

**Python:**

```python
results = context.map(
    inputs=items,
    func=process_item,
    name='process',
    config=MapConfig(max_concurrency=5)  # Process 5 items at a time
)
```

**Java:**

```java
var results = ctx.map("process", items, Result.class,
    (item, index, childCtx) -> childCtx.step("process-" + index, Result.class, s -> process(item)),
    MapConfig.builder().maxConcurrency(5).build());  // Process 5 items at a time
```

### Dynamic Concurrency

Adjust based on item characteristics:

**TypeScript:**

```typescript
const results = await context.map(
  'process',
  items,
  async (ctx, item, index) => {
    // Heavy items get their own processing
    if (item.size > 1000) {
      return await ctx.step(`heavy-${index}`, async () => 
        processHeavy(item)
      );
    }
    
    // Light items can be batched
    return await ctx.step(`light-${index}`, async () => 
      processLight(item)
    );
  },
  { maxConcurrency: 10 }
);
```

**Python:**

```python
def process_item_by_size(ctx: DurableContext, item: dict, index: int, items: list):
    # Heavy items get their own processing
    if item['size'] > 1000:
        return ctx.step(
            lambda _: process_heavy(item),
            name=f'heavy-{index}'
        )
    
    # Light items can be batched
    return ctx.step(
        lambda _: process_light(item),
        name=f'light-{index}'
    )

results = context.map(
    inputs=items,
    func=process_item_by_size,
    name='process',
    config=MapConfig(max_concurrency=10)
)
```

**Java:**

```java
var results = ctx.map("process", items, Result.class,
    (item, index, childCtx) -> {
        // Heavy items get their own processing
        if (item.getSize() > 1000) {
            return childCtx.step("heavy-" + index, Result.class,
                s -> processHeavy(item));
        }
        
        // Light items can be batched
        return childCtx.step("light-" + index, Result.class,
            s -> processLight(item));
    },
    MapConfig.builder().maxConcurrency(10).build());
```

## Advanced Patterns

### Map with Callbacks

**TypeScript:**

```typescript
const results = await context.map(
  'process-with-approval',
  items,
  async (ctx, item, index) => {
    const processed = await ctx.step('process', async () => 
      process(item)
    );
    
    const approved = await ctx.waitForCallback(
      'approval',
      async (callbackId) => sendApproval(item, callbackId),
      { timeout: { hours: 24 } }
    );
    
    return { processed, approved };
  },
  { maxConcurrency: 3 }
);
```

**Python:**

```python
from aws_durable_execution_sdk_python.config import WaitForCallbackConfig

def process_with_approval(ctx: DurableContext, item: dict, index: int, items: list):
    # Note: process_step and send_approval_email are decorated with @durable_step
    processed = ctx.step(process_step(item))
    
    def submit_approval(callback_id: str, step_ctx):
        send_approval_email(item, callback_id)
    
    approved = ctx.wait_for_callback(
        submitter=submit_approval,
        name='approval',
        config=WaitForCallbackConfig(timeout=Duration.from_hours(24))
    )
    
    return {'processed': processed, 'approved': approved}

results = context.map(
    inputs=items,
    func=process_with_approval,
    name='process-with-approval',
    config=MapConfig(max_concurrency=3)
)
```

**Java:**

```java
import software.amazon.lambda.durable.config.WaitForCallbackConfig;
import software.amazon.lambda.durable.config.CallbackConfig;

var results = ctx.map("process-with-approval", items, ApprovalResult.class,
    (item, index, childCtx) -> {
        var processed = childCtx.step("process", ProcessResult.class,
            s -> process(item));
        
        var approved = childCtx.waitForCallback("approval", ApprovalData.class,
            (callbackId, s) -> sendApproval(item, callbackId),
            WaitForCallbackConfig.builder()
                .callbackConfig(CallbackConfig.builder()
                    .timeout(Duration.ofHours(24))
                    .build())
                .build());
        
        return new ApprovalResult(processed, approved);
    },
    MapConfig.builder().maxConcurrency(3).build());
```

### Nested Map Operations

**TypeScript:**

```typescript
const results = await context.map(
  'process-batches',
  batches,
  async (ctx, batch, batchIndex) => {
    return await ctx.map(
      `batch-${batchIndex}`,
      batch.items,
      async (itemCtx, item, itemIndex) => {
        return await itemCtx.step(async () => process(item));
      }
    );
  }
);
```

**Python:**

```python
def process_batch(ctx: DurableContext, batch: dict, batch_index: int, batches: list):
    def process_item_in_batch(item_ctx: DurableContext, item: dict, item_index: int, items: list):
        return item_ctx.step(lambda _: process(item), name=f'item-{item_index}')
    
    return ctx.map(
        inputs=batch['items'],
        func=process_item_in_batch,
        name=f'batch-{batch_index}'
    )

results = context.map(
    inputs=batches,
    func=process_batch,
    name='process-batches'
)
```

**Java:**

```java
var results = ctx.map("process-batches", batches, BatchResult.class,
    (batch, batchIndex, batchCtx) -> {
        var itemResults = batchCtx.map("batch-" + batchIndex, batch.getItems(), ItemResult.class,
            (item, itemIndex, itemCtx) -> {
                return itemCtx.step("item-" + itemIndex, ItemResult.class,
                    s -> process(item));
            });
        return new BatchResult(itemResults.getResults());
    });
```

### Map with Child Contexts

**TypeScript:**

```typescript
const results = await context.map(
  'complex-process',
  items,
  async (ctx, item, index) => {
    return await ctx.runInChildContext(`item-${index}`, async (childCtx) => {
      const validated = await childCtx.step('validate', async () => 
        validate(item)
      );
      
      await childCtx.wait({ seconds: 1 });
      
      const processed = await childCtx.step('process', async () => 
        process(validated)
      );
      
      return processed;
    });
  },
  { maxConcurrency: 5 }
);
```

**Python:**

```python
def complex_process_item(ctx: DurableContext, item: dict, index: int, items: list):
    def child_workflow(child_ctx: DurableContext):
        # Note: validate_step and process_step are decorated with @durable_step
        validated = child_ctx.step(validate_step(item))
        
        child_ctx.wait(duration=Duration.from_seconds(1), name='item-delay')
        
        processed = child_ctx.step(process_step(validated))
        
        return processed
    
    return ctx.run_in_child_context(
        func=child_workflow,
        name=f'item-{index}'
    )

results = context.map(
    inputs=items,
    func=complex_process_item,
    name='complex-process',
    config=MapConfig(max_concurrency=5)
)
```

**Java:**

```java
var results = ctx.map("complex-process", items, ProcessResult.class,
    (item, index, childCtx) -> {
        return childCtx.runInChildContext("item-" + index, ProcessResult.class, inner -> {
            var validated = inner.step("validate", Validated.class, s -> validate(item));
            inner.wait("item-delay", Duration.ofSeconds(1));
            return inner.step("process", ProcessResult.class, s -> process(validated));
        });
    },
    MapConfig.builder().maxConcurrency(5).build());
```

## Performance Optimization

### Batch Size Selection

**TypeScript:**

```typescript
// Small items: Higher concurrency
const results = await context.map(
  'small-items',
  smallItems,
  processFunc,
  { maxConcurrency: 20 }
);

// Large items: Lower concurrency
const results = await context.map(
  'large-items',
  largeItems,
  processFunc,
  { maxConcurrency: 3 }
);
```

**Python:**

```python
# Small items: Higher concurrency
results = context.map(
    inputs=small_items,
    func=process_item,
    name='small-items',
    config=MapConfig(max_concurrency=20)
)

# Large items: Lower concurrency
results = context.map(
    inputs=large_items,
    func=process_item,
    name='large-items',
    config=MapConfig(max_concurrency=3)
)
```

**Java:**

```java
// Small items: Higher concurrency
var smallResults = ctx.map("small-items", smallItems, Result.class,
    (item, index, childCtx) -> childCtx.step("p-" + index, Result.class, s -> process(item)),
    MapConfig.builder().maxConcurrency(20).build());

// Large items: Lower concurrency
var largeResults = ctx.map("large-items", largeItems, Result.class,
    (item, index, childCtx) -> childCtx.step("p-" + index, Result.class, s -> process(item)),
    MapConfig.builder().maxConcurrency(3).build());
```

### Early Termination

Use completion policies to stop early:

**TypeScript:**

```typescript
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
```

**Python:**

```python
results = context.map(
    inputs=candidates,
    func=lambda ctx, candidate, index, items: ctx.step(
        lambda _: check_match(candidate),
        name=f'check-{index}'
    ),
    name='find-match',
    config=MapConfig(
        completion_config=CompletionConfig(min_successful=1)  # Stop after first success
    )
)
```

**Java:**

```java
var results = ctx.map("find-match", candidates, Match.class,
    (candidate, index, childCtx) -> {
        return childCtx.step("check-" + index, Match.class,
            s -> checkMatch(candidate));
    },
    MapConfig.builder()
        .completionConfig(CompletionConfig.minSuccessful(1))  // Stop after first success
        .build());
```

## Best Practices

1. **Set appropriate maxConcurrency** based on downstream system capacity
2. **Use completion policies** to handle partial failures gracefully
3. **Name all operations** for debugging
4. **Handle batch results explicitly** - check for failures
5. **Consider retry strategies** for failed items
6. **Monitor concurrency limits** to avoid overwhelming systems
7. **Use child contexts** for complex per-item workflows
8. **Implement circuit breakers** for external service calls
