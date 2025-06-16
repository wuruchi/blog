+++
date = '2025-05-31T19:02:59+02:00'
draft = true
title = 'Start with the Message Schema'
categories = ["Event-Driven", "AWS", "SQS"]
+++

# Motivation

We've finally reached this point. We can no longer keep adding more operations into our API endpoint workflow. It may be because we are reaching the `API Gateway` limit or because we are feeling ashamed of having a request that takes more than `n` seconds where `n` is an integer that represents a time in seconds that has been steadily increasing in the last months. Either way, it is time to adopt the famous **Event-Driven approach** and implement an asynchronous workflow using our technology of choice. For this post purposes, that would be `AWS SQS`.

This is the first post in what I expect to be a series of posts about my limited experience dealing with Asynchronous Workflows implemented using `AWS SQS`. I am going to comment on what I consider are important aspects of the development process of such a workflow. I've seen similar posts in the past, I read some of them when I first attempted my implementation. They were Ok and I am grateful to the people that took their time to put them together; however, I missed a little bit more nuance when dealing with key concepts. I get that short pieces of content is the current trend but some ideas can't be effectively summarized that way.

Let's start.

## Try to define your Message Schema, also known as your Data Contract, so you might identify patterns that can help you take decisions on the Architecture of your solution. Then, iterate.

Decide if you are going to allow a single operation per message or if multiple operations are allowed in the same message. For example, let's say that we need to update some metadata using an API and that this API implements granular endpoints that allow us to update different properties of the metadata, perhaps this API also allow us to update a set of properties using only one request and I guess that is a common requirement on these kinds of services, but let's say that for this one, you need to using different endpoints for each property. Then, it can add more complexity on the producer side to force it to provide one message per each property that needs to be updated, it would be better if we can offer the producer to define the changes on each property in a single message. This can even be a feature if there is no need to update all properties at the time, perhaps the producer only needs to update one or two properties on its most common workflows. As you might suspect now, we must consider the producer side as well as the consumer side of the story when working on the messages Schema. Let's settle on this: On the consumer side many operations are possible, on the producer side many operations are possible. We want to give the producer the possible to summarize these operations into a single message.

Our choice will directly influente the Architecture of our Consumer and the configuration of our Queue. It directly influences the execution of your consumer which can result in a different choice of consumer integrations. The typical one is Lambda but if your operations are going to take more than 15 minutes, then you need to pick something else as a consumer, but if the operation is taking so much time perhaps that is a hint that tells you that something might be misaligned in your general idea. In Software Development the pieces of a system tend to fit together dependening, among others, in your choice of technology. If the pieces of your idea are not fitting together in a standard way and require some specialized handling, then your might be getting into uncharted terrory, pushing the human knowledge of System Design further, or you might be doing something wrong, trying to fit a square into a circle shaped hole. Chances are the second one is happening. Stop for a moment an reflect on your choices. Do you want to push them further? If you decide to push further, it is highly likely that the same question, in a more apparent way, will show up in your next iteration.

Take a look at your message Schema, now consider that it will be transmitted as a string value. It might look nice as a JSON but it needs to be converted to string before sending it as a message body from the producer. In a typical application this is not a problem but things are rarely typical in Software Development. Consider the types of the information you are transmitting. If you are dealing with floating point number, then you might face some precision problems depending on your choice of JSON Serializer. This is not a problem directly related to our subject but it is worth mentioning.

Add a version number to your message Schema in the same way that you would version an API endpoint. It is a simple way to maintain backwards compatibility in case you decide to change your message schema at some point.

Think about reserving one property to specify the `source` of the message. `AWS` provides good information about the context of the message, but there might be some more information that you need on the consumer side that needs to be provided by the producer. Picture this, the updates you are making need to reflect the identity of the original caller that triggered the producer action. Then, we need to pass some kind of user identifier and possibly other identification data.

If the purpose of the workflow is to update a `resource` then at least one thing should be clear from your communication schema and that is the unique `Key` that will identify the resource that is going to be updated as a result of processing the message.

Let's see one example of message schema and comment on some its relevant characteristics:

```json
{
    "version": "1",
    "source": {
        "userId": "some-user-id",
        "serviceId": "some-service-id"
    },
    "detail": {
        "data": {
            "key": "object-unique-identifier",
            // Property: Value pair, it represents a replace operation
            "domainId": "some-domain-id",
            "description": "some-new-description",
            // Some Values need to be arrays
            "dataOwners": {
                "replace": ["original.one", "owner.one@cimpress.com", "owner.two@cimpress.com"],
                // ignored if replace exists
                "add": ["owner.one@cimpress.com", "owner.two@cimpress.com"],
                // ignored if replace exists
                "remove": []
            },
            "stewards": {
                // `replace` is not present, `add` and `remove` are considered
                "add": ["user.one@cimpress.com", "user.two@cimpress.com"],
                "remove": ["user.three@cimpress.com"]
            },
            // example of only adding tags
            "tags": {
                "add": ["newtag"],
            },
            "slackChannels": {
                "replace": ["#panda-squad"],
            },
            "status": "internal",
            // A special case
            "promiseOfFreshness": {
                "duration": 1, // some number
                "filter": "Months", // Months, Days, Hours
                "referenceColumn": "column-name"
            },
        }
    }
}
```