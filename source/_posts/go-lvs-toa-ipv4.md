---
title: LVS后端GO-Web获取用户IP
date: 2019-07-10 16:12:31
categories: 后端开发
tags: [go, lvs, tcp6]
---
## 问题
最近写了个对外的ti接口,测试时发现获取到的用户ip都是lvs的ip. 咨询hulk同事LVS是FULLNAT模式,宿主机器需要安装toa.然而安装完toa后依旧没有解决问题.z在宿主机器上查看go-web监听和链接情况:

```
netstat -pant | grep ti
```
发现go gin框架默认以tcp6监听服务，通过查询资料得知toa仅支持tcp4, 那么就需要修改下源代码让服务监听tcp4.

## 查看gin源码
想看看gin是如何进行tcp监听的，查看gin.go代码289行发现，gin框架是使用go原生http.ListenAndServe启动的web服务.
```go
func (engine *Engine) Run(addr ...string) (err error) {
	defer func() { debugPrintError(err) }()

	address := resolveAddress(addr)
	debugPrint("Listening and serving HTTP on %s\n", address)
	err = http.ListenAndServe(address, engine)
	return
}
```
通过查看ListenAndServe代码得知，go的net/http就是以默认tcp6启动的http服务.

```go
func (srv *Server) ListenAndServe() error {
	if srv.shuttingDown() {
		return ErrServerClosed
	}
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```
因此想通过gin的某些魔法设置从而以tcp4监听的想法是不可能了，并且gin的issue中也说明了这一点:
> https://github.com/gin-gonic/gin/issues/667

## 手动监听
通过手动启动tcp监听，然后配置handler来实现，tcp4启动的web服务.

```go
type Tcp4Router struct {
	Handler http.Handler
	listener net.Listener
	server *http.Server
}

func (t *Tcp4Router)Run(addr string) (err error) {

	t.server = &http.Server{Addr:addr, Handler:t.Handler}

	t.listener, err = net.Listen("tcp4", addr)
	if err != nil {
		return
	}

	err = t.server.Serve(t.listener)
	return
}
```
## 最后
  生命在于折腾，在于学习...

## 致谢学习
[https://blog.csdn.net/achejq/article/details/73733920](https://blog.csdn.net/achejq/article/details/73733920)
[https://cloud.tencent.com/document/product/1022/31524](https://cloud.tencent.com/document/product/1022/31524)
