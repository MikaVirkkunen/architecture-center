---
title: Synchronous I/O antipattern
description: 
author: dragon119
---

# Synchronous I/O antipattern

Blocking the calling thread while I/O completes can reduce performance and affect vertical scalability.

## Problem description

A synchronous I/O operation blocks the calling thread while the I/O completes. The calling thread enters a wait state and is unable to perform useful work during this interval, wasting processing resources.

Common examples of I/O include:

- Retrieving or persisting data to a database or any type of persistent storage.
- Sending a request to a web service.
- Posting a message or retrieving a message from a queue.
- Writing to or reading from a local file.

This antipattern typically occurs because:

- It appears to be the most intuitive way to perform an operation. 
- The application requires a response from a request.
- The application uses a library that only provides synchronous methods for I/O. 
- An external library performs synchronous I/O operations internally. A single synchronous I/O call can block an entire call chain.

The following code uploads a file to Azure blob storage. There are two places where the code blocks waiting for synchronous I/O: the `CreateIfNotExists` method and the `UploadFromStream` method.

```csharp
var blobClient = storageAccount.CreateCloudBlobClient();
var container = blobClient.GetContainerReference("uploadedfiles");

container.CreateIfNotExists();
var blockBlob = container.GetBlockBlobReference("myblob");

// Create or overwrite the "myblob" blob with contents from a local file.
using (var fileStream = File.OpenRead(HostingEnvironment.MapPath("~/FileToUpload.txt")))
{
    blockBlob.UploadFromStream(fileStream);
}
```

Here's an example of waiting for a response from an external service. The `GetUserProfile` method calls a remote service that returns a `UserProfile`.

```csharp
public interface IUserProfileService
{
    UserProfile GetUserProfile();
}

public class SyncController : ApiController
{
    private readonly IUserProfileService _userProfileService;

    public SyncController()
    {
        _userProfileService = new FakeUserProfileService();
    }

    // This is a synchronous method that calls the synchronous GetUserProfile method.
    public UserProfile GetUserProfile()
    {
        return _userProfileService.GetUserProfile();
    }
}
```

You can find the complete code for both of these examples [here][sample-app].

## How to fix the problem

Replace synchronous I/O operations with asynchronous operations. This frees the current thread to continue performing meaningful work rather than blocking, and helps improve the utilization of compute resources. Performing I/O asynchronously is particularly efficient for handling an unexpected surge in requests from client applications. 

Many libraries provide both synchronous and asynchronous versions of methods. Whenever possible, use the asynchronous versions. Here is the asynchronous version of the previous example that uploads a file to Azure blob storage.

```csharp
var blobClient = storageAccount.CreateCloudBlobClient();
var container = blobClient.GetContainerReference("uploadedfiles");

await container.CreateIfNotExistsAsync();

var blockBlob = container.GetBlockBlobReference("myblob");

// Create or overwrite the "myblob" blob with contents from a local file.
using (var fileStream = File.OpenRead(HostingEnvironment.MapPath("~/FileToUpload.txt")))
{
    await blockBlob.UploadFromStreamAsync(fileStream);
}
```

The `await` operator returns control to the calling environment while the asynchronous operation is performed. The code after this statement acts as a continuation that runs when the asynchronous operation has completed.

A well designed service should also provide asynchronous operations. Here is an asynchronous version of the web service that returns user profiles. The `GetUserProfileAsync` method depends on having an asynchronous version of the User Profile service.

```csharp
public interface IUserProfileService
{
    Task<UserProfile> GetUserProfileAsync();
}

public class AsyncController : ApiController
{
    private readonly IUserProfileService _userProfileService;

    public AsyncController()
    {
        _userProfileService = new FakeUserProfileService();
    }

    // This is an synchronous method that calls the Task based GetUserProfileAsync method.
    public Task<UserProfile> GetUserProfileAsync()
    {
        return _userProfileService.GetUserProfileAsync();
    }
}
```

For libraries that don't provide asynchronous versions of operations, it may be possible to create asynchronous wrappers around selected synchronous methods. Follow this approach with caution. While it may improve responsiveness on the thread that invokes the asynchronous wrapper, it actually consumes more resources. An extra thread may be created, and there is overhead associated with synchronizing the work done by this thread. Some tradeoffs are discussed in this blog post: [Should I expose asynchronous wrappers for synchronous methods?][async-wrappers]

