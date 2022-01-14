---
layout: post
title: Always set explicit timeouts
excerpt_separator: <!--more-->
tags: ['silverfin']
originally_posted: https://engineering.silverfin.com/always-set-timeouts/
---

Our next ticket is to implement another integration with 3rd party HTTP API. As it's in life, the documentation is imprecise, provided examples are outdated, and nobody is responding to our e-mails. It's not the first one though, so after some time we do have it ready. Either it is MVP, or maybe even we are confident enough to say that this integration between our app and 3rd party "works correctly" (whatever that means, in presence of the API of which the code one cannot see).

<!--more-->

Despite all of the problems, there is a strong chance that one thing wasn't a problem during development -- the availability of that HTTP service. When working on implementation the requests are sent rarely (compared to what will happen in the production system) and even then, we don't pay attention to the response times. Pull request gets merged and then for some time we look at the new metrics for some time. We check whether background jobs are processing correctly, without errors, and within reasonable times for that functionality. All good, we can proceed to the next ticket.

Fast forward a few months.

What happens when (not if, because it **will** happen sooner or later) HTTP service owned by the 3rd party will timeout? In many HTTP libraries, the default timeout is 120 or 90 seconds.

Will one background job processing for 90 seconds will be a problem? Surely not, we have dozens, if not hundreds of background workers. That won't be even considered a hiccup.

Will hundreds of background jobs processing for 90 seconds will be a problem? Uh-oh, unless we have actively thought about that scenario, **we now have hundreds of jobs stuck, waiting for an answer from an HTTP service, and nothing else is processing**. Depending on how much we rely on the background jobs it can be either only some sweat and lesson learned or a time where our service was "basically down".

That's why it's so important to ensure that all HTTP calls have reasonable timeouts set. If we think that a 3-second response is unacceptable in our system, why would we not react to the fact when third parties respond for over a minute? Of course, it's not always possible to set a timeout limit to 3 seconds. If one of the HTTP calls needs to pull lots of data, it's reasonable to set a higher timeout than on the HTTP call which fetches a simple resource. And even in that case, you don't need to accept waiting 60 seconds for opening a connection, so setting higher `read_timeout`, but low `open_timeout` is still advisable. Moreover, setting explicit timeout in the code informs other developers that we were aware of the potential outage to happen and we tried to set reasonable values for these timeouts.

Also, it makes sense to limit the number of resources being pulled. If the API does allow to customize page size, it's better to send 2 requests asking for 500 entries, than 1 request asking for 1000. Not only one can set a lower timeout in the first one, but it's also probably less likely to fail.

Of course, **timeouts are not the only thing one can do to prevent misbehaving 3rd parties** from pushing us down:
* Limiting the retries of the background jobs depending on the error being raised will ensure that a job that failed with the `TimeoutError` won't be retried again to occupy the worker for the next 90 seconds.
* Implementing a [circuit breaker](https://martinfowler.com/bliki/CircuitBreaker.html) prevents subsequent calls from being made, even without wasting time waiting for the job's failure.
* Using a separate instance for each external integration limits the damage done and at the same time is easy to implement in systems with clusterized infrastructure.

**Preventing is one, awareness is another**. Sending the metric event about timeout happened to monitoring infrastructure will ensure that the support team is aware of the issue. But now, if you have defending mechanisms in place, the only thing the support team needs to do is to give a heads up "hey, FYI integration X is not working". There is no "we have an outage, nothing else is processing except that malfunctioning integration" nor "does someone remember what's an API to remove all jobs of given type from the retries" (btw. [here you go](https://www.mikeperham.com/2021/04/20/a-tour-of-the-sidekiq-api/#scanning-and-filtering)).

With all these in place, you may make your colleague's support day a lazy one by doing little precautions beforehand.
