# Outline - The Rise of Structured Logs

_Since it doesn't look like there's a ton of organic search potential, I think it would be best to make it highly technical (they tend to do well on HN)._

_This was heavily inspired by Zach's article on [why we're building Timber](https://timber.io/blog/why-were-building-timber/). I think both articles are trying to push readers to the same conclusion._

## Why Log

1. Logs serve as the source of truth.
2. Logs show you what actually happened, not what you 'expect' to happen.
3. Logs deal with discrete events, not an aggregation of those events.

Taken from [why we building Timber](https://timber.io/blog/why-were-building-timber/) by Zach:
"The log is the single, reliable, immutable source of truth. Or as Jay Kreps so thoughtfully puts it:
The log is the record of what happened.... Since the log is immediately persisted it is used as the authoritative source in restoring all other persistent structures..."

Maybe: A callout to the [3 pillars of observability](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html) with an explanation that logging shouldn't be your only view into the system, but it is an important aspect of observing your system nevertheless. 

## The Current Solution Fails Us

_Unstructured Logging_

Taken from 'Why we built Timber' by Zach:
The way we've built applications in the last decade has changed, while our approach to debugging and monitoring has stayed constant. Applications that were once structured as monolithic architectures are now being built as lean microservices. Each microservice can be built in the language best fitted for the function, and individually scaled to handle demand.

Monitoring microservices is far from intuitive. In a perfect world, the log would serve as the source of truth, but when split into microservices, where each language formats logs differently, it's difficult to understand what's happening. Everything sounds great until you're faced with a terabyte of text and no easy way to gain insight.

## Our Variant View

We've bet our company on the idea that developers want a better way to debug their applications. This isn't to say that the text wasn't structured before, but the format differed for every runtime, logger, and platform. 

Inspired by the [JSON Schema](https://github.com/timberio/log-event-json-schema):
That's why we've defined a _simple_ [structure for log events](https://github.com/timberio/log-event-json-schema). It solves the unpredictable, brittle nature of logs by creating a contract around its structure. It's used internally at [Timber](https://timber.io/) and across thousands of other companies with success. It helps us make assumptions about your data and work with predictable structures.

## Callout to Timber
Inspired by [how is Timber different](https://docs.timber.io/concepts/how-is-timber-different/) and the [JSON Schema](https://github.com/timberio/log-event-json-schema):
We believe the utility of your logs is directly correlated with the quality of your data. You can log your own events that adhere to [this schema](https://github.com/timberio/log-event-json-schema), or you can cheat using Timber's [application level libraries](https://docs.timber.io/languages) to automatically generate high-quality log data for you. Spend less time debugging and more time shipping.



