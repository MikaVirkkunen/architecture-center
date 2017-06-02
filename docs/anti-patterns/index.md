---
title: Optimize Performance for Cloud Applications
description: 
author: dragon119
---

# Performance antipatterns for cloud applications

A *performance antipattern* is a common practice that is likely to hurt scalability when an application is placed under pressure. These antipatterns are frequent causes of performance problems, discovered during our own internal development and in customer engagements. For each antipattern, we describe why the antipattern typically occurs, typical symptoms of the antipattern, and some techniques that you can use to resolve the problem. We have included sample code that illustrates both the antipattern and a suggested solution. 

Sometimes a cloud application behaves well during performance testing. Then when the application is released, and experiences real-world workloads, it starts behaving badly &mdash;  rejecting user requests, stalling, or throwing exceptions. The development team is then faced with two questions:

- Why didn't this behavior show up during testing?
- How do we fix it?

The answer to the first question is straightforward. It's very difficult in a test environment to simulate real users, their behavior patterns, and the volumes of work they might perform. The only completely sure way to understand how a system behaves under load is to observe it in production. To be clear, we aren't suggesting that you should skip performance testing. Performance tests are crucial for getting baseline performance metrics. But you must be prepared to observe and correct performance issues when they arise in the live system.

The answer to the second question, how to fix problem, is less straightforward. Any number of factors might contribute, and sometimes the problem only manifests under certain circumstances, which can be hard to reproduce. Instrumentation and logging are key to finding the root cause, but you also have to know to look for. These antipatterns are meant to give you a starting point for your investigation. 

- **[Busy Database][BusyDatabase]**. Offloading too much processing to a data store.

- **[Busy Front End][BusyFrontEnd]**. Moving resource-intensive tasks onto background threads.

- **[Chatty I/O][ChattyIO]**. Continually sending many small network requests.

- **[Extraneous Fetching][ExtraneousFetching]**. Retrieving more data than is needed, resulting in unnecessary I/O.

- **[Improper Instantiation][ImproperInstantiation]**. Repeatedly creating and destroying objects that are designed to be shared and reused.

- **[Monolithic Persistence][MonolithicPersistence]**. Using the same data store for data with very different usage patterns.

- **[No Caching][NoCaching]**. Failing to cache data.

- **[Synchronous I/O][SynchronousIO]**. Blocking the calling thread while I/O completes. 

[BusyDatabase]: ./busy-database/index.md
[BusyFrontEnd]: ./busy-front-end/index.md
[ChattyIO]: ./chatty-io/index.md
[ExtraneousFetching]: ./extraneous-fetching/index.md
[ImproperInstantiation]: ./improper-instantiation/index.md
[MonolithicPersistence]: ./monolithic-persistence/index.md
[NoCaching]: ./no-caching/index.md
[SynchronousIO]: ./synchronous-io/index.md