---
title:  "python装饰器介绍"
date: 2015-08-04 19:13:04
excerpt: "python装饰器介绍"
tags: [decorator]
---

这篇文章主要介绍自己对python装饰器的一些理解。

### 装饰器的背景

首先装饰器是一个跟python无关的概念，目的就是希望在不改变原有的函数/类的前提下，能够附加上一些功能。而这些附加功能是通用的，也就是可能不止作用于一个函数/类。举个例子，比如有几个现成的函数，我们希望在每个函数的开始/结尾处打印log，记录每个函数的时间等。有一个办法就是在调用这些函数的地方增加上述代码，或者修改函数内部增加代码。在python里则可以使用装饰器这一语法糖轻松的实现。

<!--more-->

### 最简单的版本

```
def hello():
    print 'hello world!'


def decorator(func):
    print 'before %s' % func.__name__
    func()
    print 'after %s' % func.__name__

decorator(hello)
```

hello是我们最开始的函数，通过decorator对hello执行前后进行了加工，输出也都符合预期。但是有个问题是：我们修改了每个调用hello的地方，改为decorator(hello)，是否有办法可以不修改调用处的语句？

### 重新定义原函数

```
def hello():
    print 'hello world!'


def decorator(func):
    def _decorator():
        print 'before %s' % func.__name__
        func()
        print 'after %s' % func.__name__
    return _decorator

hello = decorator(hello)
hello()
```

只要重新定义hello这个变量（函数名/函数指针），那么我们不需要修改调用hello函数的地方了。我们在decorator函数下，又实现了一层\_decorator，hello重新指向了这个函数，就达到我们的目的了。\_decorator也可以返回func()的函数值，这样跟func就更接近了。

### python里的实现

上述代码对应python里的实现

```
def decorator(func):
    def _decorator():
        print 'before %s' % func.__name__
        func()
        print 'after %s' % func.__name__
    return _decorator


@decorator
def hello():
    print 'hello world!'

hello()
```

@只是一个语法，`@decorator def func(): ...`和`def func(): ... func = decorator(func)`并没有什么区别，只不过更直观的指出了这是一种装饰器。

### 函数带参数

```
def decorator(func):
    def _decorator(a, b):
        print 'before %s' % func.__name__
        result = func(a, b)
        print 'after %s' % func.__name__
        return result
    return _decorator


@decorator
def add(a, b):
    print 'add called.'
    return a + b

print add(1, 2)
```

函数带参数、返回值的情况只要在第二层的函数里对应的实现就可以了。

### 函数带任意参数

```
def decorator(func):
    def _decorator(*args, **kwargs):
        print 'before %s' % func.__name__
        result = func(*args, **kwargs)
        print 'after %s' % func.__name__
        return result
    return _decorator


@decorator
def add1(a, b):
    print 'add1 called.'
    return a + b


@decorator
def add2(a, b, c):
    print 'add2 called.'
    return a + b + c

print add1(1, 2)
print add2(1, 2, 3)
```

### 装饰器带参数

带参数的装饰器又额外在外面实现了一层，可以认为`decorator_args(arg)`与之前的`decorator(func)`是等价的，实现上也可看到,decortor\_args(arg)实际上返回了一个两层的decorator.
可以看做`myfunc = _decorator(func) = _decorator_args(arg)(func)`

```
def decorator_args(args):
    def decorator(func):
        def _decorator(a, b):
            print 'before %s args[%s]' % (func.__name__, args)
            result = func(a, b)
            print 'after %s args[%s]' % (func.__name__, args)
            return result
        return _decorator
    return decorator


@decorator_args('arg1')
def add(a, b):
    print 'add called.'
    return a + b


@decorator_args('arg2')
def add100(a, b):
    print 'add called.'
    return a + b


print add(1, 2)
print add100(1, 2)
```


### 问题

回到这个代码里来，hello经过装饰后有什么潜在的问题吗？

```
def decorator(func):
    def _decorator():
        print 'before %s' % func.__name__
        func()
        print 'after %s' % func.__name__
    return _decorator


@decorator
def hello():
    print 'hello world!'

print hello.__name__
```

输出是`_decorator`  

可以看到\_\_name\_\_并不是hello了，我们可以在\_decorator的实现里加上`_decorator.__name__ = func.__name__`，但是对于`__doc__`这一类的呢？


### 如何保留原有函数的属性


这个时候就需要我们的functools登场了

```
import functools


def decorator(func):
    @functools.wraps(func)
    def _decorator():
        print 'before %s' % func.__name__
        func()
        print 'after %s' % func.__name__
    return _decorator


@decorator
def hello():
    print 'hello world!'

print hello.__name__
```

装饰过的函数的属性被保留了下来。


### 一个例子

这个例子是从伯乐在线上的一片文章摘抄过来的，主要功能是对函数的执行时间计时。

```
import time
import functools


def logged(time_format):
    def decorator(func):
        @functools.wraps(func)
        def decorated_func(*args, **kwargs):
            print '%s called on %s' %\
                (func.__name__, time.strftime(time_format))
            start_time = time.time()
            result = func(*args, **kwargs)
            end_time = time.time()
            print '%s finised, executed time = %0.3fs' %\
                (func.__name__, end_time - start_time)
            return result
        return decorated_func
    return decorator


@logged('%b %d %Y - %H:%M:%S')
def add1(x, y):
    time.sleep(1)
    return x + y


@logged('%H:%M:%S')
def add2(x, y):
    time.sleep(2)
    return x + y

add1(1, 2)
add2(1, 2)
```

输出

```
add1 called on Aug 04 2015 - 20:04:35
add1 finised, executed time = 1.003s
add2 called on 20:04:36
add2 finised, executed time = 2.005s
```
