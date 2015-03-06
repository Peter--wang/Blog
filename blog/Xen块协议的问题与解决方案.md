　　早些时候，Xen Linux的维护者Konrad Rzeszutek Wilk提出了一些针对PV块协议的优化，可用于减少guest的虚拟磁盘开销。

>以下内容参考、翻译自：[a list of possible improvements to the Xen PV block protocol](https://docs.google.com/document/d/1Vh5T8Z3Tx3sUEhVB0DnNDKBNiqB_ZA8Z5YVqAsCIjuI/edit)

块协议里有一些明显的问题：
-
>假设guest和host都是64位的。

1、Segment最多有11个页。也就是说，一个请求里最多容纳44kB的数据。一个环可以放32个请求，算下来，一个环上最多可放1.4MB的数据。
>此问题3.11内核已经解决，请参考：Indirect descriptors for Xen PV disks](http://blog.csdn.net/perfecter/article/details/44087301)

2、生产和消费的索引位于同一个高速缓存行（cache line），这意味着在现有的硬件上，读和写会竞争同一个高速缓存行，会引起socket之间的乒乓效应。
3、请求和响应在同一个ring上，也会造成socket之间的乒乓效应，因为高速缓存行的所有权在socket之间切换。
4、缓存对齐。现在协议中是以16bit对齐的。这是个笨方式，请求和响应有时要跨多个缓存行。
5、中断缓和。每次处理完ring上的请求，都要做kick。有更好的方法，当处理大量数据时，可以用现有的网络中断缓和技术。
6、潜伏。request只有最多44KB的数据，意味着1MB的数据块要被切分成为多个请求。
7、未来的扩展。DIF/DIX等
>　　DIF是SCSI标准的一个新特性，它将扇区大小由512字节扩展成520字节。这些额外的字节包含了数据完整性区域（Data Integrity Field (DIF)），基本思想就是让HBA计算一个checksum值。
>　　向块写数据时，checksum保存在DIF中。存储设备接受数据的同时检查这个checksum值，并和数据一并保存。读取数据时，存储设备和HBA检查checksum。
>　　数据完整性扩展（Data Integrity Extension (DIX)）允许将这个检查上移到处理栈中：应用程序计算checksum并将之传递给HBA。这就提供了一种完整的端到端的数据完整性校验

> 摘自：What is DIF/DIX](https://access.redhat.com/solutions/41548)

8、将相应和请求分成两个ring。现有是实现是，一个block ring由一个线程处理。没有理由不这么做，相应和请求分别由两个线程处理，特别是他们可以在不同的cpu上调度。再进一步，他们可以分离成多队列（multi-queues），一个队列在一个vCPU上处理。
>多队列请参考：　[多队列块层](http://blog.csdn.net/perfecter/article/details/44067457)　

9、ring上好多空间都被浪费了。因为我们把请求和响应都放在同一个ring上，所以搞的响应结构体的大小和请求结构体的大小相同，都是112字节。如果request结构体扩展了（struct blkif_sring_entry扩展到了1500字节），response结构体也要跟着扩展。但实际上response用不了多少空间。将request和response分开，可以简化这个结构。
10、32bit和64bit的对决。在64bit guest下，request结构体是112字节，而在32bit guest下是102字节。这就需要host做额外的处理。
![blkif_ring](http://img.blog.csdn.net/20150305211304208)
这张草图显示出ring在64字节偏移处的内存分布（缓存行）。当然未来CPU也许会使用新的缓存行（32字节？）

解决方案
-
1、原文提出了两个解决方案，分别是Spectralogic和Intel提出的方案。但这两个方案都不是upstream上的方案。所以我也就偷懒不写了。
>upstream方案请参考：[Xen PV disk间接描述符](https://github.com/Peter--wang/test/blob/master/blog/Xen%20PV%20disk%E9%97%B4%E6%8E%A5%E6%8F%8F%E8%BF%B0%E7%AC%A6.md)

2、