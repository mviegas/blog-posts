---
title: A brief brainstorm about the open-closed principle and event-driven architectures
date: 2020-06-05
language: en
tags: ["architecture", "design", "messaging", "ddd" ]
---

# **tl/dr**

The *open-closed principle*, one of the SOLID ones, states that one system/module/program should be "open for extension, but closed for modification." I've always found the statement itself quite vague. In this post, we are going to discuss in a little more practical way what it might mean from a practical point of view, both in code and architecture perspectives. We're going to talk about some specific concerns here, such as Domain-Driven Design, Pub/Sub and Message Brokers, so it might be useful to at least make yourself familiar with these concepts first.

# Giving meaning to what it seems obvious

The first question that came to my mind as I was first reading about the OCP was: how can we make a system/module/functionality open for extensions but closed to modifications? And why should we do it?

We can go from micro to macro in order to answer the question, so I will start with a classic: an oriented-object example. 

Let's think about a *chatbot*, for instance. There's one simple basic and starter thing that it must be able to do that is: to talk. And, as for every conversation, a mean must be provided between the two parties so that they can listen to one another and understand each other. If our chatbot is built around the HowsEverything messenger platform, we might have something like:

```csharp
public class Chatbot
{
    void Answer(Message message)
   { 
      // Some specific implementation about to deal with the HowsEverything platform
   }
}
```

With the above example, both a language and a channel are specified so that this conversation is stablished through the HowsEverything app. But imagine that someday in the future, my chatbot is required to expand to some other platforms like Letter or Nosekindle. As the logic is platform-specific, we would have to change its functionality in order to embrace this new requirement and, therefore, break the open-closed principle. But that's not what we want.

We can take a further step and make our chatbot platform-agnostic, so that it can be extensible to which platform it may arise as a new requirement. One possibility to do so is to introduce some abstraction to our code:

```csharp
public abstract class Chatbot
{
    void abstract Answer(Message message);
}

public class HowsEverythingChatbot
{
    void Answer(Message message) { // specific implementation }
}

public class NosekindleChatbot
{
    void Answer(Message message) { // specific implementation }
}
```

Doing that, our user might be able to always communicate with our chatbot in despite of the chosen platform. And we are also able to expand our chatbot from being platform-specific to being platform-agnostic.

Abstractions are a beautiful way to achieve the open-closed principle by code, but that was just an example of this principle from a code perspective for you to be familiar with it. Now let's jump to the architecture level.

# Extensibility with an Event-Driven Architecture

To think about design and architecture is a exercise that I encourage every developer to do. Whether to think about it up-front or embrace the evolutionary side of things (I know, both ways could and should work together, but there's still a lot of discussion about it). The fact is: a successful system will grow over time. If it will be to 10, 100, 1k, 10k or 100k users depends on its purpose. But it will grow, and you'll have to make sure that it's growing wealth. Therefore, one approach that I really like and to think about in large scale systems is an *event-driven architecture*. And the first things that I like to start thinking when I have to decided whether or not to follow this approach is to learn about my requirements and see if they fit as a rich domain with its **commands** and **events**.

We can think about those like this: every action has a consequence. We can thrive this axiom to the fact the every user action within a system has a result. Improving the language: **every successful command generates an event**. An UserSignUpCommand, where a new user is created, can be successfully stated as an UserSignedUpEvent. Note this event is an *confirmation that an intention has occurred in a correct manner*. And what does the commander do with that information? It simply tells someone that it is there, i.e., **a command raises an event** so that interested parties may listen to this event and do whatever they'd like to do with that piece of knowledge.

Well, now imagine that the first requirement about the above example was simple: *"we must notify the user that he his registration was successful"*. So after a command has been processed, we listen to an event and *handle* it so that, in this handle, we send an email notification. Looks quite reactive, and it is, but quite simple still, right? 

Now imagine that some more things happen:
- The marketing people should know about a new customers so they can start to track them in a customer funnel.
- The billing people should know about new customers, so that they can give him a free gift, like a 1-month subscription voucher.
- And so on ...

And now is where this kind of approach shines: **in an event-driven architecture we can have as many stakeholders as needed for a single event**. That means, that each new part that desires to do a new thing with this information must simply **subscribe** to an event, in order to **handle** it. And by doing so, we do not change a line, neither in our command nor in our first handler. Congratulations: you've just extended your system without moving a single character of code of one existing and working functionality.

But, how can we after a successful command, notify other parties about an event? Well, one approach is to follow the **pub/sub** pattern, where the application must simply **publish** this event to someone who's responsible to handle it to the interested parties, so that these events are correctly delivered and handled by these parties. It can be done in a bunch of different ways, but before deciding *how* we must decide *when*:
- Synchronously, with an in-memory [mediator](https://refactoring.guru/design-patterns/mediator). 
- Asynchronously, with an [event-bus](https://dzone.com/articles/design-patterns-event-bus) built around a [message broker](https://www.ibm.com/cloud/learn/message-brokers).

And there's a lot of room to talk about this kind of coupling (don't fool yourself, async processes are still coupled, but to time instead of space). But that's not the goal of this post.

Finally, with this approach, we accomplish exact what the open-closed principle states: we extend our system, enriching its behaviour, without modifying what's already there. And that's what this principle is about.

# Going further with Domain-Driven Design

Domain and event-driven systems share a lot of similarities and we can use both approaches to, not only better develop one specific domain, but also to communicate specific domain actions with N **aggregate roots** in the same application. In case you want to go more in-depth, Microsoft has a [really good article](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation) with even more references about the theme.

For instance, let's take a given **aggregate root** such as *Order* in an e-commerce system. A user can search goods, add them to cart and execute the checkout with a **PlaceOrderCommand**. This command, by its turn raises an **OrderCreatedEvent**. At last, this event might be handled by different subdomains to performs tasks related to:
- payments
- warehouse, supply chain and logistics
- order tracking by the user

As I first learned about this, my eyes shined and I thought that I was entering a whole new world. But of course, once that with great power comes great responsibility, one must also be aware that these approaches have a plenty of drawbacks, specially if the event distribution is made asynchronously. In such case, you'll probably have to dive into some patterns to deal with the distribution of events such as [outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html) and [event sourcing](https://microservices.io/patterns/data/event-sourcing.html). Beyond that, it is possible that you'll have to deal with [distributed transactions and sagas](https://microservices.io/patterns/data/saga.html).

# Outcome

The open-closed principle is one of the most important principles regarding the growth of a system. To achieve it through event-driven architectures looks like an almost perfect match, as it allows new functionalities to be added with no disruption of old ones as the domain expands. And by applying event-driven we can automatically apply the open-closed principle to the architectural level.

However, one should be aware that this approach carries with it a lot of advanced requirements and complexities, so one should really outweigh the pros and cons in order to decide if it fits its use case.

What do you think about this theme? Let me know in the comments!