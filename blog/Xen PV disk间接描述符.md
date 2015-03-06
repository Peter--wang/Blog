早期的Xen利用“环”来做guest和driver domain之间的IO交换，由于设计限制，一次处理的最大IO为1408K，所以在处理大IO时，成为性能瓶颈。为了解决这个问题，Xen社区借鉴VirtIO的实现方式，提出了Indirect Descriptors这个概念。
>以下内容参考和翻译自：[Indirect descriptors for Xen PV disks](https://blog.xenproject.org/2013/08/07/indirect-descriptors-for-xen-pv-disks/)

Xen PV disk协议
-
Xen PV disk协议非常简单，它只需要在guest和[driver domain](http://wiki.xen.org/wiki/Driver_Domain)之间映射一个用于处理请求和读取响应的4K共享内存页。前后端驱动用的结构也非常简单，就是我们经常所说的“环”。“环”里面包含一个循环队列，和一些请求响应的计数器。
![PV disk protocol](https://blog.xenproject.org/wp-content/uploads/2013/07/PV_Protocol.png)
除去索引，环里可容纳的请求的最大个数向下取整为2的N次幂，结果就是环里可容纳的请求和响应的个数。按这个计算方式，环里可容纳的请求个数为32（(4096 – 64)/112 = 36）。

以下是请求的代码结构：
```
struct blkif_request {
    uint8_t        operation;
    uint8_t        nr_segments;
    blkif_vdev_t   handle;
    uint64_t       id;
    blkif_sector_t sector_number;
    struct blkif_request_segment {
        grant_ref_t gref;
        uint8_t     first_sect, last_sect;
    } seg[11];
};
```
> 作者省略了一些本文不涉及的字段

这个结构体中，最重要的是“seg”，因为它里面保存了IO数据。每个seg都持有一个grant page的引用，grant page是前后端用于传输数据的共享内存页。如果11个seg中，每个seg持有一个4K共享内存页的引用，则一个请求中最多传送44K的数据（4KB * 11 = 44KB）。一个ring中可容纳32个请求，则一次课传送的最大数据量为1408KB（44KB * 32 = 1408KB）。这个数据量不是很大，当今的磁盘基本上都可以很快处理完毕。所以这种实现方式就成了PV协议自身的瓶颈。

Intel和Citrix都分别提出了解决这个问题的方案。Intel的方案是，创建只容纳seg的ring，这种方式可以将一个请求的最大数据量提升至4MB。Citrix则为ring引入multiple pages，来增加ring上可容纳的数据量。但这两种方式都没有合入主干。Indirect Descriptors和Intel的方案类似，都是提升一个请求的数据量，但Indirect Descriptors可容纳的数据量远大于4MB。
Xen indirect descriptors implementation
-
Konrad建议借鉴VirtIO的indirect descriptors来解决这个问题，而原文的作者也非常赞成这是最佳的方案。因为这个方式加大了一个请求可容纳的数据量，而现代的存储设备也都更倾向于处理大请求。Indirect descriptors引入了只用于读写操作的新请求类型，将有所的seg放入共享内存页中，而不是直接放入请求中。下面是indirect请求的结构：
```
struct blkif_request_indirect {
    uint8_t        operation;
    uint8_t        indirect_op;
    uint16_t       nr_segments;
    uint64_t       id;
    blkif_sector_t sector_number;
    blkif_vdev_t   handle;
    uint16_t       _pad2;
    grant_ref_t    indirect_grefs[8];
};
```
主要是把seg数组替换成了grant引用数组，没个grant共享内存页都用以下这个结构体填充：
```
struct blkif_request_segment_aligned {
    grant_ref_t gref;
    uint8_t     first_sect, last_sect;
    uint16_t    _pad;
};
```
这个结构体是8字节，也就是一个共享页中可以容纳512个segment。按照定义，一共有8个grant，也就是一个请求最多容纳4096个segment，算下来一个request就可以容纳16MB的数据，这已经比之前的44KB要大很多了。如果全部用Indirect segment，则一个ring最多可以容纳512MB的数据。

这样的数据量显然很大，但为了保持内存用量和磁盘吞吐量之间的平衡，后端驱动中一个请求的最大segment数被设定为256，前端驱动这个数值被设定为32。如果用户需要，也可以在启动选项里面改这个参数，像下面这样添加一个启动选项：
```
xen_blkfront.max=64
```
这样就把segment的数量由32改成了64。对于不同的存储，没有一个规则能够指定这个值多少是合适的。所以针对不同的存储，最好还是测试一下。

PV块协议的Indirect descriptors已经合入到了3.11内核，要确定guest和driverdomain的内核都支持Indirect descriptors。