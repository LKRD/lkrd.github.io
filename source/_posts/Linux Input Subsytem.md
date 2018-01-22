---
title: <center> <font color=#000000 size=6 face="Calibri">Linux Input Subsystem</font> </center>
date: 2018-01-22
---

&emsp;&emsp;最基本的认知：一个设备驱动申请input设备并且注册，之后配置一些相关参数，在驱动加载成功后会在文件系统的/dev/input/目录下生成对应的eventX的事件设备结点，假如对于TP设备，我们触摸屏幕，在Android下使用getevent工具就可以获取到数据，应用通过这些数据就可以上报给系统发生了什么。

&emsp;&emsp;如果我们想要了解这整个过程的来龙去脉，就要提出这样几个问题：
&emsp;&emsp;**1. input子系统是如何将设备驱动和event绑定在一起的？**
&emsp;&emsp;**2. 设备驱动如何使用input子系统的接口与内核和应用交互？**

&emsp;&emsp;从设计input子系统的设计者出发，在了解整个input子系统时，我们可以先了解下其设计模式 -- observer设计模式

#### observer设计模式
&emsp;&emsp;我们知道使用面向对象的设计思维往往在设计软件模型的时候可以减少很多设计环节，将复杂的关系网络抽象化后达到高内聚和低耦合的作用。设计模式是从以往的软件系统总结出的成功、高可靠性、高复用性的可实行设计方案。在内核开发中不免也存在一些这样的设计模式，input子系统就是采用类似observer模式的设计模型进行设计的。我们先来了解下什么是observer设计模式。
**1. observer设计模式产生的背景**
&emsp;&emsp;一些面向对象的编程方式，提供了一种构建对象间复杂网络互连的能力。当对象们连接在一起时，它们就可以相互提供服务和信息。
&emsp;&emsp;通常来说，当某个对象的状态发生改变时，你仍然需要对象之间能互相通信。但是出于各种原因，你也许并不愿意因为代码环境的改变而对代码做大的修改。也许，你只想根据你的具体应用环境而改进通信代码。或者，你只想简单的重新构造通信代码来避免类和类之间的相互依赖与相互从属。
&emsp;&emsp;当一个对象的状态发生改变时，你如何通知其他对象？是否需要一个动态方案――一个就像允许脚本的执行一样，允许自由连接的方案？
**2. observer设计模式的解决思路**
&emsp;&emsp;observer模式：定义对象间的一种一对多的依赖关系,当一个对象的状态发生改变时, 所有依赖于它的对象都得到通知并被自动更新。
&emsp;&emsp;observer模式允许一个对象关注其他对象的状态，并且还为被观察者提供了一种观察结构，或者说是一个主体和一个客体。主体，也就是被观察者，可以用来联系所有的观察它的观察者。客体，也就是观察者，用来接受主体状态的改变 观察就是一个可被观察的类（也就是主体）与一个或多个观察它的类（也就是客体）的协作。不论什么时候，当被观察对象的状态变化时，所有注册过的观察者都会得到通知。
&emsp;&emsp;observer模式将被观察者（主体）从观察者（客体）分离出来。这样每个观察者都可以根据主体的变化分别采取各自的操作。对于被观察者来说，那些查询哪些类需要自己的状态信息和每次使用哪些状态信息的额外资源开销已经不存在了。另外，一个观察者可以在任何合适的时候进行注册和取消注册。你也可以定义多个具体的观察类，以便在实际应用中执行不同的操作。
&emsp;&emsp;将一个系统分割成一系列相互协作的类有一个常见的副作用：需要维护相关对象间的一致性。我们不希望为了维持一致性而使各类紧密耦合，因为这样降低了它们的可重用性。
**3. observer设计模式的组成和结构**
&emsp;&emsp;observer设计模式的组成：
&emsp;&emsp;目标（Subject）：目标知道它的观察者。可以有任意多个观察者观察同一个目标。提供注册和删除观察者对象的接口。
&emsp;&emsp;具体目标（ConcreteSubject）：将有关状态存入各ConcreteObserver对象。
&emsp;&emsp;观察者(Observer)：为那些在目标发生改变时需获得通知的对象定义一个更新接口。当它的状态发生改变时, 向它的各个观察者发出通知。
&emsp;&emsp;具体观察者(ConcreteObserver)：维护一个指向ConcreteSubject对象的引用。存储有关状态，这些状态应与目标的状态保持一致。实现Observer的更新接口以使自身状态与目标的状态保持一致。
&emsp;&emsp;其结构如下：
<center>![](observer design mode.jpg) </center>

