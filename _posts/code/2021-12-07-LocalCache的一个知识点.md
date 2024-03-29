---
layout: default
title: LocalCache的一个知识点
---

笔者曾协助调查，某服务部署时频繁发生full gc的原因。将调查过程记录下来，一来积累排查线上事故的经验。另外，由于这次事故是由于guava包中的cache使用不当造成的，希望通过本次事故，加深了对guava cache的一些认识。  

分析服务异常时dump的内存文件，发现大小约8G，给服务分配的内存大小也是8G，因此服务内存耗尽。  
<img src = "http://dbp-resource.cdn.bcebos.com/a1620f93-4200-9024-4be8-61a6751b1340/image%20%281%29.png" width = "800" height = "300"/>

和正常运行的服务dump内存相比，增加了约3G，增加的主要对象是byte[]。而且在这些byte[]对象中，存在大量容量和内容都一样的的byte[]，累计大小约为2G左右（每个对象容量为40,1891 个，累计共有4,900）。如图所示：  
<img src = "http://dbp-resource.cdn.bcebos.com/a1620f93-4200-9024-4be8-61a6751b1340/byte.png" width = "800" height = "350"/>

从上图还可以发现一个有意思的现象，byte[]都是直接被同一个类引用：scala.util.Success，而且对象的根节点是forkJoinPool对象，可以判断是异步调用成功返回的结果，放入了forkJoin线程池的任务队列中，但是没有来得及被处理，造成了堆积。为了验证这一猜想，可以查看内存中ForkJoinTask的数量。结果发现服务在正常运行状态下，ForkJoinTask数量为3,719，而在发生fullgc时，ForkJoinTask数量为20,197，可以证明线程池确实存在比较严重的堵塞了。这里要说明一下，ForkJoinTask只是个抽象类，我们应该去查看具体实例类的个数，笔者的项目是scala开发，因此具体task实例是scala.concurrent.impl.ExecutionContextImpl$AdaptedForkJoinTask。  

为了弄清楚为什么会出现大量的任务堆积，需要进一步分析线程栈信息。发现一共有26个线程，处于RUNABLE的线程有13个，其中处于WAITING状态的线程有13个，其中处于WAIT状态的线程栈信息如下所示：<br/>
<img src = "http://dbp-resource.cdn.bcebos.com/a1620f93-4200-9024-4be8-61a6751b1340/jstack.png" width = "800" height = "300"/>

可见线程都是被阻塞在同一个地方，即OpenplatformCommon类的fetchDataWithLocalCache方法处，更具体的来说，是在LocalCache类的get方法处。打开对应处的方法代码，如下所示：<br/>
<img src = "http://dbp-resource.cdn.bcebos.com/a1620f93-4200-9024-4be8-61a6751b1340/openplatform-code.png" width = "800" height = "300"/>

其中skillsByKeyCache是个google guava包中的cache类。代码大意是首先从本地拿，本地没有的话，就会调用call方法，去远程获取。由于远程调用是异步返回的，而call方法需要获取一个结果值，所以调用了Await.result方法，同步等待方法300ms返回。同时从服务的error日志中，发现在Await.result地方报了较多的超时异常。从线程栈中大量线程阻塞在future.get，推测当一个key被多个线程同时去读取时，只有一个线程会去更新key所对应的value，其他的线程都在等待，导致线程都被阻塞而不能被用于其他任务的调度，进而造成任务的堆积，发生了FULL GC。写一个小Demo来验证猜想，发现运行确实如此。<br/>
<img src = "http://dbp-resource.cdn.bcebos.com/a1620f93-4200-9024-4be8-61a6751b1340/test.png" width = "800" height = "350"/>

至此，问题的排查链路比较清晰了。整个经过应该是：多个线程同时请求LocacleCache同一个key -> 更新线程耗时过长导致其他线程同步阻塞 -> 线程阻塞导致任务堆积 -> 任务堆积导致内存增长发生FULL GC。反思一些，是对如何使用guava cache 不熟造成的，对于这个，我准备在另一篇文章中专门探究一下。那么应该如何解决这个问题呢。笔者采取的办法是热身，在服务启动之前，将一些热点的key提前放入到LocalCache中，这个非常有效，问题解决了。  
