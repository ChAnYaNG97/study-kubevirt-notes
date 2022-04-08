# Study Libvirt APIs （libvirt扫盲）

主要是通过阅读官方的[Libvirt Application Development Guide Using Python](https://libvirt.org/docs/libvirt-appdev-guide-python/en-US/html/libvirt_application_development_guide_using_python-Introduction.html)学习，尝试使用go API复现主要内容。


### 关键定义

|术语|定义|
|---|---|
|Domain|运行在一个虚拟机上的操作系统实例|
|Hypervisor|用于在节点上支持虚拟化的软件层|
|Node|一个物理服务器|
|Storage Pool|存储介质池|
|Volume|从Storage Pool中分配的存储空间，一个Volume可以被分配到一个或多个Domain，一般在Domain作为虚拟硬盘|


### 对象模型

#### Hypervisor connections

最高层/最重要的对象，一般在程序的一开始都会创建一个connection实例，用于调用API。
一个connection是与一个具体的Hypervisior建立的，


#### Guest domain
代表一个运行的虚拟机或者能够启动一个虚拟机的配置。
唯一标识：
* ID: 正整数，每一个host上的domain的ID都是唯一的
* name：字符串，每一个host上的domain的name都是唯一的
* UUID：全局唯一UUID

#### Virtual networks

Virtual networks提供guest domain的网络设备的连接方法
唯一标识：
* name：字符串，每一个host上的domain的name都是唯一的
* UUID：全局唯一UUID

#### Storage pools

用于管理host上的各种类型的存储，本地盘、逻辑卷组等。
唯一标识：
* name：字符串
* UUID：全局唯一UUID

#### Storage volumes
指分配的一块存储空间，磁盘分区、逻辑卷等。一个volume可以被多个domain进行使用。
唯一标识：
* name：字符串，每一个storage pool中的volume的name都是唯一的
* Key: 唯一标识pool中的volume，要求重启等操作过程中，保持稳定不变
* Value: 对应volume的文件系统路径，在host上每个volume的Value唯一
  
#### Host devices
指host上所有可用的硬件设备，包括物理设备、PCI设备、逻辑设备等。
唯一标识：
* name：字符串，每一个host上device的name都是唯一的



### Driver model
libvirt库提供API和ABI（应用二进制接口），针对每一种虚拟化技术，都提供一种ABI的实现。

libvirt提供的API是一种更加高层的抽象，是Hypervisor不感知的，不局限于某一种虚拟化技术。


常见的Hypervisor drivers：
* Xen：提供半虚拟化与全虚拟化。
* QEMU：任意基于QEMU的虚拟化技术，典型的如KVM。
* UML：User Model Linux Kernel，纯半虚拟化技术
* OpenVZ：感觉不怎么常用
* LXC：Linux Container Based 虚拟化
* Remote：与libvirtd通信的任意安全RPC服务
* Test：模拟driver


