# Virt-launcher

推测应该就是一个 转换层，grpc server 将 grpc 请求转换成 libvirt 的调用，对 host 上的虚拟机进行操作。

## 入口函数

```go
// cmd/virt-launcher.go

```


首先也是命令行参数处理的部分，细节部分没有过多的关注


// DomainManager




## Questions


#### 每一个 virt-launcher 都会启动一个 libvirtd ？
在 virt-launcher 的 main 函数中，有 `l.StartLibvirt(stopChan)`的调用，其中使用命令行启动了 libvirtd。


是每一个 node 一个 virt-launcher 还是 每一个 VM 一个 virt-launcher？个人理解是后者，但是每一个 virt-launcher 都会启动一个 libvirtd 有点反直觉。