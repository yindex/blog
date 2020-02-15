---
layout: post
title: NSQ代码阅读与分析之Channel
date: 2019-04-15
categories: 中间件
tags: [golang, nsq, 消息队列, 中间件, Channel]
abstract:  此channel非彼channel,只是跟golang的channel同名而已,它是NSQ消费者订阅特定Topic的一种抽象.nsqd抽到生产者push的message后,会遍历当前topic下所有消费者channel并深度拷贝message下发到所有消费这channel中.并且同意个消费通道只会被投递一次.其中nsqd的deferred message在此实现.
---
<!-- toc -->

此channel非彼channel,只是跟golang的channel同名而已,它是NSQ消费者订阅特定Topic的一种抽象.nsqd抽到生产者push的message后,会遍历当前topic下所有消费者channel并深度拷贝message下发到所有消费这channel中.并且同意个消费通道只会被投递一次.其中nsqd的deferred message在此实现.下图是nsq官网的一个消息流向图.

![nsq.gif](/img/nsq.gif)

## channel创建于初始化

```go
func NewChannel(topicName string, channelName string, ctx *context,
	deleteCallback func(*Channel)) *Channel {

	c := &Channel{
		topicName:      topicName,
		name:           channelName,
		memoryMsgChan:  make(chan *Message, ctx.nsqd.getOpts().MemQueueSize),
		clients:        make(map[int64]Consumer),
		deleteCallback: deleteCallback,
		ctx:            ctx,
	}
	if len(ctx.nsqd.getOpts().E2EProcessingLatencyPercentiles) > 0 {
		c.e2eProcessingLatencyStream = quantile.New(
			ctx.nsqd.getOpts().E2EProcessingLatencyWindowTime,
			ctx.nsqd.getOpts().E2EProcessingLatencyPercentiles,
		)
	}

	c.initPQ()

	if strings.HasSuffix(channelName, "#ephemeral") {
		c.ephemeral = true
		c.backend = newDummyBackendQueue()
	} else {
		dqLogf := func(level diskqueue.LogLevel, f string, args ...interface{}) {
			opts := ctx.nsqd.getOpts()
			lg.Logf(opts.Logger, opts.LogLevel, lg.LogLevel(level), f, args...)
		}
		// backend names, for uniqueness, automatically include the topic...
		backendName := getBackendName(topicName, channelName)
		c.backend = diskqueue.New(
			backendName,
			ctx.nsqd.getOpts().DataPath,
			ctx.nsqd.getOpts().MaxBytesPerFile,
			int32(minValidMsgLength),
			int32(ctx.nsqd.getOpts().MaxMsgSize)+minValidMsgLength,
			ctx.nsqd.getOpts().SyncEvery,
			ctx.nsqd.getOpts().SyncTimeout,
			dqLogf,
		)
	}

	c.ctx.nsqd.Notify(c)

	return c
}
```

`NewChannel`工厂方法首先根据topicName, channelName创建了一个`Channel`类型类型实例,并初始化memoryMsgchan(内存channel)和一个消费者map.然后调用initPQ方法初始化其他参数.

若当前channel为临时channel,则创建一个伪后端落盘队列.否则通过`diskqueue.New`方法创建一个后端落盘队列.

最后调用`c.ctx.nsqd.Notify(c)`将该channel通知给当前集群的nsqlookupd.

###channel退出与释放

有创建channel就有对应的删除channel方法.源码`nsqd/channel.go:152`处可以看到该方法被完全锁保护,因此channel退出方法不能并发执行.

channel是消费者订阅通道,因此channel退出前需要关闭对应的消费者订阅连接.`nsqd/channel.go:169`位置通过读写锁保护,因此channel在退出时不允许新的消费着在创建连接.

最后将该channel中剩余的内容写入到磁盘.

### 写入消息PutMessage

PutMessage通过调用`put`方法写入到内存或者磁盘中.其中put方法实现如下:

