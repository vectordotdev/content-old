# The Pythonic Guide to Logging

Logging is a development practice we all know we should be doing but is something that is rarely done correctly. I'm known to do this as well - for me, it's perpetually pushed off. Not only will I show you how to log in Python, but also some best practices, so you're not slamming your head against the keyboard in a couple of months.

## logging.info("Hello World to Logging")

The logging module is part of Python's standard library, and it automatically adds context, such as line number & timestamps, to your logs. The module makes it easy to add severity levels so you can see what is most relevant to your task. When I first approached the [logging module documentation](https://docs.python.org/3/library/logging.html), I found it hard to understand.

As Martin Golding said:
> Always code as if the guy who ends up maintaining your code will be a violent psychopath who knows where you live.

Even though I think self-preservation is a good enough reason to start logging (you don’t want the person maintaining your code to come after you), logging can allow you to get your job done quickly and more effectively. Here are a few reasons why:

_1. Logs are often your most valuable tool._

The value you extract from your logs is entirely up to you. If done properly logs can, by far, be your most valuable observability tool. Any experienced developer can tell you many stories of times logs provided the detail they needed to debug a problem.

_2. Logs let you answer unknown questions_

Because logs capture your raw event data, you can derive answers to just about any question you think of in the future, especially questions that rely on high cardinality attributes.

_3. Logs tell stories._

Logs let you answer questions such as: Why did John Doe receive an error during checkout yesterday? What inputs did he provide?

## Writing Effective Logs

I'm a believer that the best way to learn something is to do it, so open up your editor (yes, I’m serious) and get ready to learn.

Fortunately, logging has been a part of the Python standard library since version 2.3.

```python
import logging
logger = logging.getLogger(__name__)

logger.info('logging is easier than I was expecting')
```

Never seen `__name__` before? You might have seen this with `if __name__ == "__main__":`.  You don't have to use it, but it creates a logging instance that is isolated to the current Python module. Calling `logging.getLogger(__name__)` repeatedly will result in the same logger instance being returned. This instance can be configured separately from other logger instances, such as setting it to a different level or adding different handlers.

### print('Why not just use the print statement?')

It seems like it would just be easier to `print` rather than learning best practices associated with logging. Using `print` is the obvious lazy-approach since it doesn't require any setup, but it becomes painful to manage as your program grows because it's difficult to remove the statements before shipping to production.

Using the print statement does have its place in development. Let's assume that you want to track exceptions:

```python
import sys

def captureException():
  print(sys.exc_info())
```

![](./images/loggingPython/exception1.png)

`print` lets us see the return type and helps us iterate on our code quickly.

I've found `print` helpful when I'm developing, but it's bad practice to ship a package that prints directly to `stdout` because the user won't know where the print messages are coming from.

### Levels

Like most other good logging modules, the Python module is a leveled logger, meaning it allows you to differentiate between logs of different importance.

Imagine if a log message saying that a server was melting down was buried between thousands of messages of users signing in. You want to store all this information but should react differently to these messages.

There are five levels defined in the module, making it easy to differentiate between messages. Each of these levels represents a number (i.e. `critical` is 50 and `info` is 20).

![](./images/loggingPython/loggingLevels.jpeg)

```python
logger.critical("50 - this better be bad")
logger.error("40 - more serious problem")
logger.warning("30 - an unexpected event")
logger.info("20 - show user flow through program")
logger.debug("10 - used to track variables when coding, but ignored in prod")
```

The level can be modified at runtime so that the user can activate more verbose logging dynamically.

Let’s see what that means:
```python
# should show up; info is a higher level than debug
logger.setLevel(logging.DEBUG)
logger.info("1")

# shouldn't show up; info is a lower level than warning
logger.setLevel(logging.WARNING)
logger.info("2")
```

### Performance

