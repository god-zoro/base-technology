---
redirect_from: /_posts/2022-08-06-测试/
title: RedissonClient的getKeys方法耗时过久分析
tags:
  - Arthas
  - 性能分析
  - Redisson
---

### Trace命令监控总耗时与子方法耗时对不上
通过下图我们可以看到，Arthas监控下总耗时为300ms，但是所有子方法耗时监控下最大的才为87ms，还缺少了65%左右耗时不知所踪。
![arthas-redisson-pic1.png](https://i.postimg.cc/4y5PpKmY/arthas-redisson-pic1.png)

通过代码分析，我们可以看到最主要的时间耗时是消耗Stream流计算这里

![image.png](https://i.postimg.cc/fyB7J6rX/arthas-redisson-pic2.png)

通过追查源码我们可以发现流是通过Redisson提供的方法获取到的。

![image.png](https://i.postimg.cc/tCvtyGPG/arthas-redisson-pic3.png)

再往下追踪我们发现，该方法使用了Redis的SCAN命令

![image.png](https://i.postimg.cc/mgc3gpNz/arthas-redisson-pic4.png)

再往下走，我们发现这里使用到了异步操作。但是Arthas是不能够监控到异步线程的操作耗时。因此我们得知，耗时丢失的问题就是出现在了这里。
![image.png](https://i.postimg.cc/25h44qtC/arthas-redisson-pic5.png)

![image.png](https://i.postimg.cc/5y7vx8XS/arthas-redisson-pic6.png)

接下来我们做个验证。
使用<kbd>thread -n 3</kbd>命令来监控服务器最繁忙的3个线程就可以找到我们执行的方法了。
从监控中可知，我们出现问题的方法就是流的<kbd>collect</kbd>方法。追踪源码我们可以看到，方法执行的过程中一直有Redisson命令异步执行await方法，致使我们的线程卡死等待消耗了大量时间。因为执行<kbd>Redis的SCAN命令</kbd>是需要扫描全库，库里数据越多扫描时间越长等待的时间也就越长。

![image.png](https://i.postimg.cc/4NyVW20L/arthas-redisson-pic7.png)

![image.png](https://i.postimg.cc/fbndTzRp/arthas-redisson-pic8.png)