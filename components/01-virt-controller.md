# Virt-controller

Virt-controller 是一个典型的 CRD controller，主要用于调谐 VM 和 VMI CRD 的期望状态与实际状态。当用户提交 CR 到集群中时，创建相应的 Pod。


入口函数在cmd/virt-controller.go，一个非常简单直接的`watch.Execute()`
```go
func main() {
	watch.Execute()
}

```

该函数就是一个 VirtualControllerApp 的初始化和启动过程，对于 VirtControllerApp 类型，里面无非就是各种资源的informerFactory，Controller的指针，和一些需要的参数，这里由于篇幅问题不粘贴源代码，具体见源码。

Controller 真正的启动过程，位于选举成功之后的回调函数 `OnStartedLeading` 里，对于这里为什么需要选举，目前还不是特别理解，可能是 controller 会有多个协程，选举成功的协程才能作为最终运行的 controller？这个地方不太明白。


之后就是一个 controller 的经典流程

* VMController

1. NewVMController时，监听 VM、VMI 和 dataVolume 三种对象的 event，自定义 EventHandlers（AddFunc、DeleteFunc、UpdateFunc）。
   
   - VM 的 event 直接加入到队列中
   - VMI 的 event，首先判断 VMI 是否有 VM 作为其 ControllerRef，如果有，则把对应的 VM 加入到队列中，否则就是孤儿 VMI，遍历所有匹配（和 VMI 的名字相同）的 VM，将它们入队。
   - dataVolume 的 event，首先也是判断是否有 VM 作为其 ControllerRef，只有在有的时候，才将 VM 入队。
2. controller 依次从队列中取出 VM 的 event 进行对应的处理，整个调用流程是 Run() -> runWorker() -> Execute() -> execute()。
   - 从本地缓存中取到对应的 VM 对象
   - 先进行收养流程，判断是否有对应的 VMI 和 dataVolume 可以收养，如果存在则进行收养
   - 然后进行 sync 流程，