Here is an example of an asynchronous wrapper around a synchronous method.

```csharp
// Asynchronous wrapper around synchronous library method
private async Task<int> LibraryIOOperationAsync()
{
    return await Task.Run(() => LibraryIOOperation());
}
```

Now the calling code can await on the wrapper:

```csharp
// Invoke the asynchronous wrapper using a task
await LibraryIOOperationAsync();
```


## How to detect the problem
From the viewpoint of a user running an application that performs synchronous I/O
operations, the system can seem unresponsive or appear to hang periodically. The
application may even fail with timeout exceptions. A web server hosting a service
that performs internal I/O operations synchronously may become overloaded. Operations
within the web server may fail as a result, causing HTTP 500 (Internal Server Error)
response messages. Additionally, in the case of a web server such as IIS, incoming
client requests might be blocked until a thread becomes available. This can result in
excessive request queue lengths, resulting in HTTP 503 (Service Unavailable) response
messages being passed back to the caller.

You can perform the following steps to help identify problems caused by synchronous
I/O:

1. Monitor the live web server and determine whether blocked worker threads are
constraining the throughput of the system.

2. If web requests are being blocked due to lack of threads, review the application
to determine which operations may be performing I/O synchronously.

3. Perform controlled load testing of each operation that is performing synchronous
I/O to ascertain the effects of these operations on overall system performance.

The following sections apply these steps to the sample application described earlier.

----------

**Note:** If you already have an insight into where problems might lie, you may be
able to skip some of these steps. However, you should avoid making unfounded or
biased assumptions. Performing a thorough analysis can sometimes lead to the
identification of unexpected causes of performance problems. The following sections
are formulated to help you to examine applications and services systematically.

----------

### Monitoring web server performance

For Azure web applications and web roles, it is also worth monitoring the performance
of the IIS web server. In particular, you should pay attention to the ASP.NET request
queue length to establish whether requests are being blocked waiting for available
threads to process them during periods of high activity. You can gather this
information by enabling Azure diagnostics. You can retrieve the performance counter
data from the `WADPerformanceCountersTable` table generated by Azure diagnostics. The
key performance counters to examine are `\ASP.NET\Requests Queued` and
`\ASP.NET\Requests Rejected`. The article [Analyzing logs collected with Azure Diagnostics][analyzing-logs] provides more information on collecting and analyzing
diagnostics.

You should also instrument the system to capture information about the way requests are handled once they have been accepted. Tracing the flow of a request can
help identify whether it is performing slow-running calls and blocking the current
thread. Thread profiling can also prove useful to highlight requests that are being
blocked.

### Reviewing the application

Perform a code review to identify the source of any I/O operations that the
application performs. I/O operations that appear to be synchronous should be
evaluated to determine whether they are likely to cause a blockage as the number of
requests increases.

----------

**Note:** I/O operations that are expected to be very short lived and that are
unlikely to cause contention might be best left as synchronous operations. For example, accessing fast, local resources such as
small private files on an SSD drive. <<RBC: Does this work? There was an excess of punctuation as written.
In these cases the overhead of creating and managing a separate task, dispatching it
to another thread, and synchronizing with that thread when the task completes might
outweigh the benefits of performing an operation asynchronously.

----------

### Load testing the application

Load testing can help  determine the effects of synchronous I/O operations on the
performance of the system. As an example, the following graph illustrates the
performance of the synchronous `GetUserProfile` method in the `SyncController`
controller in the sample application under varying loads up to 4000 concurrent users.

----------

**Note:** The sample application was deployed to a 2-instance cloud service
containing medium virtual machines. Also note that the synchronous operation is
configured to take 2 seconds, so the minimum response time will be slightly over 2
seconds. However, when the load reaches approximately 2500 concurrent users the
average response time reaches a plateau, although the volume of requests per second
continues to increase. Note that the scale for these two measure is logarithmic, and
the number of requests per second doubles between this point and the end of the test.

----------

![Performance chart for the sample application performing synchronous I/O operations][sync-performance]