```go
func (c *Channel) put(m *Message) error {
	select {
	case c.memoryMsgChan <- m:
	default:
		b := bufferPoolGet()
		err := writeMessageToBackend(b, m, c.backend)
		bufferPoolPut(b)
		c.ctx.nsqd.SetHealth(err)
		if err != nil {
			c.ctx.nsqd.logf(LOG_ERROR, "CHANNEL(%s): failed to write message to backend - %s",
				c.name, err)
			return err
		}
	}
	return nil
}
```

如果channel的内存通道未满则将消息写入到内存通道中(Channel.memoryMsgChan)否则执行默认分支写入到后端磁盘队列(如果是临时channel这部分操作将是无效操作). 若写磁盘出错会调用setHealth设置当前节点健康状态.

### 读/写延迟消息PutMessageDeferred

`PutMessageDeferred`通过调用`StartDeferredTimeout(msg, timeout)`实现.其中timeout为该消息到期时间. `StartDeferredTimeout`方法实现如下:

```go
func (c *Channel) StartDeferredTimeout(msg *Message, timeout time.Duration) error {
	absTs := time.Now().Add(timeout).UnixNano()
	item := &pqueue.Item{Value: msg, Priority: absTs}
	err := c.pushDeferredMessage(item)
	if err != nil {
		return err
	}
	c.addToDeferredPQ(item)
	return nil
}
```

首先计算出到期时间的时间戳,并将该事件戳设置到`Item`类型`nsqd/channel.go:441`的Priority字段.通过`addToDeferredPQ`方法`nsqd/channel.go:446` 将包含消延迟消息的Item添加到当前channel的deferredPQ字段. deferredPQ是一个pqueue.PriorityQueue类型,该类型是一个小根堆优先队列,也就是Priority值越小优先级越高所以时间戳越小越靠前.

**processDeferredQueue**方法是处理延迟消息的消息循环,它的代码如下:

```go
func (c *Channel) processDeferredQueue(t int64) bool {
	c.exitMutex.RLock()
	defer c.exitMutex.RUnlock()

	if c.Exiting() {
		return false
	}

	dirty := false
	for {
		c.deferredMutex.Lock()
		item, _ := c.deferredPQ.PeekAndShift(t)
		c.deferredMutex.Unlock()

		if item == nil {
			goto exit
		}
		dirty = true

		msg := item.Value.(*Message)
		_, err := c.popDeferredMessage(msg.ID)
		if err != nil {
			goto exit
		}
		c.put(msg)
	}

exit:
	return dirty
}
```

循环体中,通过调用`c.deferredPQ.PeekAndShift(t)`扇出一条符合条件的消息.其中t为当前事件戳.那么怎么才算符合条件呢?查看`PeekAndShift`代码知道只有消息的优先级(事件戳)大于当前时间戳才算符合.

最后将扇出的消息投递回普通的消费者订阅通道(这块比较棒,代码复用不在重复发送逻辑).

```go
func (pq *PriorityQueue) PeekAndShift(max int64) (*Item, int64) {
	if pq.Len() == 0 {
		return nil, 0
	}

	item := (*pq)[0]
	if item.Priority > max {
		return nil, item.Priority - max
	}
	heap.Remove(pq, 0)

	return item, 0
}
```



> 为什么不是大于等于呢?



###最后

channel和topic的实现类似,并且给了我一种channel和topic可以再抽象一层做代码复用的感觉.

两者都是内存通道+后端落盘.都是磁盘消息与内存消息竞争执行处理.

# Ref

- [https://nsq.io/clients/tcp_protocol_spec.html](https://nsq.io/clients/tcp_protocol_spec.html)
- [https://github.com/nsqio/nsq](https://github.com/nsqio/nsq)
- [https://blog.csdn.net/skh2015java/article/details/83419493](https://blog.csdn.net/skh2015java/article/details/83419493 )

