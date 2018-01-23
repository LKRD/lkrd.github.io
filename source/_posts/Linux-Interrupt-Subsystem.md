---
title: <center> Linux Interrupt Subsystem </center>
date: 2018-01-23 11:52:55
tags:
---

GIC - Generic Interrupt Controller 通用中断控制器
PE - Processing elements
LPIs - Locality-specific Peripheral Interrrupts
PPIs - Private Peripheral Interrrupts
SGIs - Software Generated Interrupts
SPIs - Shared Peripheral Interrupts
ITS - Interrupt Translation Service
IRI - Interrupt Routing Infrastructure

编写目的
Linux驱动工程师理解和编写外设驱动的同时一个最基本的要求就是能够使用好中断处理，但往往只是会调用某些常用的中断API接口还远远不够，我们需要深入理解中断子系统的方方面面，才能在linux不断变化的过程中能够及时学会和使用中断子系统。

一  中断子系统整体框架
下面将会从硬件和软件这两方面入手，讲解中断子系统整体框架。

中断子系统硬件描述
中断子系统硬件上主要由三个角色构成，外设、中断控制器和处理器，外设需要向CPU发起中断时可通过中断信号线直接向CPU申请中断，由于现在的电子设备外设越来越多，需要进行中断的需求也越来越多，并且中断对于系统来说也要有优先级之分，在多CPU情况下有些中断需要多个CPU共同响应，有些只针对某个CPU申请中断，针对以上需求所以才有了中断控制器，ARM里面简称GIC，我们这里只是在ARM+GIC上做解释，在X86平台上又是不同的，X86使用APIC的方式管理中断控制器。

外设中断：对于外设来说，只需要通过控制GPIO口进行电平变化即可
中断控制器GIC：
<center>![](中断控制器GIC.png) </center>
GIC是ARM公司提供的通用中断控制器，目前（2017）在ARM提供四个architecture specification，即V1~V4，其中V2最多支持8个ARM core，而V3/V4支持更多，主要用在ARM64平台上。由于现在很多平台都是在ARM64上开发，所以我们这里讲解最新的V3/V4的SPEC，以上图片就是V3/V4上对于GIC整体硬件框架的描述图。
GIC的组成为一个Distributor，每个PE提供一个Redistributor和一个CPU interface，还有可选的特殊中断处理模块。
PE为处理器单元，几个PE可组成一个CPU Cluster，例如MTK P30（MT6758）用的Cortex-A53采用的是8核处理器，内部就是由两个Clusters组成，每个Cluster有4个CPUs；
对于SPIs的中断，通过Distributor，再经过Redistributor分发给各个PE，而PPIs可通过Redistributor分发给特定的PE

中断子系统软件描述
中断子系统软件部分由四部分组成：驱动层代码（driver目录下调用中断相关接口的模块），linux内核通用中断处理模块（kernel/irq目录下），中断控制器驱动代码（driver/irqchip目录下），cpu架构相关中断处理