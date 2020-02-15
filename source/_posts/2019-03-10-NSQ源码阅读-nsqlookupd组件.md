---
layout: post
title: NSQ源码阅读-nsqlookupd组件
date: 2019-03-10
categories: 中间件
tags: [golang, nsq, nsqlookup, 消息队列, http, 装饰器模式]
abstract: NSQ包含三大组件nsqd、nsqlookup、nsqadmin. nsqlookupd 是守护进程负责管理拓扑信息,客户端通过查询 nsqlookupd 来发现指定topic的生产者，并且 nsqd 节点广播topic和channel信息.它包含两个接口：TCP 接口:处理nsqd广播。HTTP 接口:客户端用它来发现和管理服务。
---
<!-- toc -->
> NSQ包含三大组件nsqd、nsqlookup、nsqadmin. nsqlookupd 是守护进程负责管理拓扑信息,客户端通过查询 nsqlookupd 来发现指定topic的生产者，并且 nsqd 节点广播topic和channel信息.它包含两个接口：TCP 接口:处理nsqd广播。HTTP 接口:客户端用它来发现和管理服务。

## 一、NSQLookupd类型

nsqlookupd.go为nsqlookup组件的入口文件,它主要包含一个NSQLookupd结构类型和该类型的工厂方法func New(\*Options)\*NSQLookupd.

NSQLookupd类型属性定义如下.

```go
type NSQLookupd struct {
	sync.RWMutex                      
	opts         *Options             
	tcpListener  net.Listener          
	httpListener net.Listener          
	waitGroup    util.WaitGroupWrapper 
	DB           *RegistrationDB
}
```

- sync.RWMutex 读写锁，主要用来更新DB中的Registration，Topic, Channels.
- opts NSQLookup相关配置.
- tcpListener 用于nsqd与nsqlookupd通信，默认监听4160端口.
- httpListener 用于nsqlookupd与nsqadmin通信默认监听4161端口.
- waitGroup 装饰器模式，wrap了一层sync.WaitGroup，用于系统退出时同步tcpListener、httpListener结束.
- DB 存储当前注册的Registration， Topic， Chanel，用于nsqd的数据管理.

NSQLookupd类型绑定的方法列表如下.

```go
- func (n *NSQLookupd) logf(level lg.LogLevel, f string, args ...interface{})
- func Main() error
- func RealTCPAddr() *net.TCPAddr
- func RealHTTPAddr() *net.TCPAddr
- func Exit()
```