&emsp;&emsp;详细可查看参考资料[1]

&emsp;&emsp;针对observer设计模式的框图，我们可以描绘出input子系统的大致框图如下：
<center>![](input整体关系图.jpg) </center>

#### input subsystem框架详细分析
&emsp;&emsp;此代码分析是基于linux 3.18内核代码
&emsp;&emsp;我们从三个模块进行分析input整个框架，包括input device driver，input core，input event handler，详细的模块之间的调用关系框图如下图所示：
<center>![](Input Subsystem.png)</center>

&emsp;&emsp;在上图可以看到input子系统中提供了三个很重要的结构体，包括**input_dev**，描述input设备，**input_handler**描述了事件处理的各种方法和**input_handle**描述了dev和handler的对应关系，这些在inpu.h中都有其描述。
&emsp;&emsp;作为驱动开发者，input device driver是必须要很熟悉的，这里是驱动开发者发挥的地方，通过调用input core中的接口申请device，注册device和上报数据。
&emsp;&emsp;驱动第一步申请input device，并初始化input_dev结构体的成员
```bash
tpd->dev = input_allocate_device(&pdev->dev);
```
&emsp;&emsp;注册input device
```bash
input_register_device(tpd->dev);
```
&emsp;&emsp;我们看下注册时主要做了什么事情？
```bash
static LIST_HEAD(input_handler_list);
int input_register_device(struct input_dev *dev)
{
	...
    
    list_add_tail(&dev->node, &input_dev_list);

	list_for_each_entry(handler, &input_handler_list, node)
		input_attach_handler(dev, handler);
        
    ...
    return 0;
}
```
&emsp;&emsp;首先会把设备结点加入到input_dev_list链表上，之后再查找input_handler_list链表。
&emsp;&emsp;但是input_handler_list是哪里来的？里面又是根据什么将设备驱动和handler list进行匹配的？
&emsp;&emsp;我们可以到input目录下，跟到event handler那里，这里有evdev，joydev，mousedev等模块，这几个事件处理模块初始化时都调用了
```bash
input_register_handler(&evdev_handler);

input_register_handler(&joydev_handler);

input_register_handler(&mousedev_handler);
```
&emsp;&emsp;这里出现了我们之前说的另一个结构体input_handler，我们看下input_register_handler做了什么
```bash
static LIST_HEAD(input_dev_list);
int input_register_handler(struct input_handler *handler)
{
	...

	list_add_tail(&handler->node, &input_handler_list);

	list_for_each_entry(dev, &input_dev_list, node)
		input_attach_handler(dev, handler);

	...
	return 0;
}
```
&emsp;&emsp;这里就可以看到每个event handler注册时都会调用list_add_tail将handler添加到input_handler_list链表中，回到device driver中，注册时如果handler list有成员，则调用input_attach_handler，我们可以看下input_attach_handler这个接口做了什么？
```bash
static int input_attach_handler(struct input_dev *dev, struct input_handler *handler)
{
	...
	id = input_match_device(handler, dev);
	...
	error = handler->connect(handler, dev, id);
    ...
    
	return error;
}
```
&emsp;&emsp;这个接口通过调用input_match_device找到匹配的一对，devcie对应handler，匹配成功就调用event handler的connect接口，那这里的事件处理有好几个，如何匹配？我们看下input_match_device这个接口
```bash
static const struct input_device_id *input_match_device(struct input_handler *handler,
							struct input_dev *dev)
{
	const struct input_device_id *id;

	for (id = handler->id_table; id->flags || id->driver_info; id++) {

		if (id->flags & INPUT_DEVICE_ID_MATCH_BUS)
			if (id->bustype != dev->id.bustype)
				continue;
		...

		if (!bitmap_subset(id->evbit, dev->evbit, EV_MAX))
			continue;

		...

		if (!handler->match || handler->match(handler, dev))
			return id;
	}

	return NULL;
}
```
&emsp;&emsp;在evdev模块中对handler的描述可以看到，id->driver_info为1，意味着将所有的设备都与evdev匹配，生成eventx，对于joydev和mousedev，会根据id成员包括总线类型(bustype)，厂家(vendor)，产品(product)，版本(version)等来决定是否匹配成功。
&emsp;&emsp;成功之后我们再来看下event handler的connect接口，以evdev为例
```bash
static int evdev_connect(struct input_handler *handler, struct input_dev *dev,
			 const struct input_device_id *id)
{
	...

	dev_set_name(&evdev->dev, "event%d", dev_no);

	evdev->handle.dev = input_get_device(dev);
	evdev->handle.name = dev_name(&evdev->dev);
	evdev->handle.handler = handler;
	evdev->handle.private = evdev;

	evdev->dev.devt = MKDEV(INPUT_MAJOR, minor);
	evdev->dev.class = &input_class;
	evdev->dev.parent = &dev->dev;
	evdev->dev.release = evdev_free;
	device_initialize(&evdev->dev);

	error = input_register_handle(&evdev->handle);
	...

	cdev_init(&evdev->cdev, &evdev_fops);
	evdev->cdev.kobj.parent = &evdev->dev.kobj;
	error = cdev_add(&evdev->cdev, evdev->dev.devt, 1);
	...

	error = device_add(&evdev->dev);
	...

	return 0;
	...
}
```
&emsp;&emsp;在上面代码中我们可以看到设置了设备结点名称为eventx，同时出现了第三个很重要的结构体input_handle，evdev中定义了handle成员，此成员负责将device和handler绑定在一起，handle获得device和handler之后进行注册，调用input_register_handle，同时初始化event handler的字符操作接口fops，我们再看下对handle是如何注册的。
```bash
int input_register_handle(struct input_handle *handle)
{
	struct input_handler *handler = handle->handler;
	struct input_dev *dev = handle->dev;
	...
	if (handler->filter)
		list_add_rcu(&handle->d_node, &dev->h_list);
	else
		list_add_tail_rcu(&handle->d_node, &dev->h_list);
	...
	list_add_tail_rcu(&handle->h_node, &handler->h_list);

	if (handler->start)
		handler->start(handle);

	return 0;
}
```
&emsp;&emsp;此接口调用了list_add_tail_rcu，将新的链表元素handle->d_node和handle->h_node添加到被CRU保护的链表dev->h_list和handler->h_list的末尾，内存栅保证了在引用这个新插入的链表元素之前，新链表的链接指针的修改对所有读执行单元是可见的。
&emsp;&emsp;input_dev，input_handler，input_handle这三者的链表关系如下：
<center>![](input结构体中链表的关系图.jpg)</center>

