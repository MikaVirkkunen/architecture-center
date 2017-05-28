---
title: No Caching antipattern
description: 
author: dragon119
---

# No Caching antipattern

In a cloud application that handles many concurrent requests, repeatedly fetching the same data can reduce performance and scalability. 

## Problem description

When data is not cached, it can result reduce application performance for several reasons:

- The application repeatedly retrieves the same information from a resource that is expensive to access, in terms of I/O overhead or latency.
- The application uses processing resources to construct the same objects or data structures for multiple requests.
- The application makes excessive calls to a service that charges for consumption, or that has a service quota and throttles clients past a certain limit.

The following code uses Entity Framework to connect to a database. Every client request results in a call to the database, even if multiple requests are fetching exactly the same data. The cost of repeated requests, in terms of I/O overhead and data access charges, can accumulate quickly.

```csharp
public class PersonRepository : IPersonRepository
{
    public async Task<Person> GetAsync(int id)
    {
        using (var context = new AdventureWorksContext())
        {
            return await context.People
                .Where(p => p.Id == id)
                .FirstOrDefaultAsync()
                .ConfigureAwait(false);
        }
    }
}
```

You can find the complete sample [here][sample-app].

This antipattern typically occurs because:

- Directly reading and writing to a data store is the simplest implementation. Caching makes the code more complicated. 
- There is concern about the overhead of maintaining the accuracy and freshness of cached data.
- An application was migrated from an on-premises system, where network latency was not an issue, and the system ran on expensive high-performance hardware, so caching wasn't considered in the original design.
- Developers aren't aware that caching is a possibility in a given scenario. For example, developers may not think of using ETags when implementing a web API.
- The benefits and drawbacks of using a cache are not clearly understood.


## How to fix the problem

The most popular caching strategy is the *on-demand* or [*cache-aside*][cache-aside-pattern] strategy. This approach is suitable for data that changes frequently.

- On read, the application first tries to get data from the cache. If the data isn't in the cache, the application retrieves it from the data source and adds it to the cache.
- On write, the application writes the change directly to the data source and removes the old value from the cache. It will be retrieved and added to the cache the next time it is required.

Here is the previous example updated to use the Cache-Aside pattern. Notice that the `GetAsync` method in the repository now calls the `CacheService` class, rather than calling the database directly. 

```csharp
public class CachedPersonRepository : IPersonRepository
{
    private readonly PersonRepository _innerRepository;

    public CachedPersonRepository(PersonRepository innerRepository)
    {
        _innerRepository = innerRepository;
    }

    public async Task<Person> GetAsync(int id)
    {
        return await CacheService.GetAsync<Person>("p:" + id, () => _innerRepository.GetAsync(id)).ConfigureAwait(false);
    }
}
```

The `GetAsync` method in `CacheService` first tries to get the data from Azure Redis Cache. If the value isn't found in Redis Cache, the method invokes a lambda function that was passed to it by the caller. The lamba function is responsible for fetching the data from the database. This implementation decouples the repository from the choice of cache storage, and decouples the `CacheService` from the choice of database. 

```csharp
public class CacheService
{
    private static ConnectionMultiplexer _connection;

    public static async Task<T> GetAsync<T>(string key, Func<Task<T>> loadCache, double expirationTimeInMinutes)
    {
        IDatabase cache = Connection.GetDatabase();
        T value = await GetAsync<T>(cache, key).ConfigureAwait(false);
        if (value == null)
        {
            // Value was not found in the cache. Call the lambda to get the value from the database.
            value = await loadCache().ConfigureAwait(false);
            if (value != null)
            {
                // Add the value to the cache.
                await SetAsync(cache, key, value, expirationTimeInMinutes).ConfigureAwait(false);
            }
        }
        return value;
    }
}
```

## Considerations

- If the cache is unavailable, perhaps because of a transient failure, don't return an error to the client. Instead, fetch the data from the original data source. However, be aware that while the cache is being recovered, the original data store could be swamped with requests, resulting in timeouts and failed connections. (After all, this is one of the motivations for using a cache in the first place.) Use a technique such as the [Circuit Breaker pattern][circuit-breaker] to avoid overwhelming the data source.

