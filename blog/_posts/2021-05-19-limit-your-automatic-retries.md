---
layout: post
title: Limit your automatic retries
excerpt_separator: <!--more-->
originally_posted: https://blog.arkency.com/limit-your-automatic-retries/
tags: ['arkency', 'distributed systems', 'integrations']
publish: true
---

Recently I've seen such code:

```ruby
def build_state(event)
  # ...
rescue RubyEventStore::WrongExpectedEventVersion
  retry
end
```

As you can see, the `build_state` may raise an `RubyEventStore::WrongExpectedVersion` which, if raised, is always automatically retried.

This error is raised if there are two concurrent writes to the same stream in `RubyEventStore` but only one of them is allowed to succeed.
One of them will successfully append its event to the end of the stream and the second one will fail.
That second request with automatic retry implemented (like in above snippet) will retry after failing, and most likely succeed this time.

**But what if it doesn't?**

What if there is so much concurrent writes, that this request will fail over and over?

What if there is a bug in the code which will always raise that error and we always retry? **We have created an infinite loop, which deployed to production can bring our system down in seconds.** :)

At least for these reasons, the code should always be retried limited number of times (one should be enough ;) ). An example could be:

```ruby
def build_state(event)
  with_retry do
    # ...
  end
end

private
def with_retry
  yield
rescue RubyEventStore::WrongExpectedVersion
  yield
end
```

## How many retries to choose?

The less, the better for your system resilience (thus **I recommend only one retry at most**). Instead of automatically retrying, it's better to let it fail the background job and retry it with [exponential backoff](https://github.com/pawelpacana/exponential-backoff) (sidekiq has that built-in, other background job libraries may have too).

The same applies to frontend, especially because user, tired of waiting, could already close the tab and we are still trying to prepare a response for him -- it's better to fail the request and let the frontend decide whether to retry it again or not. By letting it fail we have faster requests, and therefore one less reason to have a production outage related to request queuing.

Remember that if your worker continues working on this failed task, it is not working on other tasks. And some failure reasons (like network error on third parties) have a high chance of happening again.

## Third party integrations

All of the above applies also to integration with third parties. Even if HTTP request has failed because of a transient failure (like a timeout), it has a high chance of happening again if retried just after failing. Maybe your third party is temporarily overloaded, maybe they are just in the middle of the deployment, maybe they have a temporary bug on which their team is already working. Let it fail, and retry later.

## Error reporting and metrics

Why there were retries in the first place? Often they are added because the code failed once, the error came up to error reporting software and someone decided that if it has failed, then we need to retry it again. Noone likes to get a notification for an error about which you can't do anything, therefore it's best to just ignore these transient errors. [Instead of reporting them, it's better to add a metric to your monitoring system each time your request succeed or fail](https://blog.arkency.com/2015/11/monitoring-services-and-adapters-in-your-rails-app-with-honeybadger-newrelic-and-number-prepend/) and add an alert threshold when it fails too often. This is possible in all open source monitoring systems (like Grafana+InfluxDB) or proprietary ones (like NewRelic).
