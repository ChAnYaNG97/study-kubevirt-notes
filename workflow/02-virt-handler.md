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

注：此处想找一个 virt-handler 的 YAML 文件过来看一下都传了哪些参数，但是 kube-virt 的默认安装方式是通过 virt-operator 启动所有组件，这个目前先搁置一下，主要是好奇几个关键参数，如 Pod IP 是以什么形式传入的（推测是环境变量获取 Pod.Status.PodIP, 然后环境变量作为命令行的参数）。


然后调用 `app.Run()` 启动 virt-handler。判断有没有 app.HostOverride、app.PodAddress 等。然后将 Node 标记为不可调度状态（？我不理解，也没有看到什么时候解除不可调度状态的）。

Virt-handler主体也是一个Controller，主要是对VMI的时间进行监听并进行相应的处理，主要逻辑在 `execute()` 函数中，函数开头主要是对一些特殊情况进行判断，如检查 VMI 是否存在和 UID 是否为空等和处理，如是否有过期的 Domain 需要清理，Migration相关操作等。

然后调用 `defaultExecute()` 进行主要的 sync 流程。首先根据函数参数传入的 vmiExist、domainExist、VMI 的状态、Domain的状态、进行加工得到 domainAlive 状态，进一步判断 Domain 需要进行的操作状态（shutdown、delete、cleanup、update、ignoreSync，其中 cleanup 指对已经从 libvirt 中 deleted 的 Domain 进行处理的过程）。最后根据操作状态进行相应的操作，这里主要关注创建虚拟机的部分，即 `processVmUpdate()`。

`processVmUpdate()` 函数主要是获取对应的 Pod （virt-lancher）的 grpc server 的 client，每一个 virt-handler 都会维护一个 key 为 VMI.UID， value 为 LauncherClientInfo 的 syncMap。

```go
type LauncherClientInfo struct {
    // grpc 调用的接口
	Client              cmdclient.LauncherClient

    // grpc 使用的 socket file
	SocketFile          string

	DomainPipeStopChan  chan struct{}

    // 记录最近一次判断是否 initialized 的时间
	NotInitializedSince time.Time

    // 是否处于就绪状态
	Ready               bool
}
```

根据 VMI 的UID找到对应的 client，底层采用的是基于文件socket的grpc通信。

最终 `vmUpdateHelperDefault()` 调用 `client.SyncVirtualMachine(vmi, options)` 函数，向 grpc server 发送 `SyncVMI` command，交由 virt-launcher 进行后续处理。