- To prevent data from becoming stale, many caching solutions support configurable expiration periods, so that data is automatically removed from the cache after a specified interval. You may need to tune this value for your scenario. If the caching solution doesn't provide built-in expiration, you may need to implement a background process that sweeps that cache, to prevent it from growing without limits, and to purge stale data. 

- Applications that cache nonstatic data should be designed to support [eventual consistency][data-consistency-guidance].

- For web APIs, you can support client-side caching by including a Cache-Control in request and response messages, and using ETags to identify versions of objects. See [Considerations for optimizing client-side data access][client-cache].

- You don't have to cache entire entities. If most of an entity is static but only a small piece is changes frequently, cache the static elements and retrieve the dynamic elements from the data source. This approach can help to reduce the volume of I/O being performed against the data source.

- In some cases, it's useful to cache volatile data, if that data is short-lived. For example, consider a device that continually sends status updates. It might make sense to cache this information as it arrives, and not write it to a persistent store at all.  

- Besides caching data from an external data source, you can use caching to save the results of complex computations. However, you should instrument the application first to determine whether it's actually CPU-bound.

- It might be useful to prime the cache when the application start. Populate the cache with the data that is most likely to be used.

- Always include instrumentation that detects cache hits and cache misses. Use this information to tune caching policies, such what data to cache, and how long to hold data in the cache before it expires.

## How to detect the problem

A complete lack of caching can lead to poor response times when retrieving data due
to the latency when accessing a remote data store, increased contention in the data
store, and an associated lack of scalability as more users request data from the
store.

You can perform the following steps to help identify whether lack of caching is
causing performance problems in an application:

1. Review the application with the designers and developers. <<RBC: Again, this seems odd. To what end? What would they be looking for? Is this step necessary? Okay, below it talks about whether caching was included in the design. So, maybe it doesn't need to be added here and it's a given that's what you'd be reviewing since this is the no caching antipattern.>>

2. Instrument the application and monitor the live system to assess how frequently
the application retrieves data or calculates information, and the sources of this
data. <<RBC: The final clause seems awkward. Maybe a second sentence would be better? Are you really instrumenting and monitoring the sources of the data? Would saying "...how frequently the application retrieves data (and from which sources) or calculates information." work?>>

3. Profiling the application in a test environment to capture low level metrics about
the overhead associated with repeated data access operations or other frequently
performed calculations.

4. Perform load testing of each possible operation to identify how the system
responds under a normal workload and under duress.

5. If appropriate, examine the data access statistics for the underlying data stores
and review how frequently data requests are repeated. <<RBC: This seems very similar to #2, why would you instrument if you didn't look at the stats?>>

The following sections apply these steps to the sample application described earlier.

----------

**Note:** If you already have an insight into where problems might lie, you may be
able to skip some of these steps. However, you should avoid making unfounded or
biased assumptions. Performing a thorough analysis can sometimes lead to the
identification of unexpected causes of performance problems. The following sections
are formulated to help you to examine applications and services systematically.

----------

### Reviewing the application

