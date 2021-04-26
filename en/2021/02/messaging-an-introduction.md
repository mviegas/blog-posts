---
title: 'Messaging: an Introduction'
published: true
description: 'Messaging is one of the pillars for distributed architectures to be sustainable. Here we will showcase basic theoretical concepts such as Messages and their types, as well as Transport Channels and their types.'
tags: ['distributedsystems','architecture','messaging','dotnet']
cover_image:
date: 2021-02-21
canonical_url: 'https://mateuscviegas.net/2021/02/messaging-an-introduction'
---

# *tl/dr*

This post is the first one of a series about basic concepts within distributed architectures. 

1) [Messaging: An introduction](https://mateuscviegas.net/2021/02/messaging-an-introduction)
2) Messaging: Point-to-point with System.Threading.Channels
3) Messaging: Idempotence and Delivery Types
4) Messaging: Consistency and Resiliency Patterns

The goal of this series is to provide new venturers on distributed systems world with basic concepts of messaging, the tradeoffs of this kind of resource and how to provide solutions for some of these tradeoffs in order to achieve the most important benefit of messaging: loose coupling and high scalability within a distributed architecture.

Messaging is one of the pillars for this kind of architecture to be sustainable. Here we will showcase basic theoretical concepts such as Messages and their types, as well as Transport Channels and their types. Also, we will give a high level overview about when you should or shouldn't consider messaging.

# Loose coupling, but why?

First things first: we can define messaging as a way of communication between two or more resources. In this manner, these resources communicate by sending and receiving *messages* through a *messaging infrastructure* instead of doing direct and synchronous requests to each other.

This is achieved by the exchange of *messages* via one or more *message channels*: a sender, which is some component within a system, writes messages to a channel inside this messaging infrastructure, and a receiver, another component, reads messages coming in via this channel. Thus, when messaging is introduced and sync communication is removed, we also remove direct coupling between these components. We can say then that these components are **loosely coupled**.

As pretty as it might seems, one question remains though: why would we want to make components within a system loosely coupled? To answer that question, let's understand one specific type of coupling that might be the greatest reason for using messaging and distributing a system.

## Temporal coupling

An application usually has some IO-bounded dependencies such as disks and networks. These dependencies might be databases, calls to external APIs, email sender services, file storages, or even some other external system, etc. Each time a request cross an IO boundary, it will have its response coupled to the result/response of this initiated IO operation. 

Even with modern concepts such as Task-based programming in C#, the so called *asynchronous* code is, de facto, async only from a thread perspective. This means, that applications like a web API, will still have the incoming requests waiting for each one of these async blocks of code to complete in order respond the client. The asynchronicity mentioned here relies on how these requests are executed from a thread perspective, meaning they are async because they do not lock the calling thread, a CPU resource, while IO operations are in place. What happens is that the application, when calling an `async` code, passes the responsibility of the execution continuation to another thread and frees up the calling one. This is actually really important, since there's a limited number of threads to execute CPU-bounded operations on the thread pool. Imagine the following situation:

- There's an incoming request;
- This request does a query to the database and return a response with the result;
    - Since database queries are IO operations, the calling thread might be blocked while awaiting for the database to execute the query. In this case, imagine that there are multiple requests coming in and that the database is, by some reason, overloaded and not responding properly. Since the number of CPU-bounded threads is limited, there might come a time when there's no more free threads to handle incoming requests: [starvation happens](https://www.geeksforgeeks.org/deadlock-starvation-and-livelock/). In this case, the application might be unresponsive or with a very high response times for new requests, since there are no more free threads to execute and it has to wait more threads to be available.
    - Now, imagine that these incoming requests access the database via an `async` method that executes the query. When that happens, a state machine is created with the request context and the control of this state machine is passed to a new thread, so that the CPU-bounded calling thread gets free to be used by any other incoming request. This another IO-bounded thread executes its operation on the database and, since it has this state machine, it is able to provide a *continuation* to the request through a *synchronisation context*, a.k.a., returning a response to the API with the query result. In this case, we give some buffer for our application performance, allowing our thread pool to have the incoming threads relying on CPU-bounded work only and being released more frequently to handle new requests.

Tl/dr: this whole explanation was to clarify that executions with the `async` keyword are, from a *temporal* perspective, still synchronous. The overall response time still depends on the database execution. Or, perhaps, on another external and unstable web API/system. And that's why this type of coupling is called **temporal coupling** and might be an issue if you need an application to have fast response times.

Messaging allows us to remove temporal coupling by deferring requests to be executed either in background or by third-party applications. In this case, a system component sends a message containing some sort of data through a channel and another system component, which is connected to this channel, receives this message and execute. Therefore, the component that sends the message is temporally coupled only to the message dispatching and not to an entire ecosystem of dependencies that there might exist.

