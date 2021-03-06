---
title: "同步与异步、阻塞与非阻塞"
date: 2018-10-10
---

## 0x00 引入问题

看《深入分析Java Web技术内幕》的时候，看到的同步与异步，阻塞与非阻塞的概念，这个问题之前在群里也有人说过，但是当时印象不深刻，也没有在意，现在在学后端的开发，还是需要把这些概念捋清楚。

先来看看书里面是怎么说的。用我自己的话进行了转述。

对于同步（synchronous）和异步（asynchronous）：
> 同步就是一个任务的完成需要依赖另一个任务，当被依赖的任务完成以后，依赖的任务才能完成，异步就是依赖的任务通知被依赖的任务去完成工作，不需要被依赖的任务完成，依赖的任务继续进行。从任务完成的角度看，同步就是需要两个任务都完成才算完成，异步是只要依赖的完成，那就是完成了。

对于阻塞（blocking）和非阻塞（non-blocking）：
> 阻塞和非阻塞的概念是针对CPU而言的，阻塞就是CPU停下来等待慢的任务完成，然后再去干其他的事情，而非阻塞就是慢的操作在执行时，CPU去做其他的事情。

## 0x01 现有的讨论

网上有一些不同的意见。

讨论1:在[stackoverflow](https://stackoverflow.com/questions/2625493/asynchronous-vs-non-blocking)上面，得票最多的人说：大部分情况下，同步与阻塞，异步与非阻塞其实说的都是一件事情，只有在某些语境下才有差异。比如举了`socket` API的例子，在`socket`中，1.阻塞与同步是一个意思，会一直挂着线程直到有返回值；2.非阻塞的意思是，如果调用的API还没有准备好（比如数据没有就绪等），那么就会返回错误，然后结束。需要等到就绪以后再调用API；2.异步的意思是，调用API以后，API所进行的操作就会到后台去运行，等有了结果再通过回调函数或者事件返回结果。

对`socket` API不是很了解，感觉也是说的很模糊，表述也有点问题，并不值得这么多赞。

讨论2:这是来源于一篇[博客](http://www.programmr.com/blogs/difference-between-asynchronous-and-non-blocking)，首先同步的概率有个很好的例子诠释：HTTP，它就是一个同步的协议，浏览器发送一个request，然后就会一直等着这个response，这个response不来，下一个request就不会发送出去，注意，这里说的是request的同步，也就是说每一个request的任务是按序进行的，上一个request结束才会进行下一个request。阻塞的概念可以看Java中的`InputStream.read()`方法（用来读取文件），调用这个方法以后，它会一直等到读取的数据有效（比如读到了结束符或者抛出异常）才会返回。这就是一个阻塞的过程。

同步调用通常需要一个回调或者事件，用来触发响应已经有效的信号。而对于非阻塞，它会立刻返回一个结果不管是不是有效，可能需要反复调用多次才能拿到有效数据，非阻塞一般说的就是IO，异步一般说的是更加广泛的操作。


## ox02 我的一些想法

我们先从阻塞和非阻塞入手，定义很明确，那就是是不是占着CPU不拉屎。现代的CPU，可以运行多任务，其实就是把任务的执行过程分成很多的细粒度，然后在任务的细粒度之间来回切换，其实每一时刻只进行一个任务，但是给你的错觉就是同时进行了很多的任务。因此阻塞不阻塞就是看当任务**不使用**CPU的时候，是不是让它释放出来。注意加粗的关键字，它另一层的意思就是，有些任务是不使用CPU的，比如说IO，这个[问题](https://stackoverflow.com/questions/13596997/why-is-the-cpu-not-needed-to-service-i-o-requests)下面有个回答提到，他说**硬盘有自己的专属芯片，用来出来CPU交代的任务，比如说读写等。**

所以我们常说的就是IO阻塞或者IO非阻塞，相对来说阻塞的概念比较狭隘，也比较底层。

而同步与异步其实是更宏观的角度去看待问题，从任务的角度去看，而且一般来说不是指底层原子任务。

同步IO或者阻塞IO都有这种说法，也都会对IO的性能产生影响，但是是不同的维度。从书中的例子我们也能看出来。

<p align="center"><img src="https://joeltsui-blog.oss-cn-hangzhou.aliyuncs.com/2018-10-10-blocking-synchrnous.png"/>
</p>

同步阻塞和同步非阻塞，说的就是到了IO操作这步以后，是不是要等着IO操作结果的返回。但是，**同步非阻塞增加CPU的消耗**，这点不是很懂，为什么会增加CPU的消耗？

当到了异步与同步的区别，就变得更宏观了，也就是说出现了几个任务，需要往分布式数据库中的多个数据库中写入同一条数据，向一个数据库中写数据就是一个任务，写到多个数据库中就是多个任务，多个任务并行就是异步。而每一个任务操作IO的时候都是阻塞进行。

异步非阻塞进行了扩展，原来异步阻塞中每次任务是写入一份数据，现在每一个任务要传输多份数据，IO也变成了网路IO。同时数据的传输量虽然不大，但是比较频繁，因此，在每个任务异步的同时，每一个机器的每一份数据也都是非阻塞，第一条数据发出去赶紧继续发第二条。

Anyway，你也可以说咬文嚼字，但是这么说下来，确实感觉更合理，也更清晰了。

以上。