Logging introduces overhead that needs to be balanced with the performance requirements of the software you write. While the overhead is generally negligible, bad practices and mistakes can lead to [unfortunate situations](https://www.youtube.com/watch?v=uyIlAO390v4).

Here are a few helpful tips:

1. Wrap Expensive Calls in a Level Check

The [Python Logging Documentation](https://docs.python.org/3/howto/logging.html#optimization) recommends that expensive calls be wrapped in a level check to delay evaluation.

```python
if logger.isEnabledFor(logging.INFO):
    logger.debug('%s', expensive_func())
```

Now, `expensive_func` will only be called if the logging level is greater than or equal to `INFO`.

2. Avoid Logging in the Hot Path

It's difficult to say to _only log actionable information_, because this requires a future knowledge of the logged events that will be actionable.

The hot path is code that is critical for performance, so it is executed a lot (millions/billions of times if you're working at scale). It's best to avoid logging here, unless necessary because it's not available outside the hot path. Even then, it's essential to avoid logging in a tight-loop, because that can cause problems with IO throughput if a significant number of logs are generated in a short period.

### Formatting the Logs

Your logs must provide a way to reconstruct the user's actions. It is best to log a description of the action that took place rather than dumping the object because it makes it difficult to decipher what is going on unless you're given a clear set of steps. Like a detective in a crime scene, you want to be able to trace the steps to gain a birds-eye-view of the actions taken up to a point in time.

A generic log message provides just about as little information as no log message. Imagine if you had to go through your logs and you saw `removed from cart`. This doesn’t help you answer any questions. *When was the item removed? Who removed it? What did they remove?*

This can be done by formatting your logs:
```python
logFormatter = '%(asctime)s - %(user)s - %(levelname)s - %(message)s'
logging.basicConfig(format=logFormatter)
```

Some of this data, such as `time` and `levelname` can be captured automatically, but you can (and should) push `extra` context to your logs.

### Adding Context

The _biggest_ issue I've seen is the pollution of log messages. It's best to add structured data to your logs instead of string-ifying an object to enrich it, which makes it impossible to decipher the log stream in the future. The best way to deal with this is to push all your metadata to the `extra` object. Using this, you’ll be able to decipher your messages from the stream, while still grab the full context to dig deeper into your user.

```python
logger = logging.getLogger(__name__)
logger.info('purchase completed', extra={‘user': 'Sid Panjwani'})
```

## Storing & Accessing These Logs

Now that you've learned to write these (beautiful) logs, you've got to determine what to do with them. By default, logs are written to the standard output device (probably your terminal window), but Python's logging module provides a rich set of options to customize the output handling. Logging to standard output is encouraged, and platforms such as Heroku, Amazon Elastic Beanstalk, and Docker build on this standard by capturing the standard output and redirecting to other logging facilities at a platform level.

### Logging to a File

The logging module makes it easy to write your logs to a local file using a "handler" for long-term retention.

```python
import logging
logger = logging.getLogger(__name__)

handler = logging.FileHandler('myLogs.log')
handler.setLevel(logging.INFO)

logger.addHandler(handler)
logger.info('You can find this written in myLogs.log')
```

### Searching

It's easy to search through your log file using `grep`.

```bash
grep critical log_file_name
```

Now you can search for messages that contain keywords such as `critical` or `warning`.

### Rotating Logs

Though not generally taught in logging guides, the Python logging module makes it easy to log in a different file after an interval of time or after the log file reaches a certain size.

This becomes useful if you automatically want to get rid of older logs, or if you're going to search through your logs by date since you won’t have to search through a huge file to find a set of logs that are already grouped.

To create a `handler` that makes a new log file every day and automatically deletes logs more than five days old, you can use a `TimedRotatingFileHandler`. Here’s an example:

```python
logger = logging.getLogger('Rotating Log by Day')

# writes to pathToLog
# creates a new file every day because `when="d"` and `interval=1`
# automatically deletes logs more than 5 days old because `backupCount=5`
handler = TimedRotatingFileHandler(pathToLog, when="d", interval=1,  backupCount=5)
```

## The Drawbacks

It’s important to understand where logs fit in when it comes to observing your application. I agree very much with the [3 pillars of observability](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html) principle, in that logs should be one of a few tools you reach for when observing your application. Logging is great, but it can't be the only lens into your stack. Metrics allow you to aggregate data and answer questions about your customers without having to keep a trace of every action, while tracing enables you to see the events leading to an error.

Logging isn't without drawbacks:

_Logs are expensive_, especially at scale. With great detail comes high costs. There are specific tactics you can employ to reduce the cost, but the nature of logging makes it more expensive.

_Logs are overwhelming_. Logs are a firehose of information and impossible for a human to understand at scale.

## Wrapping Up

If you take anything away from this post, it should be that logs serve as the _source of truth_ for the user’s actions. Even for ephemeral actions, such as putting an item into and out of a cart, it’s essential to be able to trace the user’s steps during a bug report, and the logs allow you to determine their actions between all your MicroServices.

The logging module makes this easy. It allows you to format your logs, dynamically differentiate between messages using levels, and ship your logs externally using "handlers". Though not the only mechanism you should be using to gain insight into user actions (also look into Metrics and Tracing), it is an effective way to capture raw event data and answer unknown questions.

***Instead of doing this yourself, we’ve got a pretty awesome service [here @ Timber](https://timber.io/) (it’s seriously great) that automatically captures context with your logs to make debugging easier. Try us out for (completely) free; you don’t even need a credit card!***

### Logging to the Cloud

Writing your logs to a hosted log aggregation service seriously makes your life easier, so you can focus your time on what’s important.

_Disclaimer: I’m a current employee @ Timber. This section is entirely optional, but I sincerely believe that logging to the cloud will make your life easier (and you can try it for completely free)._

```bash
pip install timber
```

```python
import logging
import timber

logger = logging.getLogger(__name__)

timber_handler = timber.TimberHandler(api_key='...')
logger.addHandler(timber_handler)
```

**That’s it.** All you need to do is get your API key from [timber.io](https://timber.io/), and you’ll be able to see your logs. We automatically capture them from the logging module, so you can continue to follow best-practices and log normally, while we seamlessly add context.

Now that you’ve created _gorgeous_ logs that gives your coworker a view of the user’s actions, you need to know best practices for searching and reacting to these logs.

![](./images/loggingPython/footer.png)
