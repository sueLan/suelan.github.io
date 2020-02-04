---
title: 理解手机GPS定位原理
date: 2019-06-08 17:29:06
categories:
 - LBS 
---

本文取材自 TED教学视频[How does your smartphone know your location? - Wilton L. Virgo](https://www.youtube.com/watch?v=70cDSUI4XKE)
我们的手机是如何准确定位的呢？ 答案在离我们2000多英里的卫星上。

第一个问题： 为啥卫星上的时间对GPS定位如此重要？ 因为我们要知道手机跟卫星的距离。 每个卫星在不断对外广播信号。信号中记录自己的创建时间。


![48bce392.png](/img/a9bb4ba0-9918-4e5c-9fc0-3b9a28363e6b/48bce392.png)

手机收到信号后，利用收到信号的到达时间， 计算自己跟卫星的距离: 

$$distance = c * \Delta T $$

c: 光的速度
$\Delta T$: 信号传播的时间间隔 $time_{arrive} - time_{create}$

但是c = 299,792,458 m/s, 非常大。 如果用秒做单位计算$\Delta T$， 地球上所有点，甚至有些远离我们的位置，计算出来的distance都是一样的值。 

![f8edfbb6.png](/img/a9bb4ba0-9918-4e5c-9fc0-3b9a28363e6b/f8edfbb6.png)


所以我们需要非常非常非常精确的时钟计算时间. 于是原子钟粉墨登场. 就像机械钟靠嘀嗒嘀嗒摇摆来计时， 原子钟也有这种间隔时间固定的滴答滴答计时方式。 

## 原子钟

![782d2620.png](/img/a9bb4ba0-9918-4e5c-9fc0-3b9a28363e6b/782d2620.png)
原子钟依赖原子跃迁所释放或吸收的电磁波来计时。原子在一个轨道跑了一圈后，跃迁到另一个轨道的时候会以电磁波形式释放出能量。

![a68ec7af.png](/img/a9bb4ba0-9918-4e5c-9fc0-3b9a28363e6b/a68ec7af.png)

能量计算公式： $\Delta E = h \nu$ $\Delta E$ 变化的能量; h是常量(h=6.626x$10^{-34}$ Js)。 $\nu$是频率。 比如铯原子钟，它的频率是个固定值 9,192,631,770Hz，你可以想象为一个每秒跑9billion圈的时钟。所以用原子钟可以每秒精确到 1/1billion。 

![0bc32fc3.png](/img/a9bb4ba0-9918-4e5c-9fc0-3b9a28363e6b/ca6ea804.png)
这张图的detector是用来检测probe laser固定频率发出的电磁波。


有了精确的时间，我们可以知道手机跟卫星的距离。 以卫星为中心，distance为半径画圆(球)

![f58c51ad.png](/img/a9bb4ba0-9918-4e5c-9fc0-3b9a28363e6b/f58c51ad.png)

如果多几个卫星，我们可以用几何公式计算出它们的空间交叉点

![293478f1.png](/img/a9bb4ba0-9918-4e5c-9fc0-3b9a28363e6b/293478f1.png)

![6cbd0ed3.png](/img/a9bb4ba0-9918-4e5c-9fc0-3b9a28363e6b/6cbd0ed3.png)

