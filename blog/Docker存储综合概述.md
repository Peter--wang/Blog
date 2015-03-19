[TOC]
>本文参考翻译自：[Comprehensive Overview of Storage Scalability in Docker](http://developerblog.redhat.com/2014/09/30/overview-storage-scalability-docker/)

背景
-
Docker于2013年开源，并以AUFS为基础（advanced multi layered unification filesystem）。这个联合文件系统为Docker的几个卖点提供支持：

 - 容器创建速度
 - [写时拷贝](http://en.wikipedia.org/wiki/Copy-on-write) 镜像->容器

![Docker](https://rhdevelopers.files.wordpress.com/2014/05/homepage-docker-logo.png)

虽然Docker还是支持AUFS，但是[Ubuntu禁用了AUFS](https://lists.ubuntu.com/archives/ubuntu-devel/2012-February/034850.html)，并将它移到linux-image-extra。
>主要是因为AUFS没有上upstream，并且因此给Ubuntu内核维护小组带来很大的负担。
>从Oneiric开始支持OverlayFS，并且OverlayFS已进入3.18内核。

替代品
-
我们需要找到AUFS的替代品，在upstream上、稳定、可维护性好、能够长期支持、性能高。

有趣的是，红帽的内核工程师们（Joe Thornber和Mike Snitzer）已经提出了符合以上标准的方案：[device mapper精简置备](https://www.kernel.org/doc/Documentation/device-mapper/thin-provisioning.txt)。一些红帽的工程师（主要是Alex Larsson）基于device mapper为Docker 0.7写了新的驱动。如果你使用的是Fedora、CentOS、RHEL打包的Docker，默认的驱动就是device mapper，使用将稀疏文件mount的loopback设备。

传统Docker的简便安装和使用，在Device mapper精简置备加loopback这种方式下得以保留。对于在迭代项目中的敏捷开发者来说，这是好事。没有影响生产力。这也是Docker胜出的主要原意之一。

尽管已经有了device mapper，但作为企业软件公司，还是有责任比开发用例做的更多。所以红帽的工程师们做了更深入的Docker存储调查，他们意识到，在真实地生产环境中，需要有些特别的定制，尤其是存储和网络。

存储评估
-
红帽的工程师找了几种满足Docker基本标准（快速写时拷贝）的存储后端。相关的内核补丁有：

 - 具有更好扩展性和性能的[内核](https://lkml.org/lkml/2014/3/4/881)和[device-mapper](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=0e04c641b199435f3779454055f6a7de258ecdfc)
 - Fedora打包的btrfs
 - PoC编码支持[Docker中的OverlayFS](https://github.com/docker/docker/pull/7619)（需要内核支持）
 - 验证Docker卷提供几乎逻辑的性能
 - 安全、隔离以及[SELinux支持](http://opensource.com/business/14/9/security-for-docker)。


>We looked at different storage backend variations that met the basic criteria of Docker – fast CoW.  The kernel got patched.  Many times.  Out came several things:

> - A more scalable and performant kernel and device-mapper thinp …massive impact. 
> - Enabling btrfs in Fedora-packaged Docker PoC code to support OverlayFS in Docker (kernel support required)
> - Verification that Docker “volumes” provided near bare metal performance. 
> - Scoped effort necessary to bring proper security, isolation and SELinux support.

几种满足Docker镜像/容器要求的存储：

 - Device mapper loopback（loop-lvm）
 - Device mapper（direct-lvm）
 - BTRFS（Upstream上的Docker的默认配置）

在Fedora上，还有一个选择：

 - OverlayFS（以进入内核3.18版本）

Fedora和CentOS都没有发型支持AUFS的内核。如果没有指定类型，Docker会按照下面的优先排序选择存储驱动：
```
// Slice of drivers that should be used in an order
 priority = []string{
                    "aufs",
                    "btrfs",
                    "devicemapper",
                    "vfs",
                    "overlayfs",
```
或者取决于主机内核提供的功能，或者守护进程启动时的-s选项

各种存储综述
-

###Device Mapper loop-lvm
按照[文档](https://github.com/docker/docker/blob/master/daemon/graphdriver/devmapper/README.md)描述：device mapper驱动使用device mapper精简置备模块（dm-thin-pool）来实现写时拷贝快照。每个device mapper精简池都由两个块设备组成，一个放数据、一个放元数据。这些块设备默认为loopback设备，后端为自动生成的稀疏文件。

比如
```
# ls -alhs /var/lib/docker/devicemapper/devicemapper
506M -rw-------. 1 root root 100G Sep 10 20:15 data
1.1M -rw-------. 1 root root 2.0G Sep 10 20:15 metadata
```
100G的文件，实际占用仅为506MB，和metadata文件一样，都是稀疏文件。

看一下lsblk的输出。挂载了两个loop设备。一个用于容器存储，一个用于元数据，都是device mapper精简置备的。
```
loop0 7:0 0 100G 0 loop 
└─docker-252:3-8532-pool 253:0 0 100G 0 dm 
 └─docker-252:3-8532-base 253:1 0 10G 0 dm 
loop1 7:1 0 2G 0 loop 
└─docker-252:3-8532-pool 253:0 0 100G 0 dm 
 └─docker-252:3-8532-base 253:1 0 10G 0 dm
```
如上述，loop-lvm的默认配置是100G的池（所有容器共享）。如果需要大于100G，则需要修改systemd的配置文件或者/etc/sysconfig/docker。如果选择systemd，确保在/etc/systemd/system/docker.service中创建一个[override](http://www.freedesktop.org/software/systemd/man/systemd.unit.html)文件，而不是修改 /usr/lib/systemd/system/docker.service。
```
# ExecStart=/usr/bin/docker ... \
--storage-opt dm.loopdatasize=500GB \
--storage-opt dm.loopmetadatasize=10GB

And you'll need to reload systemd:
# systemctl daemon-reload
# systemctl start docker
```

###Device Mapper:  direct-lvm
direct-lvm还是使用LVM、device mapper和dm-thinp内核模块。不同的是，不再使用loopback设备，直接使用raw分区。在中等负载和高密度下具有性能优势。

利用LVM创建两个设备，一个空间大的用于Docker thinp数据，一个空间小的用于thinp元数据。比如100G和4G。假设在/dev/sdc上做LVM设备：/dev/direct-lvm/data和/dev/direct-lvm/metadata。
```
 # pvcreate /dev/sdc
 # vgcreate direct-lvm /dev/sdc
 # lvcreate --wipesignatures y -n data direct-lvm -l 95%VG
 # lvcreate --wipesignatures y -n metadata direct-lvm -l 5%VG

This next  step is not necessary the first time you set it up.
It re-initializes the storage, making it appear blank to Docker.

This would be how you "wipe" direct-lvm (since there's no filesystem, you can't exactly mkfs ;)

# dd if=/dev/zero of=/dev/direct-lvm/metadata bs=1M count=10
```
通过给Docker守护进程命令行附加单独的–storage-opt标记，来给device mapper驱动床送Docker配置。可配置的选项有：

 - dm.basesize: 基础dm设备大小 (默认10G)
 - dm.loopdatasize: loopback文件初始大小
 - dm.loopmetadatasize: loopback元数据文件初始大小
 - dm.fs: 基础镜像使用的文件系统 (xfs或者ext4)
 - dm.datadev: 设置数据使用的RAW块设备 
 - dm.metadatadev: 设置元数据使用的RAW块设备 
 - dm.blocksize: 精简池的块大小。默认64K.

使用direct-lvm时，在systemd unitfile或者/etc/sysconfig/docker中设置dm.datadev和dm.metadatadev选项：
```
ExecStart=/usr/bin/docker ... \
--storage-opt dm.datadev=/dev/direct-lvm/data \
--storage-opt dm.metadatadev=/dev/direct-lvm/metadata

And you'll need to reload systemd:
# systemctl daemon-reload
# systemctl start docker
```
原作者常常加上dm.fs=xfs，因为XFS性能已经被优化了多次。

需要注意的是loop-lvm不支持O_DIRECT，所以看起来，loop吞吐量能够到GB/S。但其实系统后面会刷缓存。direct-lvm支持O_DIRECT。

###BTRFS
[BTRFS](https://btrfs.wiki.kernel.org/index.php/Main_Page)看起来天然支持Docker。它支持cow（写时拷贝），性能不错，开发活跃。BTRFS不支持SELinux，也不支持页缓存共享。

假设在/dev/sde上做brfs：
```
# systemctl stop docker
# rm -rf /var/lib/docker
# yum install -y btrfs-progs btrfs-progs-devel
# mkfs.btrfs -f /dev/sde
# mkdir /var/lib/docker
# echo "/dev/sde /var/lib/docker btrfs defaults 0 0" >> /etc/fstab
# mount -a
```
检查文件系统
```
# btrfs filesystem show /var/lib/docker
 
 Label: none uuid: b35ef434-31e1-4239-974d-d840f84bcb7c
 Total devices 1 FS bytes used 2.00GiB
 devid 1 size 558.38GiB used 8.04GiB path /dev/sde

# btrfs filesystem df /var/lib/docker
 Data, single: total=1.01GiB, used=645.32MiB
 System, DUP: total=8.00MiB, used=16.00KiB
 System, single: total=4.00MiB, used=0.00
 Metadata, DUP: total=3.50GiB, used=1.38GiB
 Metadata, single: total=8.00MiB, used=0.00
 unknown, single: total=48.00MiB, used=0.00

# btrfs device stats /dev/sde
 [/dev/sde].write_io_errs 0
 [/dev/sde].read_io_errs 0
 [/dev/sde].flush_io_errs 0
 [/dev/sde].corruption_errs 0
 [/dev/sde].generation_errs 0
```
配置Docker unitfile或者/etc/sysconfig/docker使用btrfs：
```
# systemctl daemon-reload
# systemctl start docker
# docker info|grep Storage
Storage Driver: btrfs
```
启动几个容器
```
# btrfs subvolume list /var/lib/docker | wc -l
 4483

# btrfs subvolume list /var/lib/docker | head -5
 ID 258 gen 13 top level 5 path btrfs/subvolumes/4e7ab9722a812cb8e4426feed3dcdc289e2be13f1b2d5b91971c41b79b2fd1e3
 ID 259 gen 14 top level 5 path btrfs/subvolumes/2266bc6bcdc30a1212bdf70eebf28fcba58e53f3fb7fa942a409f75e3f1bc1be
 ID 260 gen 15 top level 5 path btrfs/subvolumes/2b7da27a1874ad3c9d71306d43a55e82ba900c17298724da391963e7ff24a788
 ID 261 gen 16 top level 5 path btrfs/subvolumes/4a1fb0a08b6a6f72c76b0cf2a3bb37eb23986699c0b2aa7967a1ddb107b7db0a
 ID 262 gen 17 top level 5 path btrfs/subvolumes/14a629d9d59f38841db83f0b76254667073619c46638c68b73b3f7c31580e9c2
```

###OverlayFS
[OverlayFS](https://git.kernel.org/cgit/linux/kernel/git/mszeredi/vfs.git/tree/Documentation/filesystems/overlayfs.txt?h=overlayfs.current)是一个现代联合文件系统，也符合Docker的基本需求。简单说来，OverlayFS联合了一个下层（父）文件系统、一个上层（子）文件系统以及一个工作目录（同子相同的文件系统）。下层文件系统是基础镜像，当创建一个Docker容器，一个新的包含delta的上层文件系统将被创建。

OverlayFS具有的优点：
 - 快速
 - 允许页缓存共享

OverlayFS也有几个缺点：

 - 要求内核版本很新（3.18以上）
 - 和btrfs一样不支持SELinux

以下是在Docker上使用OverlayFS的步骤：

 - 为OverlayFS创建下层文件系统，可以使XFS或EXT4文件系统的逻辑卷。
 - 配置Docker使用OverlayFS
 ```
 ExecStart=/root/overlayfs/dynbinary/docker ... -s overlayfs
 ```
 
 - 用‘docker info’检查，并运行容器：
```
# docker info
 Containers: 1
 Images: 28
 Storage Driver: overlayfs
 Execution Driver: native-0.2
 Kernel Version: 3.17.0-0.rc1.git0.1.playground.fc22.x86_64
 Debug mode (server): true
 Debug mode (client): false
 Fds: 19
 Goroutines: 28
 EventsListeners: 0
 Init SHA1: 2fa3cb42b355f815f50ca372f4bc4704805d296b
 Init Path: /root/overlayfs/dynbinary/dockerinit
```

###检查配置
利用iostat命令来检查容器的I/O被发到了新的存储上
```
# docker run -d fedora dd if=/dev/zero of=outfile bs=1M count=2000 oflag=direct && iostat -x 1|grep sdc
```
###为什么人人关心联合文件系统
联合文件系统提供更为高效的内存使用，允许多个容器访问同一个文件时共享页缓存，更加节省内存。相较于内存去重技术，比如KSM，会节省一些用于页扫描和合并的cpu消耗。



###几个测试数据：
![Scalability](http://www.breakage.org/wp-content/uploads/2014/09/20140819-container-create-destroy-1000-containers3.png)


![Page Cache Re-use (shared inodes)](https://rhdevelopers.files.wordpress.com/2014/09/docker-page-cache.png)

![Page Cache Re-use](https://rhdevelopers.files.wordpress.com/2014/09/docker-page-cache-vmstat-bi.png)