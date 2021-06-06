---
title: "What is observability and why do we need it?"
date: 2021-06-06T17:01:48+04:00
lastmod: 2021-06-06T17:01:53+04:00
slug: "ww-observability"
draft: false
categories:
- Observability
tags:
- observability
---

## Topic
In this article, I want to discuss what is observability and why we need such practice.

## What is an observable system?
In plain words, observability is how well you understand the internal state of a system and its tendencies. Observability is mostly about answering the questions, that may arise during high loads in production. These questions may be asked either by business or technical people.

### Why should distributed systems be observable
It is very important to make your microservices observable, so bugs and other problems in production make minimal impact on your business. Microservices and distributes systems have one nuance - they are harder to debug. In essence, a monolith is easier to debug, so generally speaking, you should either develop observable microservices or maybe go with a monolith.

> Transparency arises from deliberate design and architecture. “Adding transparency” late in development is about as effective as “adding quality.” Maybe it can be done, but only with greater effort and cost than if it’d been built in from the beginning.
>
> ___Release It!: Design and Deploy Production-Ready Software by Michael T. Nygard___

#### Don't panic
Imagine that some incident took place and we need to fix that weird bug that affects end-users as soon as possible. What will your action plan look like? If the system is observable, you can look at the metrics, traces, and logs, so you can define the problem and fix it, while it affected minimum users and minimum money is lost because of that.

It is easier to find problems when your app is monolithic, but if you use microservice architecture, it becomes a problem, because there is no one app that produces hundreds of logs per second, but instead, you have a lot of pods of different services that communicate with each other and they are deployed to somewhere in kubernetes, so you don't know what the heck happens.

#### Panic
Another reason to apply observability practices in distributes systems is to be able to answer the questions of business people or analysts. Why our customers leave feedbacks that the system works slow? Which services are bottlenecks and should be optimized?

## Tools
Generally, we separate observability into 3 fundamental components: logs, metrics, and traces.

### Logs

#### You must write JSON logs!

Use it as a rule of thumb is totally fine. That's not news that it is very hard, if possible to parse and analyze multiline plain text logs. JSON is very common and it is much easier to run queries against, so do yourself a favor and write proper logs.

#### Write meaningful logs in English.

Surely it is much more convenient to see **"no such item with id (4293) found in repository"** instead of **"ölümüş yenə də işləmir"** or **"no funciona de nuevo"**.

#### Attach some useful variables to your logs.
Your logs can contain a lot of useful information that can help you to find out why the system works incorrectly. It can be user id, trace id, or some other information that can help you to find out the problem.

### Traces
You can think of a trace as a transactional event. A transaction can lay through multiple services and have a lot of spans.
Span is a fundamental block of a transaction, that can contain some information about events.

#### Tools
There are a bunch of useful tools for visualization of traces:

jaeger:
![jaeger](/jaeger-ui-example.png)

datadog:
![datadog](/datadog-ui-example.png)

gcp:
![gcp](/gcp-ui-example.png)

and many other ones.

What is shown in the pictures above is named a waterfall chart. It shows latencies, spans, and much more useful information.

You can fire up a system like this with minimal effort. Luckily most major cloud providers have some kind of tracing SaaS, so just check your provider integration.
I absolutely love Datadog, even though it is a bit expensive, but if you can afford it, you should give it a try. You can also use jaeger as your tracing backend.

Here a list of tools that can help to make your system more observable.
- [opentelemetry](https://opentelemetry.io)
- [jaeger](https://jaegertracing.io)
- [datadog](https://www.datadoghq.com)
- [gcp trace](https://cloud.google.com/trace)

## Conclusion
We need observability to solve complex tasks like finding bugs and bottlenecks in order to minimize the impact on our system.

Also, such practices can help us to be ready for questions that business people ask.


## P.S.
I will continue series of these articles about observability with reallife examples, so stay tuned!
