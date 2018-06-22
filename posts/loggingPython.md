# The Pythonic Guide to Logging

When done properly, logs are a valuable component of your observability suite. Logs tell the story of how data has changed in your application. They let you answer questions such as: Why did John Doe receive an error during checkout yesterday? What inputs did he provide?

The practice of logging ranges from very simple static string statements to rich structured data events. In this article, we'll explore how logging in Python will give you a better view of what's going on in your application. In addition, I'll explore some best practices that will help you get the most value out of your logs.

### print('Why not just use the print statement?')

Many Python tutorials show readers how to use `print` as a debugging tool. Here's an example using `print` print an exception as a script:

```python
>>> import sys
>>>
>>> def captureException():
...     return sys.exc_info()
...
>>> try:
...     1/0
... except:
...     print(captureException())
...
(<class 'ZeroDivisionError'>, ZeroDivisionError('division by zero',), <traceback object at 0x101acb1c8>)
None

```

![](./images/loggingPython/exception1.png)

While this is useful in smaller scripts, as your application and operation requirements grow, `print` becomes a less viable solution. It doesn't offer you the flexibility to turn off entire categories of output statements and it only allows you to output to the `stdout`. It also misses information such as line number and time that it was generated that can assist with debugging. While `print` is the easiest approach since it doesn't require setup, it can quickly come back to bite you. It's also bad practice to ship a package that prints directly to `stdout` because it removes the user's ability to control the messages.

## logging.info("Hello World to Logging")

The logging module is part of Python's standard library and has been since version 2.3. It automatically adds context, such as line number & timestamps, to your logs. The module makes it easy to add namespace logging and severity levels to give you more control over where and what is output.

I'm a believer that the best way to learn something is to do it, so I'd encourage you to follow along in the Python REPL. Getting started with the logging module is simple, here's all you have to do:

```python
>>> import logging
>>> logging.basicConfig()
>>> logger = logging.getLogger(__name__)
>>>
>>> logger.critical('logging is easier than I was expecting')
CRITICAL:__main__:logging is easier than I was expecting
```

_What just happened?_ `getLogger` provided us with a logger instance. We then gave it the event 'logging is easier than I was expecting' with a level of `critical`.

### Levels

The Python module allows you to differentiate events based on their severity level. The levels are represented as integers between 0 and 50.  The module defines five constants throughout the spectrum, making it easy to differentiate between messages.

![](./images/loggingPython/loggingLevels.jpeg)

Each level has meaning attached to it, and you should think critically about the level you're logging at.

```python
>>> logger.critical("this better be bad")
CRITICAL:root:this better be bad
>>> logger.error("more serious problem")
ERROR:root:more serious problem
>>> logger.warning("an unexpected event")
WARNING:root:an unexpected event
>>> logger.info("show user flow through program")
>>> logger.debug("used to track variables when coding")
```

By default, the logger will only print `warning`, `error`, or `critical` messages. You can customize this behavior, and even modify it at runtime to activate more verbose logging dynamically.

```python
>>> # should show up; info is a higher level than debug
>>> logger.setLevel(logging.DEBUG)
>>> logger.info(1)
INFO:root:1
>>>
>>> # shouldn't show up; info is a lower level than warning
... logger.setLevel(logging.WARNING)
>>> logger.info(2)
```

### Formatting the Logs

The default formatter is not useful for formatting your logs, because it doesn't include critical information. The logging module makes it easy for you to add it in by changing the format.

We'll set the format to show the time, level, and message:

```python
>>> import logging
>>> logFormatter = '%(asctime)s - %(levelname)s - %(message)s'
>>> logging.basicConfig(format=logFormatter, level=logging.DEBUG)
>>> logger = logging.getLogger(__name__)
>>> logger.info("test")
2018-06-19 17:42:38,134 - INFO - test
```

Some of this data, such as `time` and `levelname` can be captured automatically, but you can (and should) push `extra` context to your logs.

### Adding Context

A generic log message provides just about as little information as no log message. Imagine if you had to go through your logs and you saw `removed from cart`. This doesn’t help you answer any questions. *When was the item removed? Who removed it? What did they remove?*

