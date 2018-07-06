---
title: Debugging Python code using Decorators
author: nhumrich
---

Debugging your Python code without changing it, using Decorators



Python decorators are a very powerful concept in python that allow you to "wrap" a function with another function. 
The main concept of a decorator is to abstract away something that you want to function or class to also do, besides its normal responsibility. 
This can be helpful for many reasons such as code reuse, and sticking to [curlys law](https://blog.codinghorror.com/curlys-law-do-one-thing/). 
By learning how to write your own decorators, you can significantly improve readability of your own code. 
Decorators can also be used as a great debugging or testing tool. They can change how the function behaves, without needing to actually change the code (such as adding logging lines). 
Decorators are a fairly common thing in python, familiar to those who use frameworks such as flask or click, but many only know how to use them, but not how to write their own.

# How it works
First off, let's show an example of a decorator in python. The following is a very basic example of what a decorator would like like if you were using it.

```python
@my_decorator
def hello():
    print('hello')
```

When you define a function in python, that function becomes an object. 
The function `hello` above is a function object. The `@my_decorator` is actually a function that has the ability to use the `hello` object, and return a different object to the interpreter. The object that the decorator returns, is what becomes known as `hello`. Essentially, it's the same thing as if you were going to write your own normal function, such as: `hello = decorate(hello)`. Decorate is passed the function -- which it can use however it wants -- then returns another object. The decorator has the ability to swallow the function, or return something that is not a function, if it wanted. 


# Writing your own decorator
As mentioned, a decorator is simply a function that is passed a function, and returns an object. So, to start writing a decorator, we just need to define a function. 

```python
def my_decorator(f):
    return 5
```

Any function can be used as a decorator. In this example the decorator is passed a function, and returns a different object. It simply swallows the function it is given entirely, and always returns 5. This is a very trivial example, but it's a good example of the capabilities of a decorator. 

```python
@my_decorator
def hello():
    print('hello')

>>> hello()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'int' object is not callable
'int' object is not callable

>>> hello
5
```

Because our decorator is returning an int, and not a `callable`, it can not be called as a function. Remember, the decorator's return value _replaces_ `hello`. 

In most cases, we want the object our decorator returns, to actually mimic the function we decorated. _This means that the object the decorator returns, needs to be a function itself_. For example, let's say we wanted to simply print every time the function was called, we could write a function that prints that information, then calls the function. But that function would need to be returned by the decorator. This usually leads to function nesting such as:

```python
def mydecorator(f):  # f is the function passed to us from python
    def log_f_as_called():
        print(f'{f} was called.')
        f()
    return log_f_as_called
```

As you can see, we have a nested function definition, and the decorator function is returning that function. This way, the function `hello` can still be called like a standard function, the caller does not need to know that it is decorated. We can now define `hello` as the following:

```python
@mydecorator
def hello():
    print('hello')
```

We get the following output:
```python
>>> hello()
<function hello at 0x7f27738d7510> was called.
hello
```

_(Note: the number inside the `<function hello at 0x7f27738d7510>` reference will be different for everyone, it represents the memory address)_

# Wrapping functions correctly

A function can be decorated many times, if desired. In this case, the decorators cause a chain effect. Essentially, the top decorator is passed the object from the former, and so on and so forth. For example, if we have the following code:

```python
@a
@b
@c
def hello():
    print('hello')
```
The interpreter is essentially doing `hello = a(b(c(hello)))` and all of the decorators would wrap each other. You can test this out yourself by using our existing decorator, and using it twice.

```python
@mydecorator
@mydecorator
def hello():
    print('hello')

>>> hello()
<function mydec.<locals>.a at 0x7f277383d378> was called.
<function hello at 0x7f2772f78ae8> was called.
hello
```

You will notice that the first decorator, wrapped the second, and printed it separately. 

One interesting thing you might have noticed here, is that first line printed said `<function mydec.<locals>.a at 0x7f277383d378>` instead of what the second line printed, which is what you would expect: `<function hello at 0x7f2772f78ae8>`. The is because the object returned by the decorator is a new function, not called `hello`. This is fine for our trivial example, but can often break tests and things that might be trying to introspect the function attributes. If the idea is for a decorator to act like the function it decorates, it needs to also mimic that function. Luckily, there is a python standard library decorator called `wraps` for that in functools module.  

```python
import functools
def mydecorator(f): 
    @functools.wraps(f)  # we tell wraps that the function we are wrapping is f
    def log_f_as_called():
        print(f'{f} was called.')
        f()
    return log_f_as_called

@mydecorator
@mydecorator
def hello():
    print('hello')

>>> hello()
<function hello at 0x7f27737c7950> was called.
<function hello at 0x7f27737c7f28> was called.
hello
```

Now, our new function appears just like the one its wrapping/decorating. However, we are still reliant on the fact that it returns nothing, and accepts no input. If we wanted to make that more general, we would need to pass in the arguments and return the same value. We can modify our function to look like this:


```python
import functools
def mydecorator(f): 
    @functools.wraps(f)  # wraps is a decorator that tells our function to act like f
    def log_f_as_called(*args, **kwargs):
        print(f'{f} was called with arguments={args} and kwargs={kwargs}')
        value = f(*args, **kwargs)
        print(f'{f} return value {value}')
        return value
    return log_f_as_called
```

Now we are printing everytime the function is called, including all the inputs the function receives, and what it returns. Now you can simply decorate any existing function and get debug logging on all its inputs and outputs without needed to manually write the logging code. 

# Adding variables to the decorator
If you were using the decorator for any code that you wanted to ship, and not just for yourself locally, you might want to replace all the `print` statements with logging statements. In which case, you would need to define a log level. It might be safe to assume you always want the debug log level, but what that also might depend on the function. We can provide variables to the decorator itself, to define how it should behave. For example:

```python
@debug(level='info')
def hello():
    print('hello')
```
The above code would allow us to specify that this specific function should log at the info level instead of the debug level. This is achieved in python by writing a function, that returns a decorator. Yes, a decorator is also a function. So this is essentially saying `hello = debug('info')(hello)`. That double parenthesis might look funky, but basically, debug is function, that returns a function. Adding this to our existing decorator, we would need to nest one more time, now making our code look like the following:

```python
import functools
def debug(level): 
    def mydecorator(f)
        @functools.wraps(f)
        def log_f_as_called(*args, **kwargs):
            logger.log(level, f'{f} was called with arguments={args} and kwargs={kwargs}')
            value = f(*args, **kwargs)
            logger.log(level, f'{f} return value {value}')
            return value
        return log_f_as_called
    return mydecorator
```

The changes above turn `debug` into a function, which returns a decorator which uses the correct logging level. This is starting to get a bit ugly, and overly nested. A cool little trick to solve that, which I like to do, is simply add a default kwarg `level` to `debug` and return a [partial](https://docs.python.org/2/library/functools.html#functools.partial). A partial is a "non-complete function call" that includes a function and some arguments, so that they are passed around as one object without actually calling the function yet. 

```python
import functools
def debug(f=None, *, level='debug'): 
    if f is None:
        return functools.partial(debug, level=level)
    @functools.wraps(f)   # we tell wraps that the function we are wrapping is f
    def log_f_as_called(*args, **kwargs):
        logger.log(level, f'{f} was called with arguments={args} and kwargs={kwargs}')
        value = f(*args, **kwargs)
        logger.log(level, f'{f} return value {value}')
        return value
    return log_f_as_called
```
Now the decorator can work as normal:

```
@debug
def hello():
    print('hello')
```
And use debug logging. Or, you could override the log level:
```python
@debug('warning')
def hello():
    print('hello')
```