# Message Types

Messages are chunks of data structured with both a *body* and also some *headers*. The body contains the central information and the message *payload*, and the headers might have some really useful metadata such as an *unique message identifier*, one or more *correlation identifiers* that could be used to correlate operations on different systems, etc. What's important to understand here is: messages are literally DTOs (data transfer objects). These DTOs are then serialised and sent via a channel. Either in JSON, binary, XML or any other format.

It is also common to think about messages from a semantic point of view, giving them *meaning*. This allows them to be categorised into types. Some of the most popular message types are *commands* and *events*. 

## Commands

Commands can be understood as requests specifications. A descriptive method name with zero or more parameters. Imagine, for instance, a `CreateUserCommand`. This command might have some parameters such as `Email` and `Password`. We could describe this message as the following class:

```csharp
public class CreateUserCommand
{
   public string Email { get; set; }
   public string Password { get; set; }
}
```

When this class is sent over a channel, the recipient system would then read this specification and execute the operation. The meaning of a command is basically *"do something with these attributes"*, in this case, this message is telling to the recipient: *"create a user with this email and this password"*.

## Events

On the other hand, we have *events*, that indicates and describe something that happened within the system. So for a `CreateUserCommand`, we could have a `UserCreatedEvent` describing a side-effect on our system, result from a previous operation. In this particular case, the event would be pretty similar to the command.

```csharp
public class UserCreatedEvent
{
   public string Email { get; set; }
}
```

You could read this as "an user was created with this email". This information could then be broadcasted via messaging, so that any other interested part of the system could receive this information and do something about it, like sending a welcome email, for instance.

# Channels

The [Enterprise Integration Patterns book](https://www.amazon.com/o/asin/0321200683/ref=nosim/enterpriseint-20) has the following description about channels: *"when an application has information to communicate, it doesn't just fling the information into the messaging system, it adds the information to a particular Message Channel. An application receiving information doesn't just pick it up at random from the messaging system; it retrieves the information from a particular Message Channel."*

![](https://www.enterpriseintegrationpatterns.com/img/MessageChannelSolution.gif)

There are mainly two types of message channels:

* Point-to-point: this type of channel delivers the message to exactly one, and only one, consumer that is listening to the channel. If there are multiple consumers, it means that they will concur for the incoming messages and each message will only be read by one consumer. To have multiple and competing consumers in a point-to-point channel could be useful to increase the throughput of the channel, if we have a lot of messages being sent to it, and is basically a way of scaling the message consumption. Commands are often sent through this type of channel.

* Publish-subscribe: this type of channel might have multiple consumers that receive, each one, a copy of the message. This type of channel is particularly good for some scenarios. With *pub/sub* we can easily move information around many interested parties with no temporal coupling between them. One system X can know what happens to other system Y by receiving messages sent by Y to a message channel that X, by its turn, is listening to. That way, we can easily move information around many components within a system with no tight coupling between them, making the integration process between these components smoother. Events are often sent through this type of channel.

# First trade-offs

Even though messaging is often referred as a way of integrating and remove coupling between distributed systems, we can use the same concept to extend functionality inside one single system. By using the concept of commands and events within an application, we can easily expand its features without changing existing ones. I talk a little bit more about it in an old post about the correlation between ["the open-closed principle and event-driven architectures"](https://dev.to/mviegas/a-brief-brainstorm-about-the-open-closed-principle-and-event-driven-architectures-323h).

However, just as anything regarding to software engineering and architecture, messaging has its trade-offs. By delegating the message handling to an external consumer via a message channel we introduce a **single point of failure**: the messaging infrastructure. A single point of failure, commonly named SPOF, it is a component within a system that, if fails, will stop the entire system of working. 

Therefore, since all the communication between the systems rely on a central piece of communication, the messaging infrastructure, it becomes a requirement that this piece has a very high level of availability. 

There are still many other trade-offs of applying messaging and distributing your system, such as maintenance, monitoring and development complexity introduced by many side-effects caused by the decoupling, and we will talk more in detail about them on the next posts of this series. 

# Wrap-up

Messages are a nice concept and are very fundamental when we talk about distributed systems and architectures. The concept itself is an abstraction, which implementations are diverse, as well as the possible infrastructure resources to handle it. 

In this introductory post, we showcased how we exchange temporal for loose coupling with messaging, and why it might be good. We also introduced basic message types, channel types and also discussed briefly about the consequences of introducing messaging to a system.

On the next post will showcase an abstraction of the concepts showcased here: we will remove temporal coupling in a web API by sending messages via channels created with C# and the *System.Threading.Channels* library.
