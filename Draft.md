### Background and motivation

When using the FixedWindowRateLimiter, the value set for the Retry-After header reflects the entire time window specified in the options rather than the remaining time until the next permissible request.

One would most likely expect to get back the time at which new permits are available.

### Reproduction steps
The following slighty modified test passes, which means the metadata is the same, even after a significant delay.
[Fact]
public async Task CorrectRetryMetadataWithQueuedItem()
{
    var options = new FixedWindowRateLimiterOptions
    {
        PermitLimit = 2,
        QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
        QueueLimit = 1,
        Window = TimeSpan.FromSeconds(20),
        AutoReplenishment = false
    };
    var limiter = new FixedWindowRateLimiter(options);

    using var lease = limiter.AttemptAcquire(2);
    // Queue item which changes the retry after time for failed items
    var wait = limiter.AcquireAsync(1);
    Assert.False(wait.IsCompleted);

    await Task.Delay(19*1000);
    using var failedLease = await limiter.AcquireAsync(2);
    Assert.False(failedLease.IsAcquired);
    Assert.True(failedLease.TryGetMetadata(MetadataName.RetryAfter, out var typedMetadata));
    Assert.Equal(options.Window.Ticks, typedMetadata.Ticks);
}

Say we were to set a timer to retry getting the permit at the specified RetryAfter:
The test passes since the new request simply undercuts the old request.
[Fact]
public async Task NewRequestUndercutsOldRequest()
{
    var options = new FixedWindowRateLimiterOptions
    {
        PermitLimit = 1,
        QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
        QueueLimit = 0,
        Window = TimeSpan.FromSeconds(5),
        AutoReplenishment = true
    };
    var limiter = new FixedWindowRateLimiter(options);

    using var lease = limiter.AttemptAcquire(1);
    Assert.True(lease.IsAcquired);

    Thread.Sleep(4*1000); // 1 second left of the window
    var failedLease = await limiter.AcquireAsync(1);
    Assert.False(failedLease.IsAcquired);
    failedLease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfterValue);

    var tasks = new List<Task>();

    // Task to wait for the RetryAfter duration and then send a request.
    tasks.Add(Task.Run(async () =>
    {
        await Task.Delay(retryAfterValue); // Wait for RetryAfter (5 seconds).

        using var oldLease = await limiter.AcquireAsync(1);
        Assert.False(oldLease.IsAcquired);
    }));

    // Task to send a new request 1 second into the new window.
    tasks.Add(Task.Run(async () =>
    {
        await Task.Delay(2 * 1000); // 1 second into the new window
        using var newLease = await limiter.AcquireAsync(1);
        Assert.True(newLease.IsAcquired);
    }));

    // Wait for all tasks to complete.
    await Task.WhenAll(tasks);
}


### Expected behavior
The Retry-After header should reflect the remaining time until the next request is permissible.
In test 1 it should be ~1 second.

In test 2 the old request should have gotten the permit.

### Actual behavior
The Retry-After header only reflects the static window value defined in the options, ignoring the elapsed time.
In test 2 this results in a new request getting the permit.

### Considerations
If the queue is full and QueueLimit is greater than PermitLimit, then a RetryAfter value that is longer than the Window is returned. This could result in request1 getting told to wait out the current window and a future window, which potentially could result in request1 being undercut by say request2 that managed to request permit just as a new window started, thus putting request2 in the queue.\
Request1 might never get its permit since it could continue being undercut by newer requests being put in the queue.

Theoretically a request could stay in limbo forever(if a timer is set to the RetryAfter value):\
Window = 10 seconds and PermitLimit = 10.\
The window refreshes in 1 second and no permits are available. Request1 requests a permit but is told to wait 10 seconds before requesting again, instead of 1 second. After the window is refreshed and before Request1 tries again, all 10 permits are granted to newer requests. Request1 ask for a permit 1 second before the window refreshes and so on.


### Proposed change
A barbaric but fair solution:
1. Let all requests have a fair fight for the permits. All the requests get a RetryAfter value that represents when the window is refreshed. This way all requests know when they can ask for a permit and have pretty equal chances of either getting the permit or being put in queue.
I realise that, in terms of queue times, in wont have any significant effect if there are no new requests to take the leases. But wouldnt it make more sense to be put in queue, rather than hope to maybe get a permit?

Remove the permit parameter from the function and only return the remaining time.

private FixedWindowLease CreateFailedWindowLease()
{
    // Return the remaining time 
    TimeSpan? remainingTime = _options.Window - RateLimiterHelper.GetElapsedTime(_lastReplenishmentTick);
    return new FixedWindowLease(false, remainingTime);
}


### Risks
As pointed out in Issue-77991, the suggested solution is quite difficult to unit test, since the RetryAfter would be hard most likely won't be exactly the same every run.
An example of a test failing:
Assert.Equal() Failure: Values differ
Expected: 200000000
Actual:   199999954

The overall risk of being in limbo is still there, but is greatly minimised since a request would try again as soon as a new window is available, thereby reducing the chance of being undercut by newer requests.

Some timer issues might cause a denied request to rerequest a permit just before a new window is created, thus setting the RetryAfter to something small like 0.001 second(pure speculation).

### Related issues
Issue 92557(this)\
Issue 77991
