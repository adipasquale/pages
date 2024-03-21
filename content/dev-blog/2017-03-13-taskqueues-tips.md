---
title: Taskqueues tips
date: 2017-03-13
categories: en
tags: dev
---

I wrote a post listing a few tips to write a taskqueue while I was working at Drivy. You can find it here : [https://drivy.engineering/taskqueues-tips/](https://drivy.engineering/taskqueues-tips/).

As a TLDR; this post suggests :

- Writing re-entrant idempotent tasks that require the least number of arguments possible
- Avoid class-level calls
- Avoid mutable instance variables
- Specialize your workers for more efficiency
- Queues should be designed by speed and urgency, not semantically
- Read carefully your message broker's doc (Redis, RabbitMQ ...) and monitor it
- Also monitor your queues SLAs, your worker's memory and exceptions, and your CRON
- Check your DB connection pool

I also spoke about this in a meetup talk, you can find the slides here : [https://adipasquale.github.io/taskqueues-slides-2015](https://adipasquale.github.io/taskqueues-slides-2015)
