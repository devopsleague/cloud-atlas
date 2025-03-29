.. _kvm_storage_io:

==================
KVM存储I/O
==================

I/O Scheduler
=================

默认HDD的磁盘I/O调度是完全公平队列(CFQ)，这种调度提供了对所有进程完全公平对磁盘I/O带宽。不过，在虚拟化环境中，需要针对不同设备使用不同I/O调度。

为了能够获得host主机和VM虚拟机的最佳性能，在虚拟化的物理主机上建议使用 ``deadline`` scheduler(针对SSD优化)，而在VM Guest虚拟机中则使用 ``noop`` scheduler(禁用I/O调度)。

- 检查当前自磁盘I/O调度::

   cat /sys/block/sda/queue/scheduler

例如，在我的 :ref:`hpe_dl360_gen9` 服务器，使用SSD磁盘，安装了 :ref:`ubuntu_linux` ，系统安装后使用的磁盘I/O调度就是 ``deadline`` ::

   [mq-deadline] none

- 要修改系统的默认(所有磁盘)的I/O调度器，使用内核参数 ``elevator`` ，即对虚拟化的物理主机::

   elevator=deadline

对VM虚拟机，则使用::

   elevator=noop

如果需要针对每个磁盘使用不同的I/O schedulers，则创建 ``/usr/lib/tmpfiles.d/IO_ioscheduler.conf`` ，按照以下案例配置，例如 ``/dev/sda`` 设置 ``deadline`` scheduler , ``/dev/sdb`` 设置 ``noop`` scheduler ::

   w /sys/block/sda/queue/scheduler - - - - deadline
   w /sys/block/sdb/queue/scheduler - - - - noop

异步I/O
=========

很多虚拟磁盘后端的实现采用了Linux异步I/O (Asynchronous I/O, aio)。默认情况下，aio上下文的最大数量是 ``65536`` ，但主机上运行了上百使用Linux异步I/O的VM虚拟机时候，有可能会超出这个限制。所以，当在VM Host主机上运行大量VM Guests时，需要增加 ``/proc/sys/fs/aio-max-nr`` ::

   cat /proc/sys/fs/aio-max-nr

默认显示::

   65536

- 可以通过以下命令修订 ``aio-max-nr`` ::

   sudo echo 131072 > /proc/sys/fs/aio-max-nr

- 如果要持久化 ``aio-max-nr`` ，则配置一个本地 ``sysctl`` 文件，例如 ``/etc/sysctl.d/99-sysctl.conf`` ::

   fs.aio-max-nr = 1048576

I/O虚拟化
===============

I/O虚拟化有不同的技术，各自具有利弊

.. csv-table:: I/O虚拟化解决方案
   :file: kvm_storage_io/io_virt_solutions.csv
   :widths: 30, 30, 40
   :header-rows: 1

在 :ref:`priv_cloud_infra` 采用上述 ``Device Assignment(pass-through)`` 方式，也就是 :ref:`iommu` VFIO 标准(Virtual Function I/O)。VFIO驱动直接输出设备访问到一个安全内存(IOMMU)保护环境的用户空间。使用VFIO，VM Guest虚拟机可以直接访问VM Host物理主机硬件设备(pass-through)，避免了模拟带来的性能损失。注意，这种模式不允许共享设备(和SR-IOV不同)，每个设备只能指派给一个唯一VM Guest。要实现VFIO，需要满足条件有:

- Host物理主机服务器CPU和主板芯片以及BIOS/UEFI支持
- BIOS/EFI激活了IOMMU
- 内核参数设置激活IOMMU: 对于Intel处理器，内核参数使用 ``intel_iommu=on``
- 系统提供了VFIO架构，也即是内核加载了模块 ``vfio_pci``

pass-through NVMe存储
------------------------

NVMe可以通过PCIe直接分配给虚拟机使用，可以实现最大化的存储性能。这种方式绕过了物理主机上的操作系统控制，并且直接通过PCIe总线访问存储，是虚拟化存储中性能最高的技术。

通过 :ref:`ovmf` 可以实现PCI设备直通给虚拟机，获得最好的I/O性能

pass-through SATA控制器
-------------------------

SATA控制器是位于PCI总线，所以可以将SATA控制器直接passthrough给虚拟机，这样就几乎没有延迟或超载，提供了最高的带宽。这个方案的缺点是整个SATA控制器都被VM控制，所以这个SATA控制器无法在Host主机使用，也就是说你必须有2个SATA控制器，一个给Host主机使用，一个给VM虚拟机使用。由于主机上的SATA控制器数量有限(通常就2个)，扩展管理非常麻烦(Guest虚拟机使用这个SATA控制器时候，物理主机不能使用这个SATA控制器上的所有磁盘)。

.. note::

   需要仔细检查 :ref:`hpe_dl360_gen9` 服务器手册中有关内部SATA控制器如何和SFF存储连接，如果能够区分出不同的SATA控制器，可以实践将没有用于物理主机的启动盘上的SATA控制器pass-through给虚拟机，验证这个技术

