# Logging Guide Template

What to keep in mind?
1. Don't copy a guide, you know you truly understand the topic when you can explain it in basic English
2. Try to keep a lighthearted writing style
3. If you can easily find all the information elsewhere on the internet, our article will never rank (and there’s no point in writing it).
4. Remember to use pictures, it makes the article easier to read for skimmers and more approachable. LucidChart or searching Google for pictures Labeled for Reuse are both good options.

# Logging in `Language`: Becoming a More Effective Developer

Include a small blurb about why logging is important, make it catchy and grab the readers attention.

“_Just a disclaimer: we're a logging company here @ Timber. We'd love it if you tried out our product (it's seriously great!), but that's all we're going to advertise our product ... you guys came here to learn about logging in Python, and this guide won't disappoint._”

### Why bother?

What is the lazy approach to logging in `Language`?
* Like print or writing to the console

What are the drawbacks of the lazy approach? Such as:
* Have to comment out code
* difficult to differentiate between what’s important
* hard to see where the program is emitting the statement

Is there a logging module included with the language?
* Why is it better than the lazy approach (other than what you said above)
* What are its big features?

Where does logging fit in?
* Exceptions, metrics, etc. - when should you use these instead of logging (in a few words)
* Benefits around debugging

"""
As Martin Golding said:
> "Always code as if the guy who ends up maintaining your code will be a violent psychopath who knows where you live."
"""

### Hello World of Logging

“We’re believers that the best way to learn something is to do it, so open up your editor (yes, I’m serious) and get ready to learn.”

How do you set up the module?

What are the basics?
* What is the syntax

Logging to a File
* Why can’t you just log to the console?
* How do you log to a file?

## Writing Effective Logs

Levels

Legible Messages
* Put context into `extra`, the message should be easy to follow

Structured Logging
* What problem does it solve?
* How do you structure logs using the module?
* Give a couple common ways to format the logs

“If you don’t want to do this, we [here @ Timber](timber.io) automatically structure your logs so you can focus on what really matters. You can try out our product for completely free!”

Performance
* What is the performance hit of logging?
* Using `level` to ignore log messages

Logging to Timber

Maybe: Tracking Exceptions through Logs (if the module doesn’t already do this)
* “Some companies have made it their core competency helping you capture and react to errors. If you’re just a hobbyist, it’s pretty easy to capture your errors yourself and write them to your logs.”
* Show how they can capture the stack

## Accessing your Logs

Reading logs from File
* Parsing & Searching those Logs
* Timber makes this easy

Rotating Files

SSH into server with Logs

## Wrapping Up

"***Instead of doing this yourself, we’ve got a pretty awesome service [here @ Timber](timber.io) (it’s seriously great) that automatically captures context with your logs to make debugging easier. Try us out for (completely) free; you don’t even need a credit card!***”
