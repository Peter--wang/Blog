前言
-
最近在  搞基  于SCSI的技术项目，所以要学习一下linux对于scsi的实现，以及一些iSCSI的知识。以前对一些技术的学习总是没有留下学习资料，没有传承。这次学习scsi子系统，还是要留下一些东西。对系统有了进一步的理解，并且能挤出时间的话，我就会来更新这个连载。

    本次研究的主要对象是Open iSCSI（2.0.873）/ Linux（3.11.4）/  Linux SCSI target（1.0.38）

iscsiadmin
-
iscsiadmin是提供给用户使用的命令行程序，主要功能就是设置iSCSI的一些相关功能属性，比如发现iqn；设置认证模式、用户名、密码；连接SCSI设备等等。但是iscsiadm又不做具体的工作，它只是把这些信息通过IPC调用传递给iscsid这个服务程序，由iscsid来执行真正的操作。而这里的IPC实际上就是一个本地的socket，iscsid监听这个本地socket，iscsiadm通过这个socket和iscsid交互。
iscsid
-
iscsid可以看做是用户和内核的一个桥梁，它通过mgmt_ipc（本地socket）这个东西和iscsiadm交互，响应用户的请求；利用control_fd（netlink）和内核交互，把用户的指令发送给内核。
scsi_transport_iscsi
-
这是iscsi传输层，一个内核模块。Linux在设计的时候，非常好的利用了面向对象的设计思想，这个模块就可以看做是一个对scsi传输层的抽象，它仅实现一些传输层通用的流程，具体的实现由它的子类们完成。
iscsi_tcp
-
基于tcp协议的iSCSI传输模块，是上面讲到的scsi_transport_iscsi的具体实现。传输层需要的基于tcp的数据收发就是这个模块实现的。
这里要注意一个问题，上图实际上少画了一条线，iscsid到tgtd之间应该有一个socket路径。在创建连接、session和进行认证的时候，实际上是iscsid创建了一个socket，和tgtd进行交互。以上操作都正确完成了之后，iscsid会将这个socket的句柄传递到内核，iscsi_tcp正是使用的这个socket。
后续计划
后面主要会研究一下这三个方面：
iscsi的登陆认证流程，基于chap的认证流程。
iscsi磁盘的扫描流程，一个target的设备如何映射到主机上成为一个sd设备。
iscsi磁盘的IO流程，从读写函数开始，一个IO请求是如何变成scsi命令，然后发送到target端的。
（声明：如有疑问，欢迎骚扰）
