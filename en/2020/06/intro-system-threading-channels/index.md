---
title: Exploring System.Threading.Channels
description: "This post aims to provide a practical explanation about System.Threading.Channels API and how it might be handy to implement pub/sub patterns to enable communication between Tasks"
date: 2020-07-06
language: en
tags: ["dotnet", "pubsub", "async" ]
---

# Motivation

One common situation that we face almost daily on our lives is the *producers x consumers*. It happens, for instance, when some package needs to be sent to someone. In order for the package to be delivered, it must go a long way between its *sender* and its *receiver* if we think about all the logistics regarding this. For both senders and receiver, it doesn't matter what happens in the way, only that the package must arrive to destination. In order for that to happen, a **channel** must be established so that the delivery requirements are fulfilled. 

That's what this .NET API is all about: *to provide a way so that producer Tasks can deliver messages to consumer ones*. It is available as a [Nuget package](https://www.nuget.org/packages/System.Threading.Channels) and can be used with .NET Standard(>= 1.3) and .NET Framework(>= 4.6).

---

# Deep diving

## Creating and opening channels

A **channel** can, therefore, be seen as a basic data structure to stablish a connection between producers and consumers Tasks. It is a *generic* structure and might be created with the following static factories from the class `System.Threading.Channels.Channel<T>`:

- `CreateBounded<T>(int capacity)`: this method creates a *bounded channel*. This type of channel has a limited *capacity*. That means that, when a channel is specified as bounded and has a capacity of, for instance, 5 objects, and a producer sends a 6th, it won't be able to receive this object. In that particular case, this type of channel might have some different behaviors that we are going to look further.
- `Channel<T> CreateBounded<T>(BoundedChannelOptions options)`: this method behaves exactly the same as the above, with the add of specifying some channel options that will tell how will this channel will behave when messages arrive in its full capacity. That is done through the `FullMode` enum:
  - `Wait`: here we block the thread until the channel has free space to complete a new write operation.
  - `DropNewest`: it drops the newest item written to the channel but not yet read, so that the new message can arrive.
  - `DropOldest`: in a similar way as the above, it does the same except that it is with the oldest item instead.
  - `DropWrite`: drops another item that is concurrently written to the channel. 
- `Channel<T> CreateUnbounded<T>()`: this method creates another type of channel, a *unbounded channel*. This type has no defined capacity. Unbounded channels might be more performatic, [according to these benchmarks](https://ndportmann.com/system-threading-channels/). However, this option has to be carefully used as one massive production of messages might consume a huge amount of the resources (RAM) available. It is a tradeoff that one must choose when deciding which type of channel to use: performance x safety. 
- `Channel<T> CreateUnbounded<T>(UnboundedChannelOptions options)`: just as the similar method from the bounded channels, this one defines options to unbounded ones. And one particular property is quite interesting:
  - `AllowSynchronousContinuations`: [according to the official docs](https://docs.microsoft.com/en-us/dotnet/api/system.threading.channels.channeloptions.allowsynchronouscontinuations?view=netcore-3.1#System_Threading_Channels_ChannelOptions_AllowSynchronousContinuations), this property allows that a producer is turned *temporary* into a consumer.

Both channel option classes inherit from `ChannelOptions` that, by its turn, has two more properties:
 - `SingleWriter`: with false as default, it enforces that only one write operation is executed at a time.
 - `SingleReader`: the same behavior as above, but regarding ro read operations.

 In the below example, we create a *bounded channel* that will receive integers with maximum capacity of 1 object.


```csharp
var channel = System.Threading.Channels.Channel.CreateBounded<int>(new BoundedChannelOptions(1)
{
    FullMode = BoundedChannelFullMode.Wait
});
```

## Closing channels

A channel is closed with a concept called *completion*. This resource is controlled by the channel *writers* and is enabled through the methods `Complete` and `TryComplete`. Both methods might receive an Exception as parameter to indicate that the channel was closed due some error.

That said, *readers* can always check if a channel is closed during a read operation with the property `channel.Completion.IsCompleted`.

## Producing and consuming messages

We are talking about producers, consumers, readers and writers and now we're going to take a look how all of that is done in a channel. Messages are produced by a `Writer` and consumed by a `Reader`, both properties of the `Channel<T>` class. There are some possibilities of how it is done:

1) `TryWrite/TryRead`: both methods try to read/write a message to a channel in a *synchronous* way. In case it is not possible, the message is dropped and return of the method is `false`, otherwise `true`. In the following example, we write to the console a message that was dropped by the channel. That also is to be clearer a little be further in an example where we use some delays to simulate a channel in its total capacity.

```csharp
for (int i = 0; ; i++)
{
    if (!channel.Writer.TryWrite(i))
    {
        Console.WriteLine($"Dropping {i}");
    }

    await Task.Delay(1000);
}
```

2) `WaitToWriteAsync/WaitToReadAsync`: both methods return a `ValueTask<bool>` and indicate if the channel is available for writing or reading. They're based on the `Options` we said earlier in this chapter. By default, these methods won't throw exceptions if the channel is closed, unless an Exception was passed as parameter to the channel completion. In the following example, the reader/consumer stays active while a loop is true, which might happen indefinitely if the channel is not completed. On the other hand, the channel will be inactive if no messages are arriving. The implementation along with the `TryRead`/`TryWrite` is interesting for the reason that in a scenario where multiple consumers are competing for a single message and the production of these massages is done in a random way, that message that is marked as available can be consumed more quickly by a concurrent consumer.

