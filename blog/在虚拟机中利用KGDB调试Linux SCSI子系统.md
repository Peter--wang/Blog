[toc]
本文描述了在虚拟机中，利用KGDB双机联调NBD驱动的准备过程以及使用频率较高的调试命令。以此为例，介绍调试Linux内核以及内核模块的一种较常用的方法。

在进行内核调试时，系统已经不会响应用户态程序，所以需要使用两台计算机利用串行端口或网络进行双机联调，本文介绍的是利用串行端口进行联调。
##1 准备工作
新建虚拟机虚拟机，并安装linux系统。本教程使用的是3.9.4-200.fc18.x86_64。

##2	开启内核调试
###2.1	安装debug模式的内核

> localhost:/ # <font color='red'>yum install kernel-debug</font>

###2.2	开启服务端（调试目标）
利用串口发送和接受串口消息：

> localhost:/ # <font color="red">echo ttyS0 > /sys/module/kgdboc/parameters/kgdboc</font>

引发内核调试断点：

> localhost:/ # <font color="red">echo g > /proc/sysrq-trigger</font>

在服务端运行上面两个命令，就使系统进入调试状态。此时服务端已经停止用户态响应。
###2.3	开启客户端（调试机）
这里讲的调试机，就是虚拟机的宿主机，相当于CN节点
安装debug info包：

> localhost:/ # <font color="red">yum install kernel-debuginfo</font>

启动gdb调试程序：
> localhost:/usr/src/linux # 
<font color="red">gdb /usr/lib/debug/lib/modules/3.9.4-200.fc18.x86_64.debug/vmlinux</font>

等待gdb启动完成后，显示如下信息，说明调试信息加载成功：

> Reading symbols from /usr/lib/debug/lib/modules/3.9.4-200.fc18.x86_64.debug/vmlinux...done.

查看虚拟机使用的串口：
> localhost:/ # <font color="red">virsh dumpxml fc18-san-emu</font>
（这里的虚拟机名称改成自己的）

xml回显：

```
<console type='pty' tty='/dev/pts/0\>
（/dev/pts/0 路径每次可能不同）
```

设置gdb连接服务端：
>(gdb) <font color="red">set remotebaud 115200</font>
 (gdb) <font color="red">target remote /dev/pts/0</font>

等待gdb输出以下消息，说明调试环境建立成功：
>Remote debugging using /dev/pts/0
>kgdb_breakpoint () at kernel/debug/debug_core.c:1014
>1014		wmb(); /* Sync point after breakpoint */    

##3	开始调试
###3.1	调试SCSI模块简单示例
在sd代码的sd_probe函数下断点：
> (gdb) <font color="red">b sd_probe</font>

gdb输出如下信息说明断点设置成功：
>Breakpoint 1 at 0xffffffff814b85d0: file drivers/scsi/sd.c, line 2889.

恢复运行服务端：
> (gdb) <font color="red">c</font>

在服务端上发现SCSI设备时，系统会在刚才的断点上中断：
> Breakpoint 1, sd_probe (dev=<optimized out>) at drivers/scsi/sd.c:2889
2889	{  
  
至此，SCSI扫描相关的调试工作就可以开始了