pass-through 分区或磁盘
-------------------------------

参考 `Pass through a partition? <https://www.reddit.com/r/VFIO/comments/j443ad/pass_through_a_partition/>`_

- 可以将一个磁盘分区pass through给虚拟机，但是需要注意分区在虚拟机内部会视为一个完整磁盘，所以虚拟机在这个分区中创建完整的GPT分区表，从外部看来这是一个嵌套的(nested)分区
- 需要非常小心，在物理服务器上不能直接访问pass-through给虚拟机的分区中的数据

参考 `Disk Passthrough Explained <https://passthroughpo.st/disk-passthrough-explained/>`_ 的 ``Direct SATA Controller Passthrough via vfio-pci`` :

`lennard0711/vfio <https://github.com/lennard0711/vfio>`_ 提供了一个配置案例，并且 `arch linux: PCI passthrough via OVMF - Physical disk/partition <https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Physical_disk/partition>`_ 介绍了可以pass through磁盘或分区

这种方式比直接pass-through SATA控制器多了一层物理服务器操作系统控制，所以理论上性能会差一些

基于磁盘的虚拟化存储
======================

`Tuning VM Disk Performance <https://www.heiko-sieger.info/tuning-vm-disk-performance/>`_ 的 ``Disk-based storage`` 方案和我设想相近:

- 使用LVM逻辑卷来构建 ``raw`` 磁盘，直接分配给虚拟机使用(虽然也能直接用分区，但是分区只能固定数量，很难调节，而LVM卷可以在底层伸缩)
- 虚拟机磁盘配置 ``cache=none`` 提高性能

这种方式去掉了Host主机上的文件系统层，理论上可以提高性能，不过性能肯定不如pass-through

实际上Red Hat虚拟化文档就提供了这个解决方法 `第 12 章 创建和管理精简配置的逻辑卷（精简卷） <https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/assembly_thinly-provisioned-logical-volumes_configuring-and-managing-logical-volumes>`_

QEMU磁盘IO的比较
===================

Libvirt对于磁盘设备的AIO有异步(Asynchronous IO, AIO=Native)和同步(Synchronous, AIO=Threads)两种模式，在 :ref:`openstack` 默认使用 ``aio=theads`` ::

   <disk type='file' device='disk'>
     <driver name='qemu' typ'qcow2' cache='none' io='native'/>
     <source file='/home/psurise/xfs/vm2-native-ssd.qcow2'/>
     <target dev='vdb' bus='virtio'/>
     <address type='pci' domain='ox0000' bus='0x00' slot='0x06' function='0x0'/>
   </disk>
   <disk type='file' device='disk'>
     <driver name='qemu' typ'qcow2' cache='none' io='threads'/>
     <source file='/home/psurise/xfs/vm2-threads-ssd.qcow2'/>
     <target dev='vdb' bus='virtio'/>
     <address type='pci' domain='ox0000' bus='0x00' slot='0x07' function='0x0'/>
   </disk>

- IO threads模式使用CPU资源较少，带宽增加更多
- ``AIO=Native`` 限制更少，建议使用
- 由于在文件不是完全分配的情况下，Native AIO会阻塞VM，所以Native AIO不建议用于稀疏文件
- 在完全预分配的文件，本地磁盘或者逻辑卷的情况下，建议只使用 ``aio=native`` ，但是不要用于稀疏文件(阻塞)

- 基于文件的存储

  - 容易实现，qemu提供了 ``raw`` 和 ``qcow2``

    - raw (Raw disk image format) 格式简单并且易于迁移到其他虚拟化平台
    - qcow2 提供了更小的镜像(稀疏文件)以及加密、压缩和虚拟机快照功能

  - 如果要求更好的性能，选择 ``raw`` 镜像文件格式，而如果需要节约磁盘使用，则采用 ``qcow2`` （不过，正确设置 ``qcow2`` 可以获得接近 ``raw`` 的性能)

要使用 ``raw`` 镜像文件获得最佳性能，使用以下预分配磁盘空间方式创建文件::

   qemu-img create -f raw -o preallocation=full vmdisk.img 100G

要使用 ``qcow2`` 镜像文件获得最佳性能，应该增加 ``cluster`` 大小::

   qemu-img create -f qcow2 -o cluster_size=2M vmdisk.qcow2 100G

如果完全预分配(full) ``qcow2`` 镜像的磁盘空间，会有一些性能提升，但是 ``qcow2`` 默认是稀疏文件。

  - 对于 ``ext4`` 文件系统上的VM镜像，建议使用 ``aio=threads`` 选项；但是对于其他文件系统，建议使用 ``aio=native`` 。使用参数方法举例::

     -object iothread,id=io1 \
     -device virtio-blk-pci,drive=disk0,iothread=io1 \
     -drive if=none,id=disk0,cache=none,format=raw,aio=threads,file=/path/to/vmdisk.img \

- 基于磁盘的存储

