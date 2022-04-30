# Virt-handler

Virt-handler 是一个 DaemonSet，运行在集群的每个节点上，用于将调度到该节点的 VMI 启动起来。



### 
Virt-handler也是一个控制器，主要是监听所有VM的事件，并进行



## cmd/virt-handler/virt-handler.go
```go
func main() {
	app := &virtHandlerApp{}
	service.Setup(app)
	log.InitializeLogging("virt-handler")
	app.Run()
}
```

首先通过 `service.Setup(app)` 对 virtHandler 进行相应的初始化操作，主要是根据用户提供的启动参数，对 app 的成员变量进行赋值操作，包括HostOverride、PodIpAddress 等。需要注意的是，app.BindAddress默认值为0.0.0.0，app.Port的默认值为8185（目前还不知道是什么的address:port）。

注：此处想找一个 virt-handler 的 YAML 文件过来看一下都传了哪些参数，但是 kube-virt 的默认安装方式是通过 virt-operator 启动所有组件，这个目前先搁置一下，主要是好奇 Pod IP 是以什么形式传入的。


然后调用 `app.Run()` 启动 virt-handler。
