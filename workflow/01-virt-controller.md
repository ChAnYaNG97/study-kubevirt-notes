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

选举成功之后，通过`onStartedLeading()`回调函数，启动各控制器的流程，本节主要分析`vmController`和`vmiController`两个控制器。


### VMController

1. NewVMController时，监听 VM、VMI 和 dataVolume 三种对象的 event，自定义 EventHandlers（AddFunc、DeleteFunc、UpdateFunc）。
   - VM 的 event 直接加入到队列中
   - VMI 的 event，首先判断 VMI 是否有 VM 作为其 ControllerRef，如果有，则把对应的 VM 加入到队列中，否则就是孤儿 VMI，遍历所有匹配（和 VMI 的名字相同）的 VM，将它们入队。
   - dataVolume 的 event，首先也是判断是否有 VM 作为其 ControllerRef，只有在有的时候，才将 VM 入队。

2. controller 依次从队列中取出 VM 的 event 进行对应的处理，整个调用流程是 `Run() -> runWorker() -> Execute() -> execute() -> sync()`。
   - 从本地缓存中取到对应的 VM 对象
   - 先进行收养流程，判断是否有对应的 VMI 可以收养，如果存在则进行收养
   - 获取 VM 的所有dataVolumes
   - 进行 sync 流程，sync 流程中最重要的函数是`startStop(vm, vmi)`，也就是虚拟机的启停操作。根据设置的 VM 的运行策略（RunStrategy），按照规则执行`startVMI()`和`stopVMI()`函数。在`startVMI()`函数中，设置好 VMI 对象的属性，然后通过 client 创建相应的 VMI。同理，在`stopVMI()`中，也是通过client 删除相应的VMI。
  
### VMIController

在VMI创建或者删除之后，VMIController则会收到相应的事件通知，执行对应的事件逻辑。

VMIController依次从队列中取出 VMI 的 event 进行对应的处理，整个调用流程也是 `Run() -> runWorker() -> Execute() -> execute() -> sync()`。

在`sync()`函数中，总体目标就是为 VMI的创建/删除 event 进行创建/删除对应的 Pod。

- 对于 VMI 的创建，在创建之前则需要进行一些检查，如相关联的dataVolume是否准备好。然后根据不同的情况，在templatePod上渲染出不同的Pod对象，进行创建。

- 对于 VMI 的删除，则直接删除掉所有相关联的 Pod 和附属 Pod（暂时还不知道包括哪些）。

### Scheduler
// 找到根据 VMI 对应的 Pod 的调度结果并更新 VMI 的代码
当 VMI 对应的 Pod 创建成功之后，Kubernetes 会对该 Pod 进行调度，并在对应的节点上运行起来。此时，会触发 Pod 的 Update event，并对该 Pod 所属的 VMI 入队。在 VMIController 的 execute 函数中，首先会获取 VMI 对应的 Pod，并通过 updateStatus 对 VMI 的状态进行更新，具体地，调用 UpdateCondition 按照对应 Pod 的具体状态为 VMI 设置相应的状态（VirtualMachineInstanceCondition）。

最后会通过比较 VMI 和函数的初始对 VMI 的拷贝 vmiCopy 进行比较，如果有更新的部分，则会调用 clientSet 对 VMI 进行更新，此时还是会触发一次 VMI 的 Update 事件，但是该事件应该会落到 `case vmi.IsScheduled()` 分支，不进行任何操作。

### Virt-handler






Virt-handler也是一个控制器，主要是监听所有VM的事件，并进行


### Virt-launcher








### 参考链接

https://kubernetes.io/blog/2018/05/22/getting-to-know-kubevirt/







