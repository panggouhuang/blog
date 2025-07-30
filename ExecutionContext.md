# Execution context

## HttpContextAccessor
We’re going to explore the execution context of an ASP.NET Core web service. If it’s your first time building a web service with ASP.NET Core and using the built‑in `HttpContextAccessor`, you’re probably wondering how it works. That’s because `HttpContextAccessor` is registered as a singleton service, while `HttpContext` is created per request. So, how can a singleton hold per‑request data?

Let’s dive into the `HttpContextAccessor` code:

```csharp
// Licensed to the .NET Foundation under one or more agreements.
// The .NET Foundation licenses this file to you under the MIT license.

using System.Diagnostics;

namespace Microsoft.AspNetCore.Http;

/// <summary>
/// Provides an implementation of <see cref="IHttpContextAccessor" /> based on the current execution context.
/// </summary>
[DebuggerDisplay("HttpContext = {HttpContext}")]
public class HttpContextAccessor : IHttpContextAccessor
{
    private static readonly AsyncLocal<HttpContextHolder> _httpContextCurrent = new AsyncLocal<HttpContextHolder>();

    /// <inheritdoc/>
    public HttpContext? HttpContext
    {
        get => _httpContextCurrent.Value?.Context;
        set
        {
            // Clear the current HttpContext trapped in the AsyncLocal, since it’s done.
            _httpContextCurrent.Value?.Context = null;

            if (value != null)
            {
                // Use an object indirection to hold the HttpContext in the AsyncLocal,
                // so it can be cleared in all ExecutionContexts when it’s cleared.
                _httpContextCurrent.Value = new HttpContextHolder { Context = value };
            }
        }
    }

    private sealed class HttpContextHolder
    {
        public HttpContext? Context;
    }
}
```

If we ignore `HttpContextHolder` (which is basically just a wrapper around `HttpContext`), we see that `HttpContextAccessor` uses a static `AsyncLocal<HttpContextHolder>` to store the current `HttpContext`. It might seem strange to use a static variable for something that changes on every request.

Before we get into `AsyncLocal`, let’s make sure we understand **async**.

## Asynchronous programming
For me, asynchronous programming is all about event‑based concurrency. [Concurrency is when two or more tasks can start, run, and complete in overlapping time periods, whereas parallelism is when tasks literally run at the same time.](https://stackoverflow.com/questions/1050222/what-is-the-difference-between-concurrency-and-parallelism) Event‑based means we don’t block and wait—instead, we treat operations as events and register callbacks to run when they finish.

### Event loop
The core idea is simple: register a callback for each event, and when an event fires, the loop invokes its callback. Your thread basically sits in a loop, waiting for events, then running their callbacks—and those callbacks can trigger new events:

```javascript
while (true) {
    const event = getNextEvent();
    event.callback();
}
```

Assuming `getNextEvent()` and `event` are already defined, the tricky part is capturing context in those callbacks. Callbacks aren’t just static functions—they often need access to the environment where they were created. Look at this C# example:

```csharp
class Program
{
    public Task ProcessAsyncWithContinue(int a)
    {
        return DoSomethingAsync(a).ContinueWith(t =>
        {
            Console.WriteLine($"Result: {t.Result}");
            Console.WriteLine(a);
        });
    }

    public async Task<int> DoSomethingAsync(int a)
    {
        await Task.Delay(500);
        return 100;
    }
}
```

The continuation after `DoSomethingAsync` prints two lines, but it also needs the value of `a`. In C#, lambdas capture their surrounding environment so you can access those variables later. Here, `a` is captured and remains available when the continuation runs. The `async`/`await` keywords are just syntactic sugar for this continuation pattern. Here’s an equivalent—but more readable—version:

```csharp
public async Task ProcessAsync(int a)
{
    var result = await DoSomethingAsync(a);
    Console.WriteLine($"Result: {result}");
    Console.WriteLine(a);
}
```

## AsyncLocal
Now, why do we need `AsyncLocal`? As we’ve seen, lambdas grab whatever variables are in scope so they can use them later. If you need something like `HttpContext` in a continuation, you’d have to pass it as a parameter everywhere:

```csharp
public async Task Foo(HttpContext ctx)
{
    await Bar(ctx);
}

public async Task Bar(HttpContext ctx)
{
    // …
}
```

That quickly clutters your method signatures.

This is where **AsyncLocal** comes in—it gives you an ambient context without passing parameters around. You can access certain data (e.g. `HttpContext`) from anywhere in your async flow.

- In a synchronous world, you’d use a thread‑local variable: operations run one after another, so you can safely set and clear.
- In an asynchronous world, execution can pause and resume, sometimes on different threads, but you still want the same context available in your continuations—possibly with multiple executions happening at once.

The core of `AsyncLocal` (which underpins .NET’s `ExecutionContext`) is pretty simple: mimic what closures do. Capture the context when you start, then restore it when you continue. Here’s a simplified Node.js‑style example assuming a single thread:

```typescript
class ExecutionContext {
    public static current: ExecutionContext = new ExecutionContext();
    public value: number;

    public static Capture(): ExecutionContext {
        return new ExecutionContext(ExecutionContext.current);
    }

    constructor(context?: ExecutionContext) {
        this.value = context ? context.value : 0;
    }

    public static Run(context: ExecutionContext, cb: () => void): void {
        const previous = ExecutionContext.current;
        if (previous === context) {
            cb();
            return;
        }
        ExecutionContext.current = context;
        cb();
        ExecutionContext.current = previous;
    }
}

async function timeout() {
    await new Promise(resolve => setTimeout(resolve, Math.random() * 100));
}

async function runWithContext(val: number) {
    const ec = ExecutionContext.Capture();
    ec.value = val;
    await timeout();
    ExecutionContext.Run(ec, () => {
        console.log(ExecutionContext.current.value);
    });
}

(async () => {
    await Promise.all([runWithContext(1), runWithContext(2), runWithContext(3)]);
})();
```

### Immutable
Every time we capture the current context with a snapshot and restore it in the callback, we must ensure the context is unchangeable; otherwise, it would be shared between multiple async flows, which breaks the notion of being **local**. ASP.NET Core achieves this by using copy-on-write semantics: it only copies the context when it’s modified. Since `ExecutionContext.Capture` and `ExecutionContext.Run` are on the hottest path and modifications to the `ExecutionContext` are rare, this strategy eliminates memory allocations in most cases while still maintaining lock-free behavior. The code below will always print out `42`. Check more details in [Stepen Toub's blog post](https://devblogs.microsoft.com/dotnet/how-async-await-really-works/#async/await-under-the-covers):

```csharp
var number = new AsyncLocal<int>();

number.Value = 42;
ThreadPool.QueueUserWorkItem(_ => Console.WriteLine(number.Value));
number.Value = 0;

Console.ReadLine();
```

### Async shared
If you’d like the value to be shared across all subsequent async flows after creation, store a reference type in the AsyncLocal<T> and then update its field when you want to change the value, like this:

```csharp
var number = new AsyncLocal<NumberHolder>();

number.Value = new NumberHolder() { Value = 42 };
ThreadPool.QueueUserWorkItem(_ => Console.WriteLine(number.Value.Value));
number.Value.Value = 0;

Console.ReadLine();

internal class NumberHolder
{
	public int Value { get; set; }
}
```
But with this approach, you must be careful and make sure the reference type is thread-safe, as it can be accessed from multiple threads at the same time.