```csharp
while (await channel.Reader.WaitToReadAsync())
{
    if (channel.Reader.TryRead(out string msg))
    {                    
        Console.WriteLine(msg);
    }
    else
    {
        Console.WriteLine($"Message already {msg} consumed");
    }
}
```

3) `WriteAsync/ReadAsync`: are both the *asynchronous* implementation of `TryRead`/`TryWrite`, with the addend that they throw an `ChannelClosedException` if the channel is already completed by a writer. Due to that concern, one would rather use the `WaitTo` methods.

```csharp
private static async Task ReadAsync(Channel<int> channel, int delay)
{
    while (true)
    {
        var message = await channel.Reader.ReadAsync();
        Console.WriteLine($"Readed {message}");
        await Task.Delay(delay).ConfigureAwait(false);
    }
    // An exception will be thrown when the channel is closed
}
```


### Consuming all messages at once

Besides above methods, we still have the `ReadAllAsync`, which creates an `IAsyncEnumerable` of all available data for reading at the channel and allows us to iterate in an async manner over them.


### Back to `AllowSynchronousContinuations`

As said earlier, this property allows that a writer becomes temporary also a reader. Let's think in the following example: a consumer calls a reading operation in a channel without messages. Internally, this channel creates something like a callback that will be called when a message is written to it. By default, it is done in an async manner, queueing this callback invocation so that it is executed in a different thread than the producer's.

When this property is given as `true`, we're saying to the channel that this callback can be executed in a *synchronous* way, i.e., the same thread that wrote the message will read it. That might gives us some pros regarding to performance, but must be used carefully. If some kind of `lock`, for instance, is used on write, the callback might run while the resource is still locked, which might lead to unexpected behaviors on your program's execution.

---

# Examples

