+++
date = "2018-02-18T21:31:20+08:00"
draft = false
title = "python stupid trick"
tags = ["python"]
+++

[python queue: put、get、task_done、join](https://docs.python.org/2/library/queue.html#Queue.Queue.join)

1. `Queue.put(item[, block[, timeout]])`:

    1. Put item into the queue. 
    2. If optional args block is true and timeout is None (the default), block if necessary until a free slot is available. 
    3. If timeout is a positive number, it **blocks at most timeout seconds and raises the Full exception if no free slot was available within that time**. 
    4. Otherwise (block is false), put an item on the queue if a free slot is immediately available, else **raise the Full exception (timeout is ignored in that case).**

2. `Queue.get([block[, timeout]])`:

    1. Remove and return an item from the queue. 
    2. If optional args block is true and timeout is None (the default), block if necessary until an item is available. 
    3. If timeout is a positive number, it **blocks at most timeout seconds and raises the Empty exception if no item was available within that time. **
    4. Otherwise (block is false), return an item if one is immediately available, else **raise the Empty exception (timeout is ignored in that case).**

3. `Queue.task_done()`:

    1. Indicate that a formerly enqueued task is complete. 
    2. **Used by queue consumer threads**. For each get() used to fetch a task, **a subsequent call to task_done() tells the queue that the processing on the task is complete.**
    3. If a join() is currently blocking, it will resume when all items have been processed (meaning that a task_done() call was received for every item that had been put() into the queue).
    4. Raises a ValueError if called more times than there were items placed in the queue.

4. `Queue.join()`：

    1. Blocks until all items in the queue have been gotten and processed.
    2. The count of unfinished tasks **goes up whenever an item is added to the queue**.
    3. The count **goes down whenever a consumer thread calls task_done()** to indicate that the item was retrieved and all work on it is complete.
    4. When the count of unfinished tasks drops to zero, join() unblocks.

[The KeyboardInterrupt Exception](http://effbot.org/zone/stupid-exceptions-keyboardinterrupt.htm):

1. If you try to stop a CPython program using Control-C, the interpreter throws a KeyboardInterrupt exception.
2. this is an ordinary exception, and is, like all other exceptions, caught by a “catch-all” try-except statement
3. if the update process isn’t an atomic operation in itself, it’s also a good idea to use database transactions, and roll back when something goes wrong

```python
for record in database:
    begin()
    try:
        process(record)
        if changed:
            update(record)
    except (KeyboardInterrupt, SystemExit):
        rollback()
        raise
    except:
        rollback()
        # report error and proceed
    else:
        commit()
```

[Singleton Pattern](https://py.windrunner.me/design-patterns/singleton.html):

装饰器：很方便的给不同的对象增添相同的功能

```python
def singleton(cls):
    instance = cls()
    instance.__call__ = lambda: instance
    return instance

@singleton
class Highlander:
    x = 100
    # Of course you can have any attributes or methods you like.

highlander = Highlander()
another_highlander = Highlander()
assert id(highlander) == id(another_highlander)
```

metaclass：希望的是有单例特性的类而不是为类添加单例限制

```python
class Singleton(type):
    def __init__(cls, name, bases, dict):
        super(Singleton, cls).__init__(name, bases, dict)
        cls._instance = None
    def __call__(cls, *args, **kw):
        if cls._instance is None:
            cls._instance = super(Singleton, cls).__call__(*args, **kw)
        return cls._instance

class MyClass(object):
    __metaclass__ = Singleton

one = MyClass()
two = MyClass()
```

[select](http://lvxiaoyu.com/static/posts/20170414.0.html)：

1. 检查多个waitable object并阻塞，直到它变成ready object为止，并返回相应的ready object列表
2. 接收四个参数：

    1. 用于读操作的 waitable object 列表
    2. 用于写操作的 waitable object 列表
    3. 表示异常的 waitable object 列表
    4. **超时时间：超时后select函数返回0，并且上述三个列表为空**

3. 返回三项返回值：

    1. 用于读操作的 ready object 列表
    2. 用于写操作的 ready object 列表
    3. 表示异常的 ready object 列表

```python
import socket
import select

s = socket.socket()
s.bind(('', 50007))
s.listen(5)
inputs = [s]
num = 0
while True:
    rs, ws, es = select.select(inputs, [], [])
    for r in rs:
        if r is s:
            c, addr = s.accept()
            print 'got conn from :', addr
            inputs.append(c)
        else:
            data = r.recv(1024)
            r.sendall('You send: ' +  data)
            if not data:
                inputs.remove(r)
            else:
                print 'receive data : ', data
```