It's best to add structured data to your logs instead of string-ifying an object to enrich it.  Without structured data, it's difficult to decipher the log stream in the future. The best way to deal with this is to push important metadata to the `extra` object. Using this, you’ll be able to enrich your messages in the stream.

```python
>>> import logging
>>> logFormatter = '%(asctime)s - %(user)s - %(levelname)s - %(message)s'
>>> logging.basicConfig(format=logFormatter, level=logging.DEBUG)
>>> logger = logging.getLogger(__name__)
>>> logger.info('purchase completed', extra={'user': 'Sid Panjwani'})
2018-06-19 17:44:10,276 - Sid Panjwani - INFO - purchase completed
```

### Performance

Logging introduces overhead that needs to be balanced with the performance requirements of the software you write. While the overhead is generally negligible, bad practices and mistakes can lead to [unfortunate situations](https://www.youtube.com/watch?v=uyIlAO390v4).

Here are a few helpful tips:

#### Wrap Expensive Calls in a Level Check

The [Python Logging Documentation](https://docs.python.org/3/howto/logging.html#optimization) recommends that expensive calls be wrapped in a level check to delay evaluation.

```python
if logger.isEnabledFor(logging.INFO):
    logger.debug('%s', expensive_func())
```

Now, `expensive_func` will only be called if the logging level is greater than or equal to `INFO`.

#### Avoid Logging in the Hot Path

The hot path is code that is critical for performance, so it is executed often. It's best to avoid logging here because it can become an IO bottleneck, unless it's necessary because the data to log is not available outside the hot path.

## Storing & Accessing These Logs

Now that you've learned to write these (beautiful) logs, you've got to determine what to do with them. By default, logs are written to the standard output device (probably your terminal window), but Python's logging module provides a rich set of options to customize the output handling. Logging to standard output is encouraged, and platforms such as Heroku, Amazon Elastic Beanstalk, and Docker build on this standard by capturing the standard output and redirecting to other logging facilities at a platform level.

### Logging to a File

The logging module makes it easy to write your logs to a local file using a "handler" for long-term retention.

```python
>>> import logging
>>> logger = logging.getLogger(__name__)
>>>
>>> handler = logging.FileHandler('myLogs.log')
>>> handler.setLevel(logging.INFO)
>>>
>>> logger.addHandler(handler)
>>> logger.info('You can find this written in myLogs.log')
```

Now it's easy to search through your log file using `grep`.

```bash
grep critical myLogs.log
```

Now you can search for messages that contain keywords such as `critical` or `warning`.

### Rotating Logs

The Python logging module makes it easy to log in a different file after an interval of time or after the log file reaches a certain size. This becomes useful if you automatically want to get rid of older logs, or if you're going to search through your logs by date since you won’t have to search through a huge file to find a set of logs that are already grouped.

To create a `handler` that makes a new log file every day and automatically deletes logs more than five days old, you can use a `TimedRotatingFileHandler`. Here’s an example:

```python
logger = logging.getLogger('Rotating Log by Day')

# writes to pathToLog
# creates a new file every day because `when="d"` and `interval=1`
# automatically deletes logs more than 5 days old because `backupCount=5`
handler = TimedRotatingFileHandler(pathToLog, when="d", interval=1,  backupCount=5)
```

## The Drawbacks

It’s important to understand where logs fit in when it comes to observing your application. I recommend readers take a look at the [3 pillars of observability](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html) principle, in that logs should be one of a few tools you reach for when observing your application. Logs are a component of your observability stack, but metrics and tracing are equally so.

_Logging can become expensive_, especially at scale. Metrics can be used to aggregate data and answer questions about your customers without having to keep a trace of every action, while tracing enables you to see the path of your request throughout your platform.

## Wrapping Up

If you take anything away from this post, it should be that logs serve as the _source of truth_ for the user’s actions. Even for ephemeral actions, such as putting an item into and out of a cart, it’s essential to be able to trace the user’s steps for a bug report, and the logs allow you to determine their actions.

The Python logging module makes this easy. It allows you to format your logs, dynamically differentiate between messages using levels, and ship your logs externally using "handlers". Though not the only mechanism you should be using to gain insight into user actions, it is an effective way to capture raw event data and answer unknown questions.

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

![](./images/loggingPython/footer.png)
