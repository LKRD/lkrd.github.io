---
title: Linux Input Subsystem -- MultiTouch Protocol
date: 2018-01-25 16:27:48
tags:
---
&emsp;&emsp;**A协议和Slot(B)协议的区别**：从Device获取的当前数据与上一个数据相同，如果我们不管两次数据是否一致都上报，那就是A协议；如果我们选择不上报，通过event slot将上一次数据保存起来与每一次数据进行比较，这就是Slot(B)协议。
&emsp;&emsp;需要注意的是，想要测试Device驱动的input部分是否正常的时候，假如使用的是B协议，input_report数据的时候要记得每次都要report不同的值，否则在HAL层是看不到数据不停上报的，因为前后两个数据相同的时候，B协议是不会上报到系统的。另外，在上层测试数据上报频率的时候，采用数据总量/时间差的方法，如果驱动采用的是B协议，测试结果也是不准确的。
&emsp;&emsp;**应用场景上的使用区别：TP的多点触摸**
&emsp;&emsp;A协议不会使用slot，多指处理中，它的报点序列如下：
```bash
ABS_MT_POSITION_X x[0]
ABS_MT_POSITION_Y y[0]
SYN_MT_REPORT
ABS_MT_POSITION_X x[1]
ABS_MT_POSITION_Y y[1]
SYN_MT_REPORT
…
SYN_REPORT
```
&emsp;&emsp;上面的序列中需要说明的是在每个数据包的结尾用input_my_sync()对多个触控包进行分割，会产生一个SYN_MT_REPORT事件，它负责通知系统接收当前的触控信息并准备接收下一个信息。之后调用input_sync()函数来标记一个多点触摸数据传送结束，这会通知系统对从上一个EV_SYN/SYN_REPORT以来的所有累加事件作出相应，并准备接收新的一组事件/数据包。A协议比较简单，我们也可以发现在上面的序列中根本就没有轨迹跟踪的信息，有的只是点坐标等信息，那么系统如果去判断当前的多个点各属于哪一条线呢？
&emsp;&emsp;我们假设前一次事件共有5个点，本次触摸也有5个点，系统会分别计算前一次5个点与本次5个点的距离，distance[prev_i, curr_j] (i=0,1,...,4; j=0,1,...4)，这样会产生总共5*5=25个数字。然后对这25个数字进行排序，android用的是堆排序。下面的任务就是判断哪些当前点与前一次的点最近，那么赋予它们相同的id，应用收到这个信息后，就可以知道当前点属于哪条线了。
&emsp;&emsp;手抬起来的时候又用什么样的序列来通知系统呢，实际上在移动其中一个触控点后的上报序列看起来是一样的，所有存在触控点的原始数据被发送，然后在它们之间用SYN_REPORT进行同步。
&emsp;&emsp;当第一个接触点离开后，事件序列如下：
```bash
ABS_MT_POSITION_X x[1]
ABS_MT_POSITION_Y y[1]
SYN_MT_REPORT
SYN_REPORT
```
&emsp;&emsp;当第二个接触点离开后，事件序列如下：
```bash
SYN_MT_REPORT
SYN_REPORT
```
&emsp;&emsp;假如驱动在ABS_MT事件之外上报一个BTN_TOUCH 或ABS_PRESSURE事件，最后一个SYN_MT_REPORT可以省略掉，否则，最后的SYN_REPORT会被input核心层扔掉，结果就是一个0触控点事件被传到用户空间中。

&emsp;&emsp;B协议使用了slot，还有一个新面孔TRACKING_ID，先看下整个事件序列如下：
```bash
ABS_MT_SLOT 0
ABS_MT_TRACKING_ID 45
ABS_MT_POSITION_X x[0]
ABS_MT_POSITION_Y y[0]
ABS_MT_SLOT 1
ABS_MT_TRACKING_ID 46
ABS_MT_POSITION_X x[1]
ABS_MT_POSITION_Y y[1]
SYN_REPORT
```
&emsp;&emsp;id 45的触控点在x方向移动后的事件序列如下：
```bash
ABS_MT_SLOT 0
ABS_MT_POSITION_X x[0]
SYN_REPORT
```
&emsp;&emsp;slot 0对应的接触点离开后，对应的事件序列如下：
```bash
ABS_MT_TRACKING_ID -1
SYN_REPORT
```
&emsp;&emsp;上一个被修改的slot也是0，所以ABS_MT_SLOT被省略掉。这一消息移除了接触点45相关联的slot 0，于是接触点45被销毁，slot 0被释放后可以被另一个接触点重用。
&emsp;&emsp;最后，第二个接触点离开后的时间序列如下：
```bash
ABS_MT_SLOT 1
ABS_MT_TRACKING_ID -1
SYN_REPORT
```
&emsp;&emsp;我们可以看到对于B协议设备的驱动，在每个数据包的开始，通过调用input_mt_slot()进行分割，同时带入一个参数：slot，这会产生一个ABS_MT_SLOT事件，它通知接受者准备更新给定的slot的信息。传送的结束与A协议一样。Slot协议需要使用到ABS_MT_TRACKING_ID，它要不由硬件提供，要不通过原始数据进行计算。
&emsp;&emsp;没有SYN_MT_REPORT，那么它用什么来跟踪当前点属于哪一条线呢，用的就是ABS_MT_TRACKING_ID，当前序列中某点的ID值，如果与前一次序列中某点的ID值相等，那么他们就属于同一条线。既然如此，那么android系统中还需要做排序等运算吗？当然不需要。
&emsp;&emsp;这里上报的ABS_MT_TRACKING_ID为-1，也只有这里该值才可以小于零，收到该值，系统就会清除对应的ID。看似简单的两个协议内容到这里就分析完毕了。

&emsp;&emsp;看了上面的分析，明显可以看出B协议要优于A协议，但事实上并不如此简单。B协议需要硬件上的支持，ID值并不是随便赋值的，而是硬件上跟踪了点的轨迹；如果硬件上满足不了这个条件，那么采用B协议只能闹成笑话。另外，B协议的复杂性如果掌握不好往往会带来一些莫名其妙的问题，比如如果因为某些因素（同步等），在UP的时候少清除了一个slot的信息，那么下次单击的时候你也会惊奇地发现竟然有两个点（采用了B协议，slot已经保存了点信息，除非明确清除）。

可参考
[1] 内核代码Documentation/input/multi-touch-protocol.txt
[2] http://blog.chinaunix.net/uid-29151914-id-3921536.html