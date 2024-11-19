### Background and motivation

When using the FixedWindowRateLimiter, the value set for the Retry-After header reflects the entire time window specified in the options rather than the remaining time until the next permissible request.

One would most likely expect to get back the time at which new permits are available.

### Reproduction steps
Utilize the official example provided here. The relevant code snippet is:

_rateLimiterOptions.OnRejected = (context, token) =>
{
    if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
    {
        context.HttpContext.Response.Headers.RetryAfter =
            ((int) retryAfter.TotalSeconds).ToString(NumberFormatInfo.InvariantInfo);
    }

    return ValueTask.CompletedTask;
};

Configure a Window option for, let's say, 00:01:00. After reaching the maximum request limit for this window, inspect the Retry-After header. A visual representation of the issue can be found

image

### Expected behavior
The Retry-After header should reflect the remaining time until the next request is permissible.

### Actual behavior
The Retry-After header only reflects the static window value defined in the options, ignoring the elapsed time.

### Considerations
If the queue is full and QueueLimit is greater than PermitLimit, then a RetryAfter value that is longer than the Window is returned. This could result in request1 getting told to wait out the current window and a future window, which potentially could result in request1 being undercut by say request2 that managed to request permit just as a new window started, thus putting it in the queue.\
Request1 might never get its permit since it could continue being undercut by newer requests being put in the queue.

Theoretically a request could stay in limbo forever:\
Window = 10 seconds and PermitLimit = 10.\
The window refreshes in 1 second and all permits are out. Request1 requests a permit but is told to wait 10 seconds before requesting again. After the window is refreshed and before Request1 tries again, all 10 permits are granted to newer requests. Request1 ask for a permit 1 second before the window refreshes and so on.


### Proposed change
A barbaric but fair solution:
1. Let all requests have a fair fight for the permits. All the requests get a RetryAfter value that represents when the window is refreshed. This way all requests know when they can ask for a permit and have pretty equal chances of either getting the permit or being put in queue.


### Risks
As pointed out in Issue-77991, the suggested solution is quite difficult to unit test, since the RetryAfter would be hard most likely won't be exactly the same every run.

The overall risk of being in limbo is still there, but is greatly minimised since a request would try again as soon as a new window is avaiable, thereby reducing the chance of being undercut by newer requests.

Some timer issues might cause a denied request to rerequest a permit just before a new window is created, thus setting the RetryAfter to something small like 0.001 second(pure speculation).

### Related issues
Issue 92557(this)\
Issue 77991