If you are a designer or developer familiar with the structure of the application,
and you are aware that the application does not use caching, then this is often an
indication that adding a cache might be useful. To identify the information to cache,
you need to determine exactly which resources are likely to be accessed most
frequently. Slow changing or static reference data that is read frequently are good
initial candidates. This could be data retrieved from storage or returned from a web
application or remote service. However, all resource access should be verified to
determine which resources are most suitable. Depending on the nature of the
application, fast-moving data that is written frequently may also benefit from
caching (see the considerations in the [How to correct the problem](#HowToCorrectTheProblem) section for more information.) <<RBC: One of the things I like about this doc set is the lack of internal linking. I find excessive internal linking annoying, especially in such a short doc. Isn't it obvious that the issue is discussed in the how to solve the problem section? Is this sentence needed?>>

----------

**Note:** Remember that a cached resource does not have to be a piece of data
retrieved from a data store. t could also be the results of an often repeated
computation.

----------

### Instrumenting the application and monitoring the live system

You should instrument the system and monitor it to provide more
information about the specific requests that users make while the application is in
production. <<RBC: I removed one of the "in production" instances becaue it seemed redundant.>> You can then analyze the results to group them by operation. You can use
lightweight logging  frameworks such as [NLog][NLog] or [Log4Net][Log4Net] to gather
this information. You can also deploy more powerful tools such as [Microsoft Application Insights][AppInsights], [New Relic][NewRelic], or
[AppDynamics][AppDynamics] to collect and analyze instrumentation information.

----------

**Note:** Remember that instrumentation imposes overhead in a
system, how much depends on the instrumentation strategy used and the tools adopted.

----------

As an example, if you configure the CachingDemo to capture monitoring data using
[New Relic][NewRelic], the analytics generated can quickly show you the frequency
with which each server request occurs, as shown by the image below. In this case, the
only HTTP GET operation performed is `Person/GetAsync`, the load test simply repeats
this same request each time. But in a live environment knowing the relative
frequency with which each request is performed gives you an insight into which
resources might best be cached.

![New Relic showing server requests for the CachingDemo application][NewRelic-server-requests]

### Profiling the application

If you require a deeper analysis of the application you can use a profiler to capture
and examine low level performance information in a controlled environment (not the
production system). You should examine metrics such as I/O request rates, memory
usage, and CPU utilization. Performing a detailed profiling of the system may reveal
a large number of requests to a data store or service, or repeated processing that
performs the same calculation. You can sort the profile data by the number of
requests to help identify candidate information for caching.

Note that there may be some restrictions on the environment you can perform
profiling in. For example, you may not be able to profile an application running as an
Azure Website. In these situations, you will need to profile the application running
locally. Furthermore, profiling might not provide the same data under the same load
as a production system, and the results can be skewed as a result of the additional
profiling overhead. However, you may be able to adjust the results to take this
overhead into account.

### Load testing the application

Performing load testing in a test environment can help to highlight any problems.
Load testing should simulate the pattern of data access observed in the production
environment using realistic workloads. You can use a tool such as [Visual Studio Online][VisualStudioOnline] to run load tests and examine the results.

The following graph was generated from load testing the CachingDemo sample
application without using caching. The load test simulates a step load of up to 800
users performing a typical series of operations against the system. Notice that the
capacity of the system&mdash;as measured by the number of successful tests performed each
second&mdash;reaches a plateau and additional requests are slowed as a result. The
average test time steadily increases with the workload. The response time levels off
once the user load remains constant towards the end of the test.

![Performance load test results for the uncached scenario][Performance-Load-Test-Results-Uncached]

----------

**Note:** This graph illustrates the general profile of a system running without
using caching. The throughput and response times are a function of the edition of
Azure SQL Database being used. Using the Premium service tier for the database will
likely display more exaggerated results than utilizing a lower service tier due to
the additional performance capabilities.

Additionally, the scenario itself is very simplified to highlight the general
performance pattern. A production environment with a more mixed workload should
generate a similar pattern, but the results might be more or less magnified.

----------

### Examining data access statistics

Examining data access statistics and other information provided by a data store
acting as the repository can yield some useful information, such as which queries
being repeated most frequently. For example, Microsoft SQL Server provides the
`sys.dm_exec_query_stats` management view which contains statistical information for
recently executed queries. The text for each query is available in the
`sys.dm_exec-query_plan` view. You can use a tool such as SQL Server Management
Studio to run the following SQL query and determine the frequency with which queries
are performed.

**SQL**
```SQL
SELECT UseCounts, Text, Query_Plan
FROM sys.dm_exec_cached_plans
CROSS APPLY sys.dm_exec_sql_text(plan_handle)
CROSS APPLY sys.dm_exec_query_plan(plan_handle)
```

The `UseCount` column in the results indicates how frequently each query is run. In
the following image, the third query has been run 256049 times. This is significantly
more than any other query.

![Results of querying the dynamic management views in SQL Server Management Server][Dynamic-Management-Views]

The text of this query is:

**SQL**
```SQL
(@p__linq__0 int)SELECT TOP (2)
[Extent1].[BusinessEntityId] AS [BusinessEntityId],
[Extent1].[FirstName] AS [FirstName],
[Extent1].[LastName] AS [LastName]
FROM [Person].[Person] AS [Extent1]
WHERE [Extent1].[BusinessEntityId] = @p__linq__0
```

In the [CachingDemo sample application][sample-app] shown earlier
this query is the result of the request generated by Entity Framework. This query is
repeated each time the `GetByIdAsync` method runs. The value of the `id` parameter
passed in to this method replaces the `p__linq__0` parameter in the SQL query.

----------

**Note:** In the CachingDemo sample application, performing load testing
in conjunction with examining the data access statistics of SQL Server arguably
provides sufficient information to enable you to determine which information should
be cached. In other situations, the repository might not provide the same level of
information, so additional instrumentation and profiling might be necessary, as
described in the following sections.

----------


## Consequences of the solution
Implementing caching may lead to a lack of immediate consistency, applications may
not always read the most recent version of a data item. Applications should be
designed around the principle of eventual consistency and tolerate being presented
with old data. Applying time-based eviction policies to the cache can help prevent
cached data from becoming too stale, but any such policy must be balanced against the
expected stability <<RBC: Volatility isn't used much on MSDN, I thought of replacing with instability, but stability seems to work. Is it okay technically?>> of the data. Data that is highly static and read often can reside
in the cache for longer periods than dynamic information that may become stale
quickly and which should be evicted more frequently.

To determine the effectiveness of any caching strategy, you should repeat load testing
after incorporating a cache and compare the results to those generated before the
cache was added. The following results show the graph generated by load testing the
CachingDemo sample solution with caching enabled. The volume of successful tests
still reaches a plateau, but at a higher user load. The request rate at this load is
significantly higher than that of the uncached solution. Similarly, although the
average test time still increases with load, the maximum response time is
approximately 1/20th that of the previous tests (0.05 ms against 1ms).

![Performance load test results for the cached scenario][Performance-Load-Test-Results-Cached]

----------

**Note:** Lack of caching sometimes acts as a natural regulator of throughput, and
once this restriction is relaxed the increasing volume of traffic that a website can
support by using caching might result in the system being overwhelmed. The system may return a large number of HTTP 503 (Service Unavailable)
messages, indicating that the website hosting the service is temporarily unavailable
due to the volume of work being processed. This is a common concern, increasing the
potential throughput of the application can require that the infrastructure the application runs on be scaled to handle the additional load.

----------

## Related resources
- [The Cache-Aside Pattern][cache-aside-pattern].

- [Data Consistency guidance][data-consistency-guidance].

- [Caching Guidance][caching-guidance].



[sample-app]: https://github.com/mspnp/performance-optimization/tree/master/NoCaching
[cache-aside-pattern]: https://msdn.microsoft.com/library/dn589799.aspx
[circuit-breaker]: ../../patterns/circuit-breaker.md
[client-cache]: ../../best-practices/api-implementation.md#considerations-for-optimizing-client-side-data-access
[data-consistency-guidance]: http://LINK-TO-CONSISTENCY-GUIDANCE-WHEN-PUBLISHED


[caching-guidance]: https://msdn.microsoft.com/library/dn589802.aspx
[AppInsights]: http://azure.microsoft.com/documentation/articles/app-insights-get-started/
[NLog]: http://nlog-project.org
[Log4Net]: http://logging.apache.org/log4net
[NewRelic]: http://newrelic.com/azure
[AppDynamics]: http://www.appdynamics.co.uk/cloud/windows-azure
[PerfView]: http://blogs.msdn.com/b/vancem/archive/2011/12/28/publication-of-the-perfview-performance-analysis-tool.aspx
[ANTS]: http://www.red-gate.com/products/dotnet-development/ants-performance-profiler/
[VisualStudioOnline]: http://www.visualstudio.com/get-started/load-test-your-app-vs.aspx
[NewRelic-server-requests]: _images/New-Relic.jpg
[Performance-Load-Test-Results-Uncached]:_images/InitialLoadTestResults.jpg
[Dynamic-Management-Views]: _images/SQLServerManagementStudio.jpg
[Performance-Load-Test-Results-Cached]: _images/CachedLoadTestResults.jpg

