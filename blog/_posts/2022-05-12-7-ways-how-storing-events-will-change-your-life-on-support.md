---
layout: post
title: 7 ways to improve your next support rotation with event storage
excerpt_separator: <!--more-->
tags: ['silverfin']
publish: true
originally_posted: https://engineering.silverfin.com/support-with-event-logs/
---

In every software company, part of engineering responsibilities is to help the Support Team resolve customer issues and maintain the stability of the application by responding to errors and alerts.

In this article, I want to briefly describe the benefits for engineers working on support issues of having [an application that has an event store and/or is event-driven](https://martinfowler.com/articles/201701-event-driven.html).

<!--more-->

## 1. Reading a story instead of debugging

Each support task is a little story to uncover. What happened? In what order? Who did what?

If each domain change is covered by domain events, most things become clear and the "mystery" becomes very small, and usually very technical (race conditions, concurrent writes because of missing pessimistic locks, third-party integration issues, etc.). In many cases it becomes as simple as just explaining what happened, there is no "investigation" part – you just explain why your system behaved as it did (which can obviously lead to a bug ticket or a feature request).

You just read a story written in a more technical language and basically translate it into customer language.

You can also match the story with what the customer tells you. Are there any gaps? Anything that customers didn't say because they thought it wasn't useful, or they forgot, or they clicked something by accident -- **we know it and we don't need to make guesses**.

## 2. Fewer follow-up questions
Neither developer nor customer nor support likes the flow of "please ask the customer when they did X", which involves multiple people and is slow and frustrating to all parties if there are too many follow-ups. The more information you have upfront, the faster it becomes to discover the story that led to the problem.

## 3. Engineers’ actions don't go unnoticed
If you ever had a thought "some other engineer changed here something via console... _probably_" when working on a support issue, this point is for you.

With some easy tweaks, you can store all the domain events published in the current Rails console session with specific metadata attached to each event. A good rule of thumb is to include a Trello/Jira issue link and an identifier for the developer operating the console. After that, when you are debugging, you just look at the event history and you see that "your colleague X has done Y when working on the issue Z". That way, nothing that other engineers do is left without a trace, which makes the debugging story clear.

Of course, it requires some basic discipline – we don't make changes without publishing a domain event. It may seem that it's too much work, but if the application is well-covered with domain events, most times you don't have to do anything. Sometimes, it may happen that none of the currently existing domain events fit the change you are making. But in a good event store system, publishing custom event is as easy as:

```ruby
SomeVeryCustomizedThingHappened = Class.new(RailsEventStore::Event)
  event_store.publish(SomVeryCustomizedThingHappened.new(data: {
company_id: company.id,
# ...
}))
```

Once, a colleague of mine wrote a script to fix in bulk some products in an e-commerce application. However, in his script he accidentally set the VAT rate to 0% instead of its actual value. When someone noticed that there were products with 0% VAT floating around, this was immediately escalated into a major urgent issue. The script’s author had already gone home that day, but because we had the event information stored in the database, we were able to jump in and see the whole story in a few minutes: our colleague had worked on the support issue X, and he had fixed products in bulk, but because of some default argument for the VAT, he had accidentally set all of them to 0% VAT rate.

When the story is clear, action can be taken easily and with confidence that we are fixing the right thing.

## 4. Being able to link to specific actions
What often happens when resolving support issues is a lot of communication about things like "the customer says they did X" or "the customer says they didn't do X". Without linkable events, the role of Support in these cases is basically being able to describe "customer did X around five o'clock". They won't give a link to the log entry that recorded the request, obviously.

But the event store is different – it is less technical and written in the domain language of your application. When there is an event store, backed by an event store browser, Support can give Engineering a link to a specific domain event that took place. The domain event is powerful: it describes not only what happened, but also for what tenant, when, who triggered the event, and why (how so? check the next point!).

This makes the job of Engineering is clear:

- explain why that event happened when it shouldn’t have ("Can you explain why the sync was triggered twice during the night?" – "It's because of a race condition, nothing to worry about.")
- explain why it happened at the wrong time ("Can you explain why that thing happened after 7AM instead of midnight?" - "That's because this integration had a maintenance mode at night.")
- explain why it happened in connection with other events ("We have recorded an event that the customer changed setting X to Y, so feature Z should behave differently. However, from the domain events we see that it kept its default behaviour.")
- And so on, the pattern is clear :)
Being able to link to specific actions instead of just passing around information "customer says they did X" is way faster for everyone.

## 5. Causations and correlations
What about the "why" things happened? We can't read minds of our customers. What we can do though, is read the mind of our distributed system. [Domain events may have causation and correlation information](https://railseventstore.org/docs/v2/correlation_causation/).

Causation information means that for the domain event "CustomerPaidForAnOrder" we have a direct link to the event "PaymentSuccessful".

Correlation means that all events taking place because of the same web request are grouped into a single event stream. The customer clicking one button may trigger multiple Sidekiq jobs (which may trigger other Sidekiq jobs), and usually there is no way of simply viewing "this is all that happened (throughout the application’s entire domain, not just in Sidekiq logs) after that web request".

## 6. (Best) Audit log for free

The audit log.

Many projects need one sooner or later. If your application is event-driven and you do store your events, you get this one basically for free.

The event store is audit log and more as it also has technical information that you don't necessarily want to present in the audit log (if only for the sake of reducing noise).

If your application is covered in events, building an audit log is a simple project of a few steps:

1. Decide on which events you want to show to the user.
2. Link the chosen events to a specific stream, like "FirmAuditLog$1234".
3. Build a UI to present it nicely.

Et voila, what could have been a months-long project is done in a week (again, only if your application is already event-driven with event storage).

## 7. Empowering lines of support

With everything said above, customer support becomes empowered by a great tool. With more specific information at hand, each line of support has more knowledge to handle their daily tasks better.

With domain events it's easier to spot patterns, or communicate between engineers and support personnel: for both parties, it's easier to explain that "to resolve such problems in the future, you need to take a look at domain events and check whether X, Y, and Z happened". This handily beats elaborate descriptions written in prose, with a bunch of links to different parts of the admin panel attached.

If you are considering moving your application toward event-driven architecture, have no doubt: doing so will have a tremendously good effect on the support part of your work.