&emsp;&emsp;在上图中每个dev或handler匹配后都会对应一个handle，所以其实对于input_dev与input_handler是一个多对多的关系，一个dev可以对应多个handler，一个handler也可以对应多个dev。

&emsp;&emsp;至此，整个初始化算是完成了。接下来看下设备是如何将数据传递给event handler的？
&emsp;&emsp;input core定义了数据输入格式：
```bash
struct input_value {
	__u16 type;
	__u16 code;
	__s32 value;
};
```
&emsp;&emsp;包括事件类型，类型对应的代码，和对应的数值
&emsp;&emsp;如果驱动需要上报数据，通过调用input_report_xxx接口，包括上报键值，相对参数，绝对参数等，input_sync同步input数据，一般是每一组数据上报完成后需要调用的，input_mt_sync槽数据同步，这些接口都是调用了input_event，我们看下input_event接口
```bash
void input_event(struct input_dev *dev,
		 unsigned int type, unsigned int code, int value)
{
	...
	input_handle_event(dev, type, code, value);
	...
}

static void input_handle_event(struct input_dev *dev,
			       unsigned int type, unsigned int code, int value)
{
	...
	input_pass_values(dev, dev->vals, dev->num_vals);
	...

}
static void input_pass_values(struct input_dev *dev,
			      struct input_value *vals, unsigned int count)
{
	...
	handle = rcu_dereference(dev->grab);
	if (handle) {
		count = input_to_handler(handle, vals, count);
	} else {
		list_for_each_entry_rcu(handle, &dev->h_list, d_node)
			if (handle->open)
				count = input_to_handler(handle, vals, count);
	}
	...
}
static unsigned int input_to_handler(struct input_handle *handle,
			struct input_value *vals, unsigned int count)
{
	struct input_handler *handler = handle->handler;
	...

	if (handler->events)
		handler->events(handle, vals, count);
	else if (handler->event)
		for (v = vals; v != end; v++)
			handler->event(handle, v->type, v->code, v->value);

	return count;
}
```
&emsp;&emsp;input_event一直将dev传递进入调用，到input_paass_value调用了rcu_dereference获得handle，一般情况下应用没有操作grab是获取不到handle的，所以这里是通过dev->h_list获得handle，这里会判断event是否被open过，获取之后input_to_handler调用了handler->events的接口，看下handler的events接口，以evdev为例
```bash
static void evdev_events(struct input_handle *handle,
			 const struct input_value *vals, unsigned int count)
{
	...
	client = rcu_dereference(evdev->grab);

	if (client)
		evdev_pass_values(client, vals, count, time_mono, time_real);
	else
		list_for_each_entry_rcu(client, &evdev->client_list, node)
			evdev_pass_values(client, vals, count,
					  time_mono, time_real);
	...
}
```
&emsp;&emsp;看到这里，同样一般都是无法从rcu_dereference获取得到client，我们看下这个client是在什么时候加入client_list的？
```bash
static int evdev_open(struct inode *inode, struct file *file)
{
	struct evdev *evdev = container_of(inode->i_cdev, struct evdev, cdev);
    ...
	struct evdev_client *client;
	...
	client->evdev = evdev;
	evdev_attach_client(evdev, client);

	error = evdev_open_device(evdev);
	...
}
static void evdev_attach_client(struct evdev *evdev,
				struct evdev_client *client)
{
	spin_lock(&evdev->client_lock);
	list_add_tail_rcu(&client->node, &evdev->client_list);
	spin_unlock(&evdev->client_lock);
}
```
&emsp;&emsp;应用在操作event时首先需要open，陷入到内核后就调用evdev_open这个接口，申请client并且调用evdev_attach_client将client加入到client_list中。回到evdev_events，找到了client后调用evdev_pass_values，接着往下看。
```bash
static void evdev_pass_values(struct evdev_client *client,
			const struct input_value *vals, unsigned int count,
			ktime_t mono, ktime_t real)
{
	...
	for (v = vals; v != vals + count; v++) {
		event.type = v->type;
		event.code = v->code;
		event.value = v->value;
		__pass_event(client, &event);
		if (v->type == EV_SYN && v->code == SYN_REPORT)
			wakeup = true;
	}
	...
	if (wakeup)
		wake_up_interruptible(&evdev->wait);
}
static void __pass_event(struct evdev_client *client,
			 const struct input_event *event)
{
	client->buffer[client->head++] = *event;
	...
	if (event->type == EV_SYN && event->code == SYN_REPORT) {
		...
		kill_fasync(&client->fasync, SIGIO, POLL_IN);
	}
}
```
&emsp;&emsp;在上面的各个接口调用下来可以知道最终将event数据的数据类型为EV_SYN并且数据代码为SYN_REPORT时，通过异步通知kill_fasync的方式通知应用。

&emsp;&emsp;上面了解到不同的input device经过初始化抽象出event，handler提供同样的操作对event进行操作，作为被观察者device，其状态的改变都会经过handle通知给观察着handler，handler经过自己的方式（异步通知）通知上层。而有些input设备例如鼠标，其不仅匹配了event handler，还匹配了mouse devcie handler，所以鼠标设备的改变，将会同时通知event device handler和mouse device handler。

#### 总结
&emsp;&emsp;本文通过observer设计模式展开对input系统的理解，以evdev为例，讲到在初始化阶段利用了多对多的设计模型，input_handle是如何将input_dev和input_handler绑定在一起，device driver又是如何通过input_event上报数据的，到这里其实也只是看到input子系统的某一部分，在更多的使用input子系统并且调用其接口时才能够体会设计的妙处，例如里面用到的RCU机制，对于Android上如何获取input上报的数据和上报数据的格式后续再探讨。

参考资料
[1] 设计模式之观察者模式Observer http://blog.csdn.net/hguisu/article/details/7556625
[2] Linux Input子系统 http://blog.chinaunix.net/uid-29151914-id-3887032.html
[3] LINUX下input子系统分析 http://blog.csdn.net/tang_jin_chan/article/details/9771435
