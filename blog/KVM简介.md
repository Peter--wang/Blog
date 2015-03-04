KVM 全称 Kernel-based Virtual Machine， 即基于 Linux 内核的虚拟化技术， 精确的说，就是 KVM VMM 的核心功能是通过一个 Linux 内核模块实现的。 “基于 Linux 内核”是 KVM 在软件实现上不同于其他 VMM 实现的最重要特点， 使得 KVM 在实现上能获得如下好处 :
利用 Linux 内核已有的功能和基础服务，减少不必要的重新开发。 如任务调度，物理内存管理，内存空间虚拟化，电源管理等功能，通常是一个 VMM 所必须具备的，但 KVM 可不必重新开发这些功能，直接使用 Linux 上已经相当成熟的技术。
利用强大的 Linux 社区，吸引优秀的 Linux 内核程序员参与到 KVM 的开发中， 壮大 KVM 的群体， 这些程序员以及红帽等 Linux 社区背后的厂商，也乐于在 Linux 上发展一个成熟的 VMM 技术。
可以长期享受 Linux 内核技术不断成熟和进步的好处，优化 KVM 的实现。 如 Linux 内核的 HugeTLBPage 技术可以用于削减 KVM 在虚拟机内存使用上的性能开销， eventfd 可以用于提升 KVM 内核执行路径和 Qemu-kvm 用户空间的交互效率。
KVM 在 VMM 的理论上属于硬件辅助的虚拟化技术， 即 KVM 需要利用 AMD-V 提供的虚拟化能力。AMD-V 让 KVM 上的虚拟机正常情况下运行在 “guest” 模式， 在执行敏感的的指令或行为时透明地切换到 “host” 模式，并在 “host”模式由 KVM VMM 的代码仿真那些敏感的指令或行为，完成后又回到“guest”模式由虚拟机运行其正常的代码。 KVM 的 VMM 代码实际上就是当虚拟机被捕获时才进入，执行仿真代码，然后又执行状态切换回到虚拟机代码直到其下一次被捕获，如此循环不断。 当然 KVM 的 VMM 在仿真复杂的行为时，可能需要用户空间的帮助， 所以 KVM 在此期间切换到用户空间。

KVM 在 Linux 上的实现

KVM 的 VMM 代码包括内核代码和用户空间代码俩部分。 内核代码即 kvm.ko 模块的代码，分布在内核源码的 virt/kvm 和 arch/x86/kvm 两个目录下， 前者是和硬件结构无关的代码， 后者是和硬件结构相关的代码，其中和 AMD-V 相关的代码由 arch/x86/kvm/svm.c 文件提供。KVM 内核空间代码实现的功能包括如下几个方面 :

1. 实现硬件辅助的虚拟化的核心功能。包括实现“guest”模式及模式切换，虚拟 CPU 的状态和控制，指令仿真等所有之前提到的 x86 虚拟化所要解决的问题。代码分布在 arch/x86/kvm 下 x86.c， svm.c， emulate.c 等文件中。

2. 提供用户空间对KVM控制。实现 /dev/kvm 字符设备接口用于 qemu-kvm 对 KVM 的控制，包括创建虚拟机，创建虚拟 CPU， 创建虚拟机内存空间，运行虚拟机等功能的 ioctl() 接口 . 其实现的 ioctl() 功能主要分成二类，针对单个虚拟机整体的功能和针对单个虚拟 CPU 的功能。 相关代码分布在 kvm_main.c， x86.c， svm.c 等文件中。

3. 对x86平台设备进行仿真。包括仿真 PIC，IOAPIC，Local APIC 及 PIT 的代码。

4. 实现IO Port空间和MMIO空间的仿真。这是实现平台设备和外部设备仿真所需要的基础功能。代码分布在 coalesced_mmio.c， iodev.h， emulate.c， x86.c 及 svm.c 等文件中。

5. 实现基于文件描述符的通知机制。代码在 eventfd.c 文件中。 其中 ioeventfd 机制用于内核空间仿真 PIO 后向用户空间发送通知， irqfd 机制用于 qemu-kvm 进程向虚拟机注入中断。

6.MMU虚拟化的支持。包括 shadow 页表的实现，TLB 的虚拟化。 主要代码在 mmu.c 文件中。

7. 和PCI-Passthrough相关的功能。代码分布在 kvm_main.c 和 iommu.c 中。当然对物理 IOMMU 单元的管理还需要 Linux 内核 IOMMU 设备抽象层的代码及厂家 IOMMU 驱动的代码，分别分布在 driver/base/iommu.c 及 arch/x86/kernel/ 下的 amd_iommu.c 和 amd_iommu_init.c 文件中。 对 IOMMU 的支持是独立于 KVM 的，它也可以被用于非系统虚拟化的情景。

用户空间的代码就是 qemu-kvm 的代码。 Qemu-kvm 是从 qemu 分支出来的项目 , 其目的就是利用传统 qemu 的功能，来实现各种虚拟设备的仿真，并提供对 KVM 进行控制的用户空间接口。Qemu-kvm 实现的功能包括如下方面 :

1. 实现与KVM内核接口的用户空间逻辑。包括对 x86 虚拟机 PC 结构的定义，虚拟机的创建，平台设备的创建，虚拟 CPU 的创建接口。

2. 各种层次和类型的硬件设备的仿真。包括 BIOS， PCI Hub，PCI 设备 ( 包括 USB 控制器，IDE 控制器，SCSI 适配器， 声卡，显卡，网卡等 )，USB 设备， IDE 磁盘， SCSI 磁盘。 Qemu-kvm 采用一个 qdev 的设备模型，来简化不同类型设备的仿真实现。

3. 虚拟块设备的不同磁盘Image文件格式的支持。包括 qcow2， qed， vmdk， vpc 等。 这方面的能力影响 KVM 平台的可管理性 ( 如快照 ) 及 KVM 虚拟化技术和管理软件的兼容性。

4.VNC，SPICE等表示层协议的支持。这方面的支持能力决定声卡，显卡的仿真在用户使用层次的效果。

5.Virtio设备的后端。VirtIO 是 KVM 上设备虚拟化的标准，即完全虚拟出来的网络和块设备，客操作系统端使用虚拟的前端驱动和 Qemu-kvm 实现的后端之间紧密合作实现的网络和块设备逻辑， VirtIO 设备的效率要远高于完全由 Qemu-kvm 仿真出的传统的网络和块设备。

6.QMP 协议的支持。 QMP 即 Qemu Monitor Protocol， 是虚拟化管理接口层，如 libvirtd， 用来控制和查询 Qemu-kvm 状态及信息的协议。