In isolation, it is not necessarily obvious from this test that performing
synchronous I/O is a problem unless the response time is outside of agreed SLAs.
However, if processing resources are more constrained (either as a result of a much
larger volume of users, or using smaller scale hosts) then you may reach a tipping
point where the web server is unable to process the volume of outstanding requests in
a timely manner and client applications receive time-out exceptions. To understand
why this is so, consider that the application runs as a web role. Incoming requests
are queued by the IIS web server and handed to a thread running in the ASP.NET thread
pool. Because each operation performs I/O synchronously, the thread is blocked until
the operation completes. As the workload increases, the rate requests are
received at also increases up to the point at which all of the ASP.NET threads in the
thread pool are allocated and blocked. At this time, any further incoming requests
cannot be not fulfilled immediately and wait in the queue until a running operation
completes and a thread becomes available. As the queue length grows, requests start
to time out, and this is characterized by the number of errors that are returned to
clients.

For comparison purposes with the earlier examples, the following graph illustrates
the performance of the asynchronous `GetUserProfileAsync` method in the
`AsyncController` controller in the sample application under the same varying loads
and conditions as the first test (a 2-instance set of medium sized virtual machines).

![Performance chart for the sample application performing asynchronous I/O operations][async-performance]

This time the throughput is far higher. This is because the ASP.NET threads handling
the incoming requests are not blocked by I/O requests. Instead, they become available
to process further requests. In the same period of time as the previous test, the
system successfully handles a nearly tenfold increase in throughput as measured in
requests per second (remember that the scale is logarithmic). Additionally, the
average response time is relatively constant and remains approximately 25 times
smaller than that shown by the previous test.

----------

**Note:** The default maximum queue length supported by IIS is 5000 outstanding
requests, regardless of the size of the host virtual machine. Once this number of
requests is exceeded they may start failing. Scaling the size of virtual machines
vertically will not help. In this situation, you should scale horizontally and spread
the load over more instances. Alternatively, you may be able to modify the default
ASP.NET [MaxConcurrentRequestsPerCPU][MaxConcurrentRequestsPerCPU] and
[MaxConcurrentThreadsPerCPU][MaxConcurrentThreadsPerCPU] configuration settings, which
will increase the ASP.NET queue length.

----------


## Consequences of the solution

The system should be able to support more concurrent user requests than before, and
as a result be more scalable. Functionally, the system should remain unchanged.
Performing load tests and monitoring the request queue lengths should reveal that
requests are retrieved more quickly rather than timing out as they spend less time
blocked waiting for an available thread.

You should note that improving I/O performance may cause other parts of the system to
become bottlenecks. Alleviating the effects of synchronous I/O operations that were
previously constraining performance may cause overloading of other parts of the
system. For example, unblocking threads could result in an increased volume of
concurrent requests to shared resources resulting in increased competition and
possibly resource starvation or throttling. Therefore it may be necessary to scale or
redesign the I/O resources. Increase the number of web servers or partition data
stores to help reduce contention.

## Related resources

- [Should I expose asynchronous wrappers for synchronous methods?][async-wrappers]

- [Using Asynchronous Methods in ASP.NET 4.5][AsyncASPNETMethods]

- [Analyzing logs collected with Azure Diagnostics][analyzing-logs]

[sample-app]: https://github.com/mspnp/performance-optimization/tree/master/SynchronousIO
[fullDemonstrationOfSolution]: https://github.com/mspnp/performance-optimization/tree/master/SynchronousIO
[async-wrappers]: http://blogs.msdn.com/b/pfxteam/archive/2012/03/24/10287244.aspx
[AsyncASPNETMethods]: http://www.asp.net/web-forms/overview/performance-and-caching/using-asynchronous-methods-in-aspnet-45
[analyzing-logs]: https://msdn.microsoft.com/library/azure/dn904284.aspx
[MaxConcurrentRequestsPerCPU]: https://msdn.microsoft.com/library/system.web.hosting.hostingenvironment.maxconcurrentrequestspercpu.aspx
[MaxConcurrentThreadsPerCPU]: https://msdn.microsoft.com/library/system.web.hosting.hostingenvironment.maxconcurrentthreadspercpu.aspx
[sync-performance]: _images/SyncPerformance.jpg
[async-performance]: _images/AsyncPerformance.jpg
