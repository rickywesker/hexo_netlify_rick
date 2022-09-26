---
title: Python Asynchronous Programming Part One
date: 2022-09-26 15:06:40
tags:
  - 编程
  - Python标准库
  - 异步编程
categories:
  - 知识蒸馏

---

![a](https://pic2.zhimg.com/v2-6e24040bcefef3325dd2493f99013257_1440w.jpg?source=172ae18b)

# Python Asynchronous Programming Part One

## Motivation

很多时候我们编写的程序都是串行的，也就是说我们运行的顺序和我们书写主函数中的代码顺序是一致的，例如下面一段程序：

```python
def foo():
  print("I am a fool, but I love you.")
	return
def bar():
  print("Take me out, let us go to the bar.")
  return
def main():
  foo()
  bar()

main()
```

运行的结果就是

```python
[1] I am a fool, but I love you.
[2] Take me out, let us go to the bar.
```

也就是“做完这个做下一个”的方法。

However这样的方法在处理某些业务问题的时候显得有点太“悠然自在”，例如你需要向一个服务器发出请求接受数据，而服务端为了保持稳定对你作出了2秒一次请求的限制，在这两秒里，如果是以Synchronous（同步）的方式，那就是“硬等”，这两秒其实对于计算机的线程来讲是空闲的。那么这两秒我们是否可以用来做其他事情呢？答案是肯定的，不然也不会有这篇从入门到入坟的asyncio文章。

## 如何使用

关于堵塞和非堵塞通信，以及同步异步的教科书概念这里就不一一阐述，有兴趣的朋友可以自行翻阅 《操作系统》。

### Asyncio里的 Async/Await

首先我们需要了解一下什么是Coroutine，中文是协程，也就是我们可以把一个**Thread线程通过分时复用的方式运行多个协程**。

![a](https://pic2.zhimg.com/v2-6e24040bcefef3325dd2493f99013257_1440w.jpg?source=172ae18b)

也就是我们上文所说的，如何分配那两秒的等待时间给其他协程进行调度。

### 直接上手

假设你是一名摆大的学子，你大一，你很野，同时选了高数线代各种必修课专业课，期末考到了你才开始预习，但你的习惯是，看一会儿书，然后看一会儿手机以这种自我奖励机制消除疲惫感，我们怎么实现这个逻辑呢？假设你学习一个章节需要一小时，而你放松的时间是25分钟，这里我们用1和0.25秒电脑时钟时间来表示。我们先看完整代码。

```python
import asyncio
import random
behavior = ['Weibo','Wechat','JoyHub','Twitter','Instagram']

async def review_for_finals():
    print('Start studying....\n')
    await asyncio.sleep(1)
    print('Start studying lagrange multiples...\n')
    await asyncio.sleep(1)
    print('Start studying Vandermore Determinant...\n')
    await asyncio.sleep(1)
    print('Done Studying, going to 理教...\n')
    return {'Maths_score':60,'Linear_algebra':67}

async def play_your_phone():
    for i in range(10):
        print("Playing " + behavior[random.randint(0,4)])
        await asyncio.sleep(0.25)

async def main():
    task1 = asyncio.create_task(review_for_finals())
    task2 = asyncio.create_task(play_your_phone())
    value = await task1
    print("Your final score: ")
    print(value)
    await task2

asyncio.run(main())
```

程序的输出是：

```python
Start studying....

Playing Wechat
Playing Instagram
Playing Weibo
Playing Weibo
Start studying lagrange multiples...

Playing Twitter
Playing Wechat
Playing JoyHub
Playing Twitter
Start studying Vandermore Determinant...

Playing Wechat
Playing Twitter
Done Studying, going to 理教...

Your final score: 
{'Maths_score': 60, 'Linear_algebra': 67}
```

可见，我们合理地利用了1小时，就是说在这1小时里，你同时看书和刷手机。如果是同步的逻辑，我们需要完成1小时学习之后才能进行25分钟的娱乐，但在异步机制里，你在学习的过程中还合理利用了奖励机制，刷了好多微博甚至还上了一下Joyhub！真有你的，摆大学子！

### 代码分析

首先，我们的所有函数都使用了``async`` 关键字去声明异步函数，而每个异步函数里又包含了``await`` 关键字，这个await后面跟随的是一个coroutine协程，说明我们要进行 **异步等待** ，等待的过程中我们会把资源分配给其他任务，也就是我们的 **Task**，这个task我们通过 asyncio的 ``create_task``来构造。 我们看看输出，我们 开始学习的时候会进行1小时（1秒电脑时间），此时进行等待，资源被调度到 ``play_your_phone``，而每次娱乐又会等待25分钟，所以我们看到的输出是一次学习紧接着4次娱乐（也就是说我们压根没学习？？），然后又会想起要学习，4次娱乐直到娱乐task结束（总数10）。（真实）

## 结论

这只是一个sample code来测试一下asyncio的用法，之后了解到更多高级用法再继续更新，以上！