- [logf](https://github.com/zixks/nsq-analysis/blob/master/nsqlookupd/logger.go)方法 内部调用了internal日志方法,用于NSQLookupd上下文写日志.
- Main方法主要负责启动、停止,维护tcp http server等操作.
- RealTCPAddr, RealHTTPAddr获取当前服务地址.
- Exit() 同步关闭tcp http服务.

## 二、opts  成员

opts变量主要存储NSQLookup相关配置信息，类型定义如下:

```go
type Options struct {
	LogLevel  string `flag:"log-level"`
	LogPrefix string `flag:"log-prefix"`
	Verbose   bool   `flag:"verbose"`
	Logger    Logger
	logLevel  lg.LogLevel
	TCPAddress       string `flag:"tcp-address"`
	HTTPAddress      string `flag:"http-address"`
	BroadcastAddress string `flag:"broadcast-address"`
	InactiveProducerTimeout time.Duration `flag:"inactive-producer-timeout"` 
	TombstoneLifetime       time.Duration `flag:"tombstone-lifetime"`        
}
```

- LogLevel, LogPrefix定义了日志级别、日志前缀，Logger定义具体的日志类型，TCPAddress，HTTPAddress配置http，tcp相关服务监听地址.BroadcastAddress定义了广播地址.
- InactiveProducerTimeout 定义producer失活时间阈值，当超过InactiveProducerTimeout没有收到producer心跳时，则认为当前producer不在活跃.
- [TombstoneLifetime](https://github.com/mickey0524/nsq-analysis/blob/master/nsqlookupd/options.go#L23)避免发生竞争，当一个nsqd不再产生一个特定的toipc, 需要去掉这个toipc，这个时候，试图尝试删除topic信息与新的消费者已经发现这个主题的节点，重连, 会更新nsqlookup产生竞争.

## 三、waitGroup 成员

waitGroup成员是[util.WaitGroupWrapper](https://github.com/zixks/nsq-analysis/blob/master/internal/util/wait_group_wrapper.go)类型,内部组合了sync.WaitGroup类型并绑定Wrap装饰方.用以系统退出时同步关闭tcpServer和httpServer服务进行安全退出.

在nslookupd.go的入口方法中，装箱tcpserver和httpserver服务<sup>[nsqlookupd/nsqlookupd.go:64](https://github.com/zixks/nsq-analysis/blob/master/nsqlookupd/nsqlookupd.go#L86)</sup>, 在nslookupd退出时调用Exit()方法<sup>[nsqlookupd/nsqlookupd.go:87](https://github.com/zixks/nsq-analysis/blob/master/nsqlookupd/nsqlookupd.go#L86)</sup>, 它会等待所有被wrap的方法都执行结束后（调用了wg.done后）进行安全退出.

## 四、httpListener成员

httpListener成员类型定义在[http.go](https://github.com/zixks/nsq-analysis/blob/master/nsqlookupd/http.go)中， 该文件定义httpServer的同时定义了node变量和一个httpServer的工厂方法.httpServer类型包含一个Context上下文类型和一个路由请求处理handler(nsq使用了julienschmidt/httprouter路由，号称最快路由).

httpServer提供的api接口如下列表，同时它在注册路由时也使用了装饰器http_api.Decorate,装饰了log，http_api等. 同时httpServer提供了大量的DEBUG接口用于系统调试.

```go
router.Handle("GET", "/ping", http_api.Decorate(s.pingHandler, log, http_api.PlainText))
router.Handle("GET", "/info", http_api.Decorate(s.doInfo, log, http_api.V1))
router.Handle("GET", "/debug", http_api.Decorate(s.doDebug, log, http_api.V1))
router.Handle("GET", "/lookup", http_api.Decorate(s.doLookup, log, http_api.V1))
router.Handle("GET", "/topics", http_api.Decorate(s.doTopics, log, http_api.V1))
router.Handle("GET", "/channels", http_api.Decorate(s.doChannels, log, http_api.V1))
router.Handle("GET", "/nodes", http_api.Decorate(s.doNodes, log, http_api.V1))
router.Handle("POST", "/topic/create", http_api.Decorate(s.doCreateTopic, log, http_api.V1))
router.Handle("POST", "/topic/delete", http_api.Decorate(s.doDeleteTopic, log, http_api.V1))
router.Handle("POST", "/channel/create", http_api.Decorate(s.doCreateChannel, log, http_api.V1))
router.Handle("POST", "/channel/delete", http_api.Decorate(s.doDeleteChannel, log, http_api.V1))
router.Handle("POST", "/topic/tombstone", http_api.Decorate(s.doTombstoneTopicProducer, log, http_api.V1))
router.HandlerFunc("GET", "/debug/pprof", pprof.Index)
router.HandlerFunc("GET", "/debug/pprof/cmdline", pprof.Cmdline)
router.HandlerFunc("GET", "/debug/pprof/symbol", pprof.Symbol)
router.HandlerFunc("POST", "/debug/pprof/symbol", pprof.Symbol)
router.HandlerFunc("GET", "/debug/pprof/profile", pprof.Profile)
router.Handler("GET", "/debug/pprof/heap", pprof.Handler("heap"))
router.Handler("GET", "/debug/pprof/goroutine", pprof.Handler("goroutine"))
router.Handler("GET", "/debug/pprof/block", pprof.Handler("block"))
router.Handler("GET", "/debug/pprof/threadcreate", pprof.Handler("threadcreate"))
```

API接口比较多，功能也都顾名思义比较直接.这儿重点记录几个典型接口实现.

### func(s *httpServer) doCreateTopic

创建topic接口; 内部实现主要包含了topic参数获取，topic检测，日志记录以及最重要的topic检测. Topic合法检测主要检测了起长度限(internal/protocol/names.go:22)制和一个正则检验(internal/protocol/names.go:8). 

Topic创建通过nsqlookupd的db成员中的AddRegistration方法实现,AddRegistration在实现注册时使用读写锁进行并发资源保护. 代码实现如下:

```go
func (r *RegistrationDB) AddRegistration(k Registration) {
	r.Lock()
	defer r.Unlock()
	_, ok := r.registrationMap[k]
	if !ok {
		r.registrationMap[k] = make(map[string]*Producer)
	}
}
```

### func (s *httpServer) doLookup

查询指定Topic的nsqd节点接口(nsqlookupd/http.go:105)，函数内部主要调用了DB变量绑定的方法，从而实现查询功能.

httpServer提供的api接口方法主要在做与用户交互的一些功能，关于数据的管理都有后端db来提供实现.这也体现了mvc的思路吧算是.

## 五、tcpListener 成员

tcpServer类型定义在tcp.go(nsqlookupd/tcp.go:10)中，函数内部包含一个NSQLookupd的Context变量，以及一个处理请求的方法``` func Handle(net.Conn)```.

Handle方法主要完成两个操作.

### 客户端版本号识别

调用io.ReadFull读取客户端请求中的版本号，根据该函数描述`ReadFull reads exactly len(buf) bytes from r into buf.` ，精确读取客户端请求前4字节数据作为客户端版本号. [当前代码](https://github.com/zixks/nsq-analysis)版本仅支持版本号为V1的客户端, 若客户端合法则创建LookupProtocolV1类型并由该类型负责处理消息循环.其他版本则返回`E_BAD_PROTOCOL`

### [IOLoop消息循环](nsqlookupd/lookup_protocol_v1.go:20)

服务端首先对客户端请求进行预处理，以换行符号‘\\n’ 作为请求间的切分标志，并且针对每次请求以空格“ ”分割请求内容作为参数列表. 将请求参数列表和链接对象传给LookupProtocolV1.exec()处理. 根据客户端(nsqd)请求内容的第一个参数，确定请求功能类型.

IOLoop通过Exec支持四种操作: PING, IDENTIFY, REGISTER UNREGISTER.

- PING 心跳包，nsqd存活检测, 并更新nsqd lastUpdate(原子操作)

- IDENTIFY 主要负责nsqd 连接tcpServer时候发送的注册包，根据nsqd发送的body内容实例化一个PeerInfo用来存储nsqd的meta信息，并讲PeerInfo存储到producer map中.

  ```go
  peerInfo := PeerInfo{id: client.RemoteAddr().String()}
  err = json.Unmarshal(body, &peerInfo)
  if err != nil {
      return nil, protocol.NewFatalClientErr(err, "E_BAD_BODY", "IDENTIFY failed to decode JSON body")
  }
  peerInfo.RemoteAddress = client.RemoteAddr().String()
  ...
  atomic.StoreInt64(&peerInfo.lastUpdate, time.Now().UnixNano())
  
  ```

- REGISTER 获取topic channel构造Registration，存储到DB中

  ```go
  if p.ctx.nsqlookupd.DB.AddProducer(key, &Producer{peerInfo: client.peerInfo}){
  	p.ctx.nsqlookupd.logf(LOG_INFO, "DB: client(%s) REGISTER category:%s key:%s subkey:%s",client, "channel", topic, channel)
  }
  
  if p.ctx.nsqlookupd.DB.AddProducer(key, &Producer{peerInfo: client.peerInfo}){
  	p.ctx.nsqlookupd.logf(LOG_INFO, "DB: client(%s) REGISTER category:%s key:%s subkey:%s",client, "topic", topic, "")
  }
  		
  ```

- UNREGISTER 顾名思义为REGISTER逆操作.

## 六、DB成员
DB成员类型定义在nsqlookupd/registration_db.go，该文件中定义以下7中类型，该文件内实现了nsqlookupd的核心业务模型逻辑，tcpServer和httpServer中对外提供的接口都是由该部分提供具体实现.应该算是模型层吧.
该部分提供底层操作逻辑供tcpserver&httpserver调用，因此阅读类型结构后根据tcpserver&httpserver作为线索理清楚DB绑定方法.

### DB成员类型依赖关系图
<img src="/img/nslookupd_type.png"/>

### DB成员提供的服务

#### 创建Topic

在RegistrationDB中添加一个Registration key和默认的ProducerMap 并返回false.若已经存在则返回true

#### 删除Topic

先根据当前topic检索出该topic上绑定的channels，并删除所有的相关channel（channel也存储在registrationMap中）然后删除该topic相关信息(Producer， PeerInfo...)
ps: channel 和 topic都存储在registrationMap中，由key:Registration{Category, Key, SubKey}来区分存储的数据类别.

#### 屏蔽Topic

[nsqlookupd/http.go:doTombstoneTopicProducer](https://github.com/nsqio/nsq/blob/master/nsqlookupd/http.go#L178), 屏蔽制定node的一个topic. 在http请求中获取要屏蔽的node和topic,并根据topicName检索出相应的Producer，[根据请求中的node屏蔽Producer中对应的producer](https://github.com/nsqio/nsq/blob/master/nsqlookupd/http.go#L198). 

虽然go是值传递，for循环时是临时对象，但是根据```Producers```类型定义可以知道该类型中存储的是指针类型，所以再执行```p.Tombstone()```时是在具体地址上执行.

```go
type Producers []*Producer
```

#### 创建Channel

创建channel步骤与创建topic相同,都是调用AddRegistration，只是在创建channel后调用了一次创建该channel的topic（[不会重复创建](https://github.com/nsqio/nsq/blob/master/nsqlookupd/registration_db.go#L76)）

#### 删除Channel

Channel删除比较直接，根据请求中的Channel值，检索出相关Registration然后删除.

#### Lookup功能

根据传入的topic值，检索判断是否存在对应的Registration.检索出相应的producer&channels并返回.

#### nsqd连接nsqlookupd [[IDENTIFY]](https://github.com/nsqio/nsq/blob/master/nsqlookupd/lookup_protocol_v1.go#L194)

- 解析出请求中peerInfo
- 原子操作更新peerInfo.lastUpdate
- 注册到Producer

……




## 参考链接

- [[1]nsq源码阅读](https://github.com/mickey0524/nsq-analysis)
- [[2]go简单工厂模式](https://github.com/zixks/golang-design-pattern/tree/master/00_simple_factory)
- [[3]工厂方法:维基百科](https://zh.wikipedia.org/zh-hans/%E5%B7%A5%E5%8E%82%E6%96%B9%E6%B3%95#%E7%AE%80%E5%8D%95%E5%B7%A5%E5%8E%82)
- [[4]golang 装饰模式](https://github.com/zixks/golang-design-pattern/tree/master/20_decorator) 
- [[5]Decorator pattern](https://en.wikipedia.org/wiki/Decorator_pattern)    



