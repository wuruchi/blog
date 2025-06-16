+++
date = '2025-05-29T19:02:59+02:00'
draft = true
title = 'Comments on using an SQS Queue to implement an asynchronous workflow'
+++

# Motivation

At first, I considered writing some generic comments on the usage of this technology and avoid getting into implementation details that depend on some specific service. As I started working on this article I realized that the generic path only leads to more complexity and that useful examples are required to provide better explanations. Let's contextualize this article.

# Relevant Terminology

* Producer: The service that produces the messages according to the format already established.
* Consumer: The service that consumes the messages and applies the actions requested in the message.

## Start with your Message Schema

Try to define your Message Schema, also known as your Data Contract, so you might identify patterns that can help you take decisions on the Architecture of your solution. Then, iterate.

Decide if you are going to allow a single operation per message or if multiple operations are allowed in the same message. For example, let's say that we need to update some metadata using an API and that this API implements granular endpoints that allow us to update different properties of the metadata, perhaps this API also allow us to update a set of properties using only one request and I guess that is a common requirement on these kinds of services, but let's say that for this one, you need to using different endpoints for each property. Then, it can add more complexity on the producer side to force it to provide one message per each property that needs to be updated, it would be better if we can offer the producer to define the changes on each property in a single message. This can even be a feature if there is no need to update all properties at the time, perhaps the producer only needs to update one or two properties on its most common workflows. As you might suspect now, we must consider the producer side as well as the consumer side of the story when working on the messages Schema. Let's settle on this: On the consumer side many operations are possible, on the producer side many operations are possible. We want to give the producer the possible to summarize these operations into a single message.

Our choice will directly influente the Architecture of our Consumer and the configuration of our Queue. It directly influences the execution of your consumer which can result in a different choice of consumer integrations. The typical one is Lambda but if your operations are going to take more than 15 minutes, then you need to pick something else as a consumer, but if the operation is taking so much time perhaps that is a hint that tells you that something might be misaligned in your general idea. In Software Development the pieces of a system tend to fit together dependening, among others, in your choice of technology. If the pieces of your idea are not fitting together in a standard way and require some specialized handling, then your might be getting into uncharted terrory, pushing the human knowledge of System Design further, or you might be doing something wrong, trying to fit a square into a circle shaped hole. Chances are the second one is happening. Stop for a moment an reflect on your choices. Do you want to push them further? If you decide to push further, it is highly likely that the same question, in a more apparent way, will show up in your next iteration.

Take a look at your message Schema, now consider that it will be transmitted as a string value. It might look nice as a JSON but it needs to be converted to string before sending it as a message body from the producer. In a typical application this is not a problem.

Add a version number to your message Schema in the same way that you would version an API endpoint.

Think about reserving one property to specify the `source` of the message. `AWS` provides good information about the context of the message, but there might be some more information that you need on the consumer side. Picture this, the updates you are making need to reflect the identity of the original caller that triggered the producer action. Then, we need to pass some kind of identifier of this user and possibly other identification data. There is more nuance to a decision.

At least one thing should be clear from your communication Schema, that is the unique `Key` that will identify the resource (or group of resources) that is going to be affected by the processing of your message.

## Is FIFO required?

Typically, when updating a resource that implementes multiple properties, we need to maintain the order of updates. If you are using a `FIFO` queue then it is very important to define a `Key` that identifies that resource. This value will also act as the `MessageGroupId` that the producer will provide to the queue.

## If you know the maximum execution time of your consumer then you know the `Visibility Timeout`

## FIFO guarantees execution order for the Group but ultimately the responsibility falls on the Consumer

If you want to use `Batches`, then you need to be extra careful about `maxReceiveCount`. If you are willing to handle the extra complexity of multiple retries, then increase your `maxReceiveCount` to more than `1`.

## If FIFO depending on the MessageGroupId and Batching are enabled, then you cannot execute all messages in the batch concurrently

## What is your DLQ message handling strategy?

You might want to start manual.

## You need to keep a reliable registry of operations executed

You don't want to rely solely on CloudWatch Logs.

## Ultimately your Consumer is using some other service to perform operations. How much concurrency can that service handle?

## Define your concurrency at the Lambda Event Source Mapping level

Otherwise you will throttle your Lambda and potentially your other project's Lambdas.
