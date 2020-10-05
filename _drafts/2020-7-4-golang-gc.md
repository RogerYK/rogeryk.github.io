---
layout: post
title: Tracing GC必须STW吗?
---

GC一般来说有两种实现方式，一种是通过引用计数，这种方法的好处是释放内存时比较平缓, 不需要将正在运行的程序挂起。缺点就是每次传递引用的都要更新计数，会带来带来额外的损耗，吞吐量可能会低些，此外会有循环引用的问题，因此应用较少。 因此比如Object-C中就使用基于引用计数的GC，其平滑的内存释放，非常适用GUI这种用户对暂停敏感的场景。 还有一种就是通过可达性分析的Tracing GC，在适当的时候，垃圾回收器会启动，从所有的根对象（栈上和静态对象）开始遍历，遍历完成后， 将所有没有访问到的对象释放。在遍历的过程中，通常下需要将正在运行的程序挂起，会有一定的暂停，不过没有了引用计数的损耗，吞吐量会更高。例如java中使用就是Tracing GC。由于Tracing GC使用的比较多，所以大家一谈到某个语言带了GC，就觉得会造成暂停，无法应用在低延迟的场景，比如游戏引擎，量化交易。当然，GC必然会带来一定的性能损耗，不过Tracing GC一定需要应用程序暂停吗？可否在不挂起程序的情况下进行GC，从而降低GC对程序延时的影响呢？比如目前java中最新推出的ZGC算法，保证最大暂停时间低于10ms。而在Golang 1.8中，甚至将大部分暂定控制在1ms下。本文主要基于Golang，研究为什么Golang可以将GC的暂停控制在1ms以内。  

## 三色标记
Tracing GC

## 参考
<https://www.zhihu.com/question/35109537/answer/61379522>
<https://www.zhihu.com/question/42353634>
<https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md>
<https://blog.golang.org/ismmkeynote>