qemu/kvm 虚拟机可以直接使用磁盘或分区，只需要将 ``-drive``
设置为指定分区而不是镜像文件名即可。直接使用磁盘作为虚拟存储会失去伸缩性，以及不能使用快照(用于备份)。这种情况下，解决的方法是使用LVM逻辑卷管理。也就是说，并不是直接把裸磁盘分配给虚拟机，而是使用LVM逻辑卷(也是块设备)来代替简单的磁盘或磁盘分区，这样就能获得直接的磁盘性能，同时提供逻辑卷管理的伸缩性(例如可以在底层添加磁盘，扩展逻辑卷等)。

注意，需要先使用LVM逻辑卷管理配置好，然后才能使用它(逻辑卷)作为虚拟机磁盘，所以操作会有些繁琐。

此外，在使用LVM逻辑卷作为虚拟机存储时，应该将虚拟机存储参数设置为 ``cache=none`` 来获得最佳性能。至于使用 ``aio=native`` 还是 ``aio=threads`` 设置则视系统中同时运行的虚拟机数量而定。在使用SSD存储的系统中，如果只运行一个VM，则使用 ``aio=theads`` 可以增加带宽；而同时运行很多VM，则使用 ``aio=native`` 可以获得较好的性能。

以下案例是使用 ``virtio-blk-pci`` 驱动访问存储分区 ``/dev/sdb1`` ::

   -object iothread,id=io1 \
   -device virtio-blk-pci,drive=disk0,iothread=io1 \
   -drive if=none,id=disk0,cache=none,format=raw,aio=threads,file=/dev/sdb1 \

以下案例是使用 ``virtio-scsi-pci`` 驱动定义一个 ioh3420 root port 驱动(PCIe)::

   -device pcie-root-port,bus=pcie.0,addr=1c.0,multifunction=on, port=1,chassis=1,id=root.1 \
   -object iothread,id=io1 \
   -device virtio-scsi-pci,id=scsi0,iothread=io1,num_queues=4,bus=pcie.0 \
   -drive id=scsi0,file=/dev/sdb1,if=none,format=raw,aio=threads,cache=none \
   -device scsi-hd,drive=scsi0 \

小结
======

- 通过pass-through（VFIO)方式获得最高性能(接近于物理主机): 对于存储、网络、GPU，使用VFIO都是性能最优方法
- 如果设备(GPU或网卡)支持 :ref:`sr-iov` 则可以结合 VFIO 实现将 VF 设备pass-through给虚拟机，不仅实现高性能I/O，同时也实现一个物理设备按需切分分配给不同虚拟机，最大化使用率
- SATA存储或NVMe存储的VFIO pass-through都是直接把控制器assign给虚拟机，所以会导致连接在控制器上的所有磁盘设备都归虚拟机使用。这种情况适合每个控制器上连接1个磁盘，对于NVMe存储，通过 :ref:`pcie_bifurcation` 可以实现分割成多个PCIe控制器，实现每个NVMe存储设备pass-through个不同的虚拟机。不过，对于SATA设备则没有这么方便
- 比pass-through（VFIO)性能略差的是采用 Para-virtualization 虚拟化技术，也就是采用 ``virtio-blk`` ``virtio-net`` ``virtio-scsi`` ，虽然性能略差，但是带来了易于热迁移以及管理运维方便的优势

我的部署策略:

- 物理主机libvirt使用 ``/dev/sda`` SSD存储独立划分的 ``/dev/sda4`` 构建LVM卷，但是物理主机不使用文件系统，而是直接把LVM卷输出给虚拟机直接使用以获得性能提升

  - 虚拟存储采用 ``cache=none`` 和 ``aio=native`` 参数
  - 虚拟机采用 ``XFS`` 文件系统，参数设置 ``noop`` (或者 ``deadline`` 需要性能压测)

- 独立的两块机械硬盘 ``/dev/sdb`` 和 ``/dev/sdc`` 通过 ``virtio-blk`` 直接输出给虚拟机使用，构建 :ref:`gluster`

- 购买NVMe扩展卡并使用 :ref:`pcie_bifurcation` 将 PCIe 3.0 X16 切分成 X4X4X4X4 ，分别安装4个NVMe m.2存储设备，其中1个提供物理主机的libvirt作为虚拟机存储池，另外3个通过 VFIO 直接 pass-through 给3个虚拟机，构建 :ref:`ceph`

.. note::

   为了方便管理KVM虚拟机存储，实践采用 :ref:`libvirt_storage`

参考
=======

- `SUSE Linux Enterprise Server 15 SP1 Virtualization Best Practices <https://documentation.suse.com/sles/15-SP1/pdf/article-vt-best-practices_color_en.pdf>`_
- `SUSE Linux Enterprise Server 15 SP1 Virtualization Guide <https://documentation.suse.com/sles/15-SP1/html/SLES-all/book-virt.html>`_
- `Tuning VM Disk Performance <https://www.heiko-sieger.info/tuning-vm-disk-performance/>`_
- `QEMU Disk IO Which performs Better: Native or threads? <https://www.slideshare.net/pradeepkumarsuvce/qemu-disk-io-which-performs-better-native-or-threads>`_
