# Python 杂谈：with 与上下文管理器


本文简单介绍 Python 中的 with 与上下文管理器，以及如何自定义上下文管理器。
<!--more-->

大家应该都写过下面这样的代码：

```python
with open('file.txt', 'w') as f:
    f.write('Hello World')
```

上面的代码向文件 `file.txt` 中写入了字符 Hello World，`with` 语句会在代码块执行完后自动关闭文件。并且，无论这里的写文件操作是否成功，是否有异常，`with` 语句都会保证文件被关闭。

如果不使用 `with`，我们必须要像下面这样写，才能适当地处理异常：

```python
try:
    f = open('file.txt', 'w')
    f.write('Hello World')
finally:
    f.close()
```

很明显，`with` 语句更简洁。那么 `with` 语句背后发生了什么呢？

## 上下文管理器

文章开头代码中 `with` 语句的执行过程如下：

1. 执行 `open('file.txt', 'w')`，此函数返回一个文件对象；
2. 执行文件对象的 `__enter__` 方法，并将 `__enter__` 方法的返回值赋给变量 `f`；
3. 执行 `with` 语句块内部的代码体；
4. 不管执行过程中是否出现异常，执行执行文件对象的 `__exit__` 方法，在此方法中关闭文件。

此处 `with` 语句能够起到 `try...finally...` 语句的效果正是因为 `open` 函数返回的文件对象实现了 `__enter__` 方法和 `__exit__` 方法。一个对象如果实现了这两个方法，就称为**上下文管理器**。

在实际应用中，`__enter__` 一般用于分配资源，例如打开文件、连接数据库、获取线程锁等等；`__exit__` 一般用于释放资源，例如关闭文件、关闭数据库连接、释放线程锁等等。

## 自定义上下文管理器

我们当然也可以定义自己的上下文管理器，先来看一下 `__enter__` 方法的定义：

> `object.__enter__(self)`
>
> Enter the runtime context related to this object. The with statement will bind this method's return value to the target(s) specified in the as clause of the statement, if any.

简单来说就是，进入与 object 相关联的上下文。with 语句会将此方法的返回值赋给 as 关键字指定的目标。此方法的用处上面已经讲过了。下面看 `__exit__` 方法的定义：

> `object.__exit__(self, exc_type, exc_value, traceback)`
>
> Exit the runtime context related to this object. The parameters describe the exception that caused the context to be exited. If the context was exited without an exception, all three arguments will be None.

退出与 object 相关联的上下文。除 self 以外的三个参数共同描述了导致退出上下文的异常，如果没有异常发生，则三个参数均为 None。

> If an exception is supplied, and the method wishes to suppress the exception (i.e., prevent it from being propagated), it should return a true value. Otherwise, the exception will be processed normally upon exit from this method.
>
> Note that `__exit__()` methods should not reraise the passed-in exception; this is the caller's responsibility.

如果发生了异常，并且不想将异常传播到上下文之外的代码块，那么此方法应该返回 True。否则，异常将会在 with 语句块结束后被重新抛出，让 with 语句块之外的代码来处理。`__exit__()` 不应该再次抛出由参数传入的异常，这是调用者的职责，不归此方法管理。

现在我们就来试试自定义一个上下文管理器，我们定义一个专有的上下文管理，用于打开文件，每次打开都向里面写入当前操作时的系统时间，顺便打印一下函数的调用顺序：

```python
import datetime

class myOpen(object):
    def __init__(self, name, mode='r'):
        self.name = name
        self.mode = mode

    def __enter__(self):
        print("[In __enter__]: acquiring resources...")
        self.file = open(self.name, self.mode)
        self.file.write(
            datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S : '))
        return self.file

    def __exit__(self, exc_type, exc_value, traceback):
        print("[In __exit__]: releasing resources...")
        if self.file:
            self.file.close()
        if exc_type is None:
            print("[In __exit__]: exited without exception...")
            return True
        else:
            print("[In __exit__]: exited with exception: %s" % exc_value)
            return False

with myOpen('text.txt', 'a') as f:
    print("[In with body]: writing to file...")
    f.write('I love Python!')
```

控制台输出如下：

```bash
[In __enter__]: acquiring resources...
[In with body]: writing to file...
[In __exit__]: releasing resources...
[In __exit__]: exited without exception...
```

text.txt 文件中的内如下：

```plain
2019-03-21 22:31:43 : I love Python!
```

如果我们在 `with` 语句块中抛出一个异常，由于我们写的 `__exit__` 方法中，当有异常是返回 `False`，所以异常会被从新抛出。我们故意抛出一个异常试试：

```python
with myOpen('text.txt', 'a') as f:
    print("[In with body]: writing to file...")
    f.write('I love Python!')
    raise (Exception('Something is wrong!!!'))
```

运行结果：

```plain
[In __enter__]: acquiring resources...
[In with body]: writing to file...
[In __exit__]: releasing resources...
[In __exit__]: exited with exception: Something is wrong!!!
Traceback (most recent call last):
  File ".\001-custom_context_manager.py", line 31, in <module>
    raise (Exception('Something is wrong!!!'))
Exception: Something is wrong!!!
```

你可以尝试令 `__exit__` 方法始终返回 `True`，那么所有在 `with` 语句中发生的异常都会被忽略。

我们还可以使用上下文管理器处理很多事务，例如进入 `with` 语句块之前创建网络连接、获取锁等等。

