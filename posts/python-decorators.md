---
title: Prometheus: The Good, the Bad, and the Ugly
author: nhumrich
---

Debugging your Python code without changing it, using Decorators



Python decorators are a very powerful metaprogramming concept in python that allow you to change how a function or class behaves by simply putting an annotation on the definition. 
The main concept is to abstract away something that you want to function or class to also do, besides its normal function. 
This can be helpful for many reasons, such as code reuse, sticking to [curlys law](https://blog.codinghorror.com/curlys-law-do-one-thing/), and others. By learning how to write your own decorators, you can significantly improve readability of your own code. 
Decorators also can be used as a great debugging or testing tool, by changing how the function or class behaves, without needing to actually change the code, or add logging lines. It is common to see decorators in python code, commonly when using frameworks such as flask or click. 

# How it works
When you define a function, in python, that function becomes an object. If you write a function called `hello_world` in python, that function is actually just an object called `hello_world` that is a callable. Very similar to how a lambda would work, `hello = lambda: print(‘hello’)` would be the same as `def hello(): print(‘hello’)`. Python decorators work by wrapping that object at definition time, and returning a new object definition. Essentially, if you were decorating a lambda, it would be the equivalent of `hello = decorate(lambda: print(‘hello’)`. Decorate is passed the function, then can use that function however it wants, then returns another object. The sky is technically the limit. The decorator has the ability to swallow the function, or return something that is not a function, if it wanted. 

Here is an example of a decorator that does useless things, it simply swallows the function entirely, and always returns 5. This is mostly never what you want, but it’s a good example that a decorator is simply passed a function, and returns an object. 

```python
@my_decorator_that_just_returns_5
def hello():
    print(‘hello’)

>>> hello()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'int' object is not callable
'int' object is not callable

>>> hello
5
```


# Writing your own decorator
As mentioned, a decorator is simply a function that is passed a function, and returns an object. So to start writing a decorator, we just need to define a function. In most cases, we want the object our decorator returns, to actually mimic the function we decorated. That means that the object the decorator returns, needs to be a function itself. For example, let’s say we wanted to simply log every time the function was called, we could write a function that logs that information, then calls the function. But that function would need to be returned by the decorator. This usually leads to function nesting such as:

```python
def mydecorator(f):  # f is the function passed to us from python
    def log_f_as_called():
        print(f’{f} was called.’)
        f()
    return log_f_as_called
```

As you can see, we have a nested function definition, and the decorator function is returning that function. This way, the function `hello` can still be called like a standard function, the caller will be none the wiser that its decorated. Now, if we define hello as the following:

```python
@mydecorator
def hello():
    print(‘hello’)
```

We get the following output:
```python
>>> hello()
called <function hello at 0x7f27738d7510>
hello
```

You can also decorate a function many times, to do many things. In which case, its just a chain effect, essentially, the top decorator is passed the object from the former. So, if we have:

```python
@a
@b
@c
def hello():
    print(‘hello’)
```
The interpreter is essentially doing `hello = a(b(c(hello)))` and all the decorators would wrap each other. You can test this out by using our existing decorator, and just using it twice:

```python
@mydecorator
@mydecorator
def hello():
    print(‘hello’)

>>> hello()
called <function mydec.<locals>.a at 0x7f277383d378>
called <function hello at 0x7f2772f78ae8>
hello
```

You will notice that the first decorator, wrapped the second, and logged it separately. One interesting thing here, is that once `hello` was wrapped once, it no longer appeared as hello to the interpreter. That is fine for our trivial example, but can often break tests and things that might be trying to introspect the function attributes. If the idea is for a decorator to act like the function it decorates, it needs to also mimic that function. Luckily, there is a python standard library decorator `wraps` for that in functools. It tells the function to appear as the function it wraps. 

```python
import functools
def mydecorator(f): 
    @functools.wraps(f)   # we tell wraps that the function we are wrapping is f
    def log_f_as_called():
        print(f‘{f} was called.’)
        f()
    return log_f_as_called

@mydecorator
@mydecorator
def hello():
    print(‘hello’)

>>> hello()
@mydecorator
@mydecorator
def hello():
    print(‘hello’)

>>> hello()
called <function hello at 0x7f27737c7950>
called <function hello at 0x7f27737c7f28>
hello

```

Now, our new function appears just like the one its wrapping/decorating. However, we still are reliant on the fact that it returns nothing, and accepts no input. If we wanted to make that more general, we would need to pass in the arguments and return the same value. We can modify our function to look like this:


```python
import functools
def mydecorator(f): 
    @functools.wraps(f)   # we tell wraps that the function we are wrapping is f
    def log_f_as_called(*args, **kwargs):
        print(f‘{f} was called with arguments={args} and kwargs={kwargs}’)
        value = f(*args, **kwargs)
        print(f’{f} return value {value}’)
        return value
    return log_f_as_called
```

Now we are logging everytime the function is called, including all the inputs the function receives, and what it returns. Now you can simply decorate any existing function and get free debug logging, without needing to change the function at all. 

# Adding variables to the decorator
If you were using the decorator for any code that you wanted to ship, and not just for yourself locally, you might want to replace all the `print` statements with logging statements. In which case, you would need to define a log level. It might be safe to assume you always want the debug log level, but what if that depends on the function? We can provide variables to the decorator itself, to define how it should behave. For example:

```python
@debug(level=‘info’)
def hello():
    print(‘hello’)
```
This would allow us to specify that this specific function should log at the info level instead of the debug level. This is achieved in python by writing a function, that returns a decorator. Yes, a decorator is also a function. So this is essentially saying `hello = debug(‘info’)(hello)`. That double parenthesis might look funky, but basically, debug is function, that returns a function. Adding this to our decorator, we would need to nest one more time, now making our code look like this:

```python
import functools
def debug(level): 
    def mydecorator(f)
        @functools.wraps(f)   # we tell wraps that the function we are wrapping is f
        def log_f_as_called(*args, **kwargs):
            logger.log(level, f‘{f} was called with arguments={args} and kwargs={kwargs}’)
            value = f(*args, **kwargs)
            logger.log(level, f’{f} return value {value}’)
            return value
        return log_f_as_called
    return mydecorator
```

This is starting to get kind of ugly, and overly nested, so a cool little trick that I like to do is simply add a default kwarg `level` to `debug` and return a [partial](https://docs.python.org/2/library/functools.html#functools.partial).

```python
import functools
def debug(f=None, *, level=’debug’): 
    if f is None:
        return functools.partial(debug, level=level)
    @functools.wraps(f)   # we tell wraps that the function we are wrapping is f
    def log_f_as_called(*args, **kwargs):
        logger.log(level, f‘{f} was called with arguments={args} and kwargs={kwargs}’)
        value = f(*args, **kwargs)
        logger.log(level, f’{f} return value {value}’)
        return value
    return log_f_as_called
```
Now the decorator can work as normal:

```
@debug
def hello():
    print(‘hello’)
```
And use debug logging. Or, you could override the log level:
```python
@debug(‘warning’)
def hello():
    print(‘hello’)
```



