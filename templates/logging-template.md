# Logging Guide Template

What to keep in mind?
1. Don't copy a guide, you know you truly understand the topic when you can explain it in basic English
2. Keep a lighthearted writing style
3. If you can easily find all the information elsewhere on the internet, our article will never rank (and there’s no point in writing it).
4. Remember to use pictures, it makes the article easier to read for skimmers and more approachable. LucidChart or searching Google for pictures "Labeled for Reuse" are both good options.

## SEO Tips

#### Title

* Keep the title under 60 characters long

Need help picking your title?
1. Google what you think would be a good title.
2. Take the first 3 links and plug them into the [ahrefs site explorer](https://ahrefs.com/site-explorer).
3. Go to the organic search tab and look @ the "Top 5 Organic Keywords".
4. Base your title off something like this.

#### Meta Description

This is your 'Preview' in Contentful

* Keep this under 300 characters so Google doesn't shorten it
* Readers will use this as a proxy for if they should click on your article
* The more appealing this makes your article look, the higher you will rank

Need help picking your meta description?
* If ads show up when you Google, look at what they're doing. They're likely spending a lot of money AB testing meta descriptions.
* Make sure to look at what the content rises to the top of your search term. It's there for a reason.

#### Content

Below this, there is already a great guide you can follow to serve as an outline for your content.

Remember that Google's job is to show users the best answer to the search query. Without any SEO work, this definitely won't happen ... but it's a prerequisite of doing SEO work.

# Attention-Grabbing Title: The Ultimate Guide to Logging in {Language}
#### i.e. The Pythonic Guide to logging

Small blurb about why logging is important, make it catch and grab the readers attention.
_When done properly, logs are a valuable component of your observability suite. Logs tell the story of how data has changed in your application. They let you answer questions such as: Why did John Doe receive an error during checkout yesterday? What inputs did he provide?

The practice of logging ranges from very simple static string statements to rich structured data events. In this article, we'll explore how logging in {Language} will give you a better view of what's going on in your application. In addition, I'll explore some best practices that will help you get the most value out of your logs._

### The Lazy approach

What is the lazy approach to logging in `Language`?
* i.e. print / writing to the console

**Real World Example**

What are the drawbacks of the lazy approach? Such as:
* Have to comment out code
* Can only output to `stdout`
* Doesn't capture context
* difficult to differentiate between what’s important

## Hello World to Logging

Is there a logging module included with the language?
* What are its big features?
* What does it do that the lazy-approach couldn't?

**Real World (basic) Example**

### Levels

Why do you want a leveled logger? What levels are there?

**Example using Levels**

When do you use each level?

**Example showing how to set Level**

### Formatting the Logs

Why do you want to format your logs?
_The default formatter is not useful for formatting your logs, because it doesn't include critical information. The logging module makes it easy for you to add it in by changing the format._

**Example**

### Adding Context to Logs

Why do you want contextual logs?
_A generic log message provides just about as little information as no log message. Imagine if you had to go through your logs and you saw `removed from cart`. This doesn’t help you answer relevant questions such as: When was the item removed? Who removed it? What did they remove?_

How do you push context to logs?

**Example**

### Performance

_Logging introduces overhead that needs to be balanced with the performance requirements of the software you write. While the overhead is generally negligible, bad practices and mistakes can lead to [unfortunate situations](https://www.youtube.com/watch?v=uyIlAO390v4)._

What can you do affect performance less?
ie. wrap expensive calls in level check, avoid logging in the hot path, etc.

**Examples for Each**

### Logging to a File

**Example**

How do you search the log?
* `grep` probably works

### Rotating Logs

Why do you want Rotating Logs?
_This becomes useful if you automatically want to get rid of older logs, or if you're going to search through your logs by date since you won’t have to search through a huge file to find a set of logs that are already grouped._

**Example**

## The Drawbacks

Logs should only be one tool you reach for when observing your application. Take a look at the 3 pillars of observability.
_It’s important to understand where logs fit in when it comes to observing your application. I recommend readers take a look at the [3 pillars of observability](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html) principle, in that logs should be one of a few tools you reach for when observing your application. Logs are a component of your observability stack, but metrics and tracing are equally so.

**Logging can become expensive**, especially at scale. Metrics can be used to aggregate data and answer questions about your customers without having to keep a trace of every action, while tracing enables you to see the path of your request throughout your platform._

## Wrapping Up

Key takeaways.
_If you take anything away from this post, it should be that logs serve as the **source of truth** for the user’s actions. Even for ephemeral actions, such as putting an item into and out of a cart, it’s essential to be able to trace the user’s steps for a bug report, and the logs allow you to determine their actions._


Major features of the logging module.

***Instead of doing this yourself, we’ve got a pretty awesome service [here @ Timber](https://timber.io/) (it’s seriously great) that automatically captures context with your logs to make debugging easier. Try us out for (completely) free; you don’t even need a credit card!***

### Logging to the Cloud

_Writing your logs to a hosted log aggregation service seriously makes your life easier, so you can focus your time on what’s important._

_Disclaimer: I’m a current employee @ Timber. This section is entirely optional, but I sincerely believe that logging to the cloud will make your life easier (and you can try it for completely free)._

**Example**

_**That’s it.** All you need to do is get your API key from [timber.io](https://timber.io/), and you’ll be able to see your logs. We automatically capture them from the logging module, so you can continue to follow best-practices and log normally, while we seamlessly add context._
