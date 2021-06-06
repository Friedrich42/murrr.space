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

## What is observability

In plain words observability is how well you understand internal state of a system and its tendencies. 

### What is observable system?

Observability is mostly about answering to questions, that arise during high loads in production and questions may be asked either by business or technical people.

#### Don't panic
Imagine that some incident took place and we need to fix that weird bug that affects end users as soon as possible. What will your action plan look like? If the system is observable, you can look at the metrics, traces and logs, so you can define the problem and fix it, while it affected to minimum users and minimum money is lost because of that.

It is easier to find problems when your app is monolithic, but if you use microservice architecture, it becomes a problem, because there is no one app that produces hundreds of logs per second, but instead you have a lot of pods somewhere in kubernetes and you don't know what the heck happens.

#### Panic
Another reason to use observability in distributes systems is to be able to answer the questions of business people or analysts. Why our customers leave feedbacks that the system works slow? Which services are bottlenecks and should be optimized?

### Conclusion

We need observability in order to solve complex tasks like finding bugs and bottlenecks.