The below examples are available on [this repo](https://github.com/mviegas/ChannelsDemo). There we have some basic producer/consumer implementations interacting through a *channel* in an async and non-blocking way.

## Writing to a channel with its maximum capacity

Here we set different speeds of write and read so that the read occurs in moments where the channel is full.

```csharp
private static async Task ChannelOutOfCapacityExample()
{
    // Setting a read delay bigger than a wrote delay, so that we can see what happens when channels are "full"
    const int readDelay = 500;

    const int writeDelay = 100;

    // Creating a bounded channel with capacity of 1 
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1)
    {
        // Setting this property we say that when the channel is full and another item is dispatched to it, it should wait until the current item is processed to process the next
        FullMode = BoundedChannelFullMode.Wait
    });

    // Calling Task.Run so that the Channel.Writer executes in a different synchronization context than the Channel.Reader
    _ = Task.Run(async () =>
    {
        for (int i = 0; ; i++)
        {
            if (!channel.Writer.TryWrite(i))
            {
                ExtendedConsole.WriteLine($"Dropping {i}", ConsoleColor.Red);
            }

            await Task.Delay(writeDelay).ConfigureAwait(false);
        }
    });

    while (true)
    {
        var message = await channel.Reader.ReadAsync().ConfigureAwait(false);

        ExtendedConsole.WriteLine($"Readed {message}", ConsoleColor.Green);

        await Task.Delay(readDelay).ConfigureAwait(false);
    }
}
```

## Closing a channel avoiding exceptions on reading

Here we show the `WaitToReadAsync` in order to avoid exceptions on channel completion.

```csharp
private static async Task ChannelCompletedWithoutExceptionRaised()
{
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1)
    {
        FullMode = BoundedChannelFullMode.Wait
    });

    _ = Task.Run(async () =>
    {
        for (int i = 0; ; i++)
        {
            await channel.Writer.WriteAsync(i);
            ExtendedConsole.WriteLine($"Writing {i}", ConsoleColor.Blue);

            if (i == 10)
            {
                ExtendedConsole.WriteLine($"Writer: completing channel after 10 executions", ConsoleColor.Yellow);
                channel.Writer.TryComplete();
            }
        }
    });

    // Using WaitToRead, no exception is raised when channel is completed, unless it is explicit passed on completion
    while (await channel.Reader.WaitToReadAsync(default).ConfigureAwait(false))
    {
        if (channel.Reader.TryRead(out int msg))
        {
            ExtendedConsole.WriteLine($"Readed {msg}", ConsoleColor.Green);
        }
        else
        {
            Console.WriteLine($"Message already {msg} consumed");
        }
    }
}
```


---

## Competing consumers

In the below example, we have a channel with one single producer (writer) and three consumers (readers). That way, when a consumer is activated by the `WaitToReadAsync` method, there are also others consuming competing on the same channel and, therefore, the message which activated the read operation might already be consumed, what would lead to `TryRead` to return false.

```csharp
private static async Task ChannelWithCompetingConsumers()
{
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1)
    {
        FullMode = BoundedChannelFullMode.Wait
    });

    _ = Task.Run(async () =>
    {
        for (int i = 0; ; i++)
        {
            await channel.Writer.WriteAsync(i);
            ExtendedConsole.WriteLine($"Writing {i}", ConsoleColor.Blue);

            if (i == 50)
            {
                ExtendedConsole.WriteLine($"Writer: completing channel after 50 executions", ConsoleColor.Yellow);
                channel.Writer.TryComplete();
            }
        }
    });

    var firstConsumer = new Consumer<int>(channel.Reader);
    var secondConsumer = new Consumer<int>(channel.Reader);
    var thirdConsumer = new Consumer<int>(channel.Reader);

    await Task.WhenAll(
        firstConsumer.ConsumeAsync(default),
        secondConsumer.ConsumeAsync(default),
        thirdConsumer.ConsumeAsync(default)).ConfigureAwait(false);
}


private class Consumer<T>
{
    private readonly ChannelReader<T> _reader;
    public Guid Id { get; }
    private readonly Random _random = new Random(100);

    public Consumer(ChannelReader<T> reader)
    {
        Id = Guid.NewGuid();
        _reader = reader;
    }

    public async Task ConsumeAsync(CancellationToken cancellationToken)
    {
        // Using WaitToRead, no exception is raised when channel is completed, unless it is explicit passed on completion
        while (await _reader.WaitToReadAsync(cancellationToken))
        {
            if (_reader.TryRead(out T msg))
                ExtendedConsole.WriteLine($"ID {Id} === Readed {msg}", ConsoleColor.Green);
            else
                ExtendedConsole.WriteLine($"ID {Id} === Consumer awoken but message already consumed", ConsoleColor.Yellow);

            await Task.Delay(_random.Next(500));
        }
    }
}
```

# Wrap-up

**Channels** are really powerful abstractions that might be used to implement async and non-blocking pub/sub between Tasks. Its implementation aims to work in concurrent scenarios where performance and flexibility are required. They are a really interesting resource, specially when one needs to perform some background processing in a single application.

---

# References

- Working with Channels in .NET: https://www.youtube.com/watch?v=gT06qvQLtJ0

- An Introduction to System.Threading.Channels: https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/

- An Introduction to System.Threading.Channels (mesmo título, outro post): https://www.stevejgordon.co.uk/an-introduction-to-system-threading-channels

- Exploring System.Threading.Channels: https://ndportmann.com/system-threading-channels/