# Logging in Python: Becoming a More Effective Developer

Logging is a development practice we all know we should be doing but is something that is rarely done correctly. I'm known to do this as well -- for me, it's perpetually pushed off until tomorrow. From now, I'm going to hold myself to a higher standard (and I think you should too!).

If you already know the basics, feel free to skip to the bottom. Even if you know your way around the logging module in Python, I’m confident you’ll be able to take something away from this post. 

### Why bother?

It seems like it would just be easier to `print` rather than learning the best practices associated with logging. 

```python
dogs = ["Retriever", "Labrador", "Bulldog"]
for dog in dogs:
	print(dog)
```

When you're working alone on a small project, you can do this. I wouldn't recommend it because the logging package is easy enough to set up that it still makes sense to follow best practices, but you won't struggle to understand your print statements.

**So why bother with the logging module?**

The logging module automatically adds context, such as line number & timestamps, to your logs. The module makes it easy to add severity levels so you can see what is most relevant to your task. When I first approached the [logging module documentation](https://docs.python.org/3/library/logging.html), I found it hard to understand. Not only will I show you how to log in Python, but also some best practices you can follow to make, so you're not slamming your head against your keyboard in a couple of months.

As Martin Golding said:
> Always code as if the guy who ends up maintaining your code will be a violent psychopath who knows where you live.

Even though I think self-preservation is a good enough reason to start logging (you don’t want the person maintaining your code to come after you), logging can allow you to get your job done quickly and more effectively. Here are a few reasons why:

1. Visibility into User Behavior

When using a cloud-based solution such as AWS, it can often feel that you're putting something into a black box when something breaks. It's easy to see what's going in, and you know what you expect should come out, but it's near-impossible to tell what's happening on the inside. Logging can serve as your end-to-end solution to give you visibility into your cloud-based components.

2.  Prevent Problems

If you log effectively, it's possible for you to see issues as they occur, before the user even realizes it. With the power of instant alerts, you can detect and fix a problem without the user reaching out to you.

3. Troubleshooting

At times, things will go wrong with your software. The best you can do is to minimize the number of times this happens. When a user does reach out with an issue, the application logs serve as a source of truth of what the user has done and what has broken.

#### Logging isn’t Perfect

I find that most guides tend to skip the negatives of using the tool they’ve been trying to teach you. It’s important to understand that there are both pros and cons associated with using all these tools.

_Logs are expensive_. There’s no getting around it, especially when you’re at scale, logs not only allow you to inspect the user’s state at any point in time but also trace how they got to that point. 

You can use metrics to get around this. Metrics allow you to aggregate data and answer questions about your customers without having to keep a trace of every action a user has taken. The biggest issue with metrics is that you have to decide the question you want to answer before collecting the data, but they allow to create insights for cheap. 

## Writing Effective Logs

***It's seriously this easy...***

We’re believers that the best way to learn something is to do it, so open up your editor (yes, I’m serious) and get ready to learn.

Fortunately, logging has been a part of the Python standard library since version 2.3.

```python
import logging
logger = logging.getLogger(__name__)
```

Never seen `__name__` before? You might have seen this with `if __name__ == "__main__":`.  You don't have to use it with the logger, but it allows you to see what file the log is coming from when you read the logs. Basically, `__name__` allows you to see what file you are currently in if the file was imported _OR_ it will return `__main__` if you started your script from that file. [Here](https://www.youtube.com/watch?v=sugvnHA7ElY)) is a video that explains it well (and goes by quickly @ 2x speed).

### Levels

Like any good logging module, the one included with the Python library allows you to differentiate between logs of different importance.

Imagine if a log message saying that a server was melting down was buried between thousands of messages of users signing in. You want to store all this information but should react differently to these messages.

There are five levels defined in the module, making it easy to differentiate between messages. Though you don’t have to follow by these guidelines, it makes it easy only to pay attention to the relevant messages. 

![](./images/loggingPython/loggingLevels.jpeg)

```python
logger.critical("this better be bad")
logger.error("more serious problem")
logger.warning("an unexpected event")
logger.info("show user flow through program")
logger.debug("used to track variables when coding, but ignored in prod")
```

#### Performance

Even with Python (a language that is notoriously slow), it’s important to think about your performance. When creating programs that are built-to-scale, you don’t unnecessarily want to waste CPU cycles because multiplied by a million users, that [that could get expensive](https://www.youtube.com/watch?v=uyIlAO390v4).

How can we fix this while still using the logging module to its full capabilities Fortunately, the module has built-in the ability to ignore log messages of lower levels, so memory isn’t allocated, and CPU cycles aren’t wasted for log messages that don’t provide valuable information. 

Let’s see what that means:
```python
# should show up; info is a higher level than debug
logger.setLevel(logging.DEBUG)
logger.info("1")

# shouldn't show up; info is a lower level than warning
logger.setLevel(logging.WARNING)
logger.info("2")
```

We all know that feeling — getting angry at our program because it’s not acting as we expect and we spam print messages into our program trying to understand what’s going on. The moment we fix the issue, we’re forced to go back and delete those messages. Instead, it’s easy to set the level of the logger to ignore them. 

*Sighs Relief.* Now you can write as much as you need to the log while debugging, then change the level of your logger before pushing to production.

### Logging to a File

Generally, you don't (just) want to log to the console for them to be deleted immediately. The logging module makes it easy to write your logs to a local file using a `handler`. 

```python
import logging
logger = logging.getLogger(__name__)

handler = logging.FileHandler('myLogs.log')
handler.setLevel(logging.INFO)

logger.addHandler(handler)
logger.info('You can find this written in myLogs.log')
```

### Logging to the Cloud

Writing your logs to the cloud seriously makes your life so much easier, it serves as a layer of abstraction so you don’t have to worry how the logs are getting to the service and you can focus your time on what’s important.

_Disclaimer: I’m a current employee @ Timber. This section is entirely optional, but I seriously believe that logging to the cloud will make your life easier (and you can try it for completely free)._

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

**That’s it.** All you need to do is get your API key from [timber.io](https://timber.io/) and you’ll be able to see your logs. We automatically capture them from the logging module, so you can continue to follow best-practices and log normally, while we seamlessly add context. 

### Creating the _Picasso_ of Logs
 
Like a great painting, your logs need to paint a picture for the next person who looks at them. Most people dump the object or a breakpoint in their logs, but they’ll hate themselves in 2 years when they try to understand what was happening in their code. The logs _must_ give the next developer a birds-eye-view into the actions taken by the user up to a point in time. 

A generic log message provides just about as little information as no log message. Imagine if you had to go through your logs and you saw `purchase completed`. This doesn’t help you answer any questions. *When was the purchase completed? Who completed it? What did they buy?*

This can all be done by formatting your logs:
```python
logFormatter = '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
logging.basicConfig(format=logFormatter)
```

#### Structured Data

The _biggest_ issue we’ve seen is the pollution of log messages. People will put objects or variables into the log. Though it’s helpful while debugging your code, it’s impossible to decipher what you were thinking when looking at the stream in the future. The best way to deal with this is to push all your metadata to the `extra` object. Using this, you’ll be able to decipher your messages from the stream, while still grab the full context to dig deeper into your user from any messages.

Some of this data, such as `time` and `levelname` can be captured automatically, but if you can (and should) push `extra` context to your logs.

```python
logger = logging.getLogger(__name__)
logger.info('purchase completed', extra={‘user': 'Sid Panjwani'})
```

If you’re wondering why you do not see your logs, remember to change the `level`. By default, it’s set to `warning`.

*Yep, it’s that’s easy…*

### Tracking Exceptions through Logs

Some companies have made it their core competency helping you capture and react to exceptions. If you’re just a hobbyist, it’s pretty easy to capture your errors yourself and write them to your logs. 

```python
def captureException():
	# this should do something

try:
	1/0
except:
	captureException()
```

When capturing an exception, it’s essential to add context. Similar to when you’re debugging, you want to know who the user is and what they were doing when the exception struck.

```python
import sys

# remember to set up your logger

def captureException():
	logger.warning(sys.exc_info())
```

![](./images/loggingPython/exception1.png)

This gives us a `traceback` object that we can use to see the stack from our exception. We didn’t even need to pass anything into our `captureException` method. 

Let’s figure out what the `traceback` object gives us and clean up this information a little.

```python
import sys
import traceback

def captureException():
	r = list(sys.exc_info())
	
	e = dict()
	e["name"] = r[1]
```

![](./images/loggingPython/exception2.png)

Now, this can be cleaned up using some string manipulation in Python.

## Accessing those Logs

Now that you’ve created _gorgeous_ logs that give the next developer a view of the user’s actions, you need to know best practices for searching and reacting to these logs.

### Searching

Going through your logs from a file is a tedious process, it’s honestly better to use a [cloud-based solution](https://timber.io/) that acts as a layer of abstraction, so you don’t have to think about parsing colossal log files.

If you’re dead-set on searching through your files locally, Python is one of the best languages to do so. Here is an example that generalizes the text and log file:
```python
def search(input_filename, output_filename, text):
	# overwrite output file
	with open(output_filename, "w") as out_file:
		with open(input_filename) as in_file:
			# loop over each log line and check if text appears
			for line in in_file:
				if text in line:
					out_file.write(line)
```

This makes it easy to search for log messages that contain keywords such as `critical` or `warning`.

### Rotating Logs

Though not generally taught in logging guides, the Python logging module makes it easy to log in a different file after an interval of time or after the log file reaches a certain size.

This becomes useful if you automatically want to get rid of older logs, or if you're going to search through your logs by date, since you won’t have to search through a huge file to find a set of logs that are already grouped.

To create a `handler` that makes a new log file every day and automatically deletes logs more than five days old, you can use a `TimedRotatingFileHandler`. Here’s an example:

```python
logger = logging.getLogger('Rotating Log by Day')

# writes to pathToLog
# creates a new file every day because `when="d"` and `interval=1`
# automatically deletes logs more than 5 days old because `backupCount=5`
handler = TimedRotatingFileHandler(pathToLog, when="d", interval=1,  backupCount=5)
```

### Accessing Server Logs through SSH

When troubleshooting an issue on your server, you might have to check your error logs. The best way to do that is to SSH into your server.

If you don’t know how to SSH into your server, [here](https://help.dreamhost.com/hc/en-us/articles/216041267-SSH-overview) is a great guide that explains how.

Once you are connected, you can go to the `logs` folder. 
```bash
cd logs
```

 You can use `ls` to list all the types of logs you can view and `cd` into the relevant domain.

To see the last 20 logs from `error.log`:
```bash
tail error.log
```

To show new logs in real-time:
```bash
tail -f error.log
```

To search for a specific keyword in a log file, you can use the `grep` command.

To see all logs that have `warning` in `error.log`:
```bash
cat error.log | grep "warning"
```

## Wrapping Up

Logs can be a pain to deal with. That’s why we recommend a [cloud-based provider](https://timber.io/) that can deal with aggregating the logs and making them intuitive, but it’s difficult to try new technology when the payoff isn’t obvious. 

#### Source of Truth

If you take anything away from this post, it should be that logs serve as the _source of truth_ for the user’s actions. Even for ephemeral actions, such as putting an item into and out of a cart, it’s essential to be able to trace the user’s steps during a bug report and the logs allow you to trace their actions between all your [MicroServices](https://timber.io/blog/docker-and-the-rise-of-microservices/).

***Instead of doing this yourself, we’ve got a pretty awesome service [here @ Timber](https://timber.io/) (it’s seriously great) that automatically captures context with your logs to make debugging easier. Try us out for (completely) free; you don’t even need a credit card!***

![](./images/loggingPython/footer.png)