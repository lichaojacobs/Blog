---
title: kubernetes shared Informer 源码解析
date: 2020-09-27 15:24:33
tags:
	- 架构
	- kubernetes
    - 源码
---

[知乎专栏链接](https://zhuanlan.zhihu.com/p/255078405)

kubernetes 是一款设计优秀的开源分布式系统。其中贯穿始终的设计模式是结合etcd的list-watch 模式，通过它来解耦系统中各个组件间的数据交互流程。今天来深入解析一下shared informer的源码实现细节。

## 背景

- informer是list-watch实现中的核心模块，通过它可以注册资源变化的回调函数
- 一个服务里面可能会起多个controller去消费同一个Resource CRUD事件，如果每个controller都起一个informer，则很浪费资源。通过SharedInformer，不同controller往其中注册Listener就能实现类似观察者模式的处理方式了。

## Informer基本架构

![Informer架构](http://jacobs.wanhb.cn/images/shared-informer-1.jpg)

## SharedInformer基本组成

```text
type sharedIndexInformer struct {
	indexer    Indexer
	controller Controller
	processor  *sharedProcessor
	blockDeltas sync.Mutex
        listerWatcher ListerWatcher
        ...
}
```

- indexer: 本地缓存，底层的实现是threadSafeMap
- controller: 承上启下的事件控制器（通过List-watch机制从 API Server 拿到事件，下发给 Informer 进行处理）
- sharedeProcessor: 封装多个事件消费者的处理逻辑，client端通过AddEventHandler接口加入到事件消费Listener列表中
- blockDeltas：用来在informer运行期间锁住整个事件的分发处理，保证listener在运行期间能够安全的加入
- listerWatcher：具体要从apiServer拉取要消费对象的实现

### 如何往informer添加消费者

通过sharedIndexInformer.AddEventHandler方法client端将自己的消费逻辑注册到shared_informer中：

- 创建新的listener，将处理函数封装进去

- 如果informer 还没有启动，这时直接将listener加入processor中即可，等待后续随informer一同启动

- 如果informer已经启动，即在运行期加入handler

- - 则此时为了安全的将listener加入，需要先block事件的分发（通过blockDeltas）
  - 然后通过addListener加入到processor中，并启动run/pop方法开始消费
  - 通过indexer list方法获取本地已经处理的所有事件给新加入的listener重新消费一遍，来达到不漏消息的目的

```text
func (s *sharedIndexInformer) AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) {
        // 省略非关键代码...
	listener := newProcessListener(handler, resyncPeriod, determineResyncPeriod(resyncPeriod, s.resyncCheckPeriod), s.clock.Now(), initialBufferSize)

	if !s.started {
		s.processor.addListener(listener)
		return
	}
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	s.processor.addListener(listener)
	for _, item := range s.indexer.List() {
		listener.add(addNotification{newObj: item})
	}
}
```

### infomer 的工作流程

我们先来看以下细化的流程图

![sharedInformer细节](http://jacobs.wanhb.cn/images/shared-informer-2.jpg)

从顶层看，启动的controller通过Reflector利用list-watch接口从apiserver同步对象和变更事件

```text
func (c *controller) Run(stopCh <-chan struct{}) {
        ....
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
        // ... 省略非关键代码
	wg.StartWithChannel(stopCh, r.Run)

	wait.Until(c.processLoop, time.Second, stopCh)
}

func (r *Reflector) Run(stopCh <-chan struct{}) {
	wait.Until(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			utilruntime.HandleError(err)
		}
	}, r.period, stopCh)
}
```

reflector拿到变更事件后，将事件塞入reflector的store中，其实也是controller的Queue，对应到shared_informer中是构造的DeltaFIFO实现

![代码片段](http://jacobs.wanhb.cn/images/shared-informer-3.jpg)

![代码片段](http://jacobs.wanhb.cn/images/shared-informer-4.jpg)

![代码片段](http://jacobs.wanhb.cn/images/shared-informer-5.jpg)

进入deltaFIFO中的事件，统一由sharedInformer中定义的HandleDeltas方法来消费和处理，处理的逻辑主要是按照不同的类型，分发到processor中。同时在这里还会根据事件类型更新indexer中的本地缓存

```text
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()
	for _, d := range obj.(Deltas) {
		switch d.Type {
		case Sync, Added, Updated:
			isSync := d.Type == Sync
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, isSync)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```

通过processor的distribute方法，其实是将Notification对象通知到底层注册的listener list，由listener 定义的处理Handler来消费事件

```text
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	if sync {
		for _, listener := range p.syncingListeners {
			listener.add(obj)
		}
	} else {
		for _, listener := range p.listeners {
			listener.add(obj)
		}
	}
}
```

通过listener add方法，实际上是把事件发送到了addChannel中

- 由于listener在sharedInformer启动时就已经启动了pop GoRoutine，它会轮询的从addChannel中消费事件，根据一定策略加入到nextChannel中
- run GoRoutine 负责消费nexChannel中的事件，根据类型并派发到listener定义的handler中去做实际的处理
- 也就是说Handler必须实现以下几个方法

```text
type ResourceEventHandler interface {
	OnAdd(obj interface{})
	OnUpdate(oldObj, newObj interface{})
	OnDelete(obj interface{})
}
```

