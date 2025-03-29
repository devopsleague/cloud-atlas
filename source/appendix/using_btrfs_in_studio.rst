.. _using_btrfs_in_studio:

==============================
在Studio中使用btrfs文件系统
==============================

.. note::

   请注意，本文是最初我构想通过btrfs的文件系统，通过划分btrfs子卷结合启用文件系统压缩来实现KVM和docker的镜像存储。一方面btrfs的卷管理可以隔离各个应用，另一方面btrfs的高级特性具有类似ZFS的共享卷实际存储空间同时具备透明压缩功能，特别适合大量的数据存储。

   Docker官方文档 `Use the BTRFS storage driver <https://docs.docker.com/storage/storagedriver/btrfs-driver/>`_ 提供的解决方案是将完整磁盘设备划归Docker管理，不能和KVM共享使用。这和我最初规划以及实践有所不同，但是我的解决方案依然是通用的方法，适合多种应用共享磁盘，只不过需要更多的人工介入，和docker完全独占磁盘设备有所差异。

   本文记录了我最初的操作实践，对于需要在相同磁盘设备上支持不同应用运行有参考价值。

在 :ref:`btrfs_in_studio` 中，我们划分了独立的磁盘设备 ``/dev/sda3`` 用于Btrfs，以下是构建KVM和docker以及用户目录共享btrfs文件系统存储的方案。

Btrfs工具
=============

安装Btrfs工具 ``btrfs-progs`` （在RHEL/CentOS中名为 ``btrfs-tools`` 软件包)::

   apt install btrfs-progs

加载btrfs模块::

   modprobe btrfs

磁盘分区
=============

使用 ``parted`` 创建 ``/dev/sda3`` 来构建btrfs::

   parted -a optimal

.. note::

   ``parted`` 提供了4k对齐优化（参考 `Create partition aligned using parted <https://unix.stackexchange.com/questions/38164/create-partition-aligned-using-parted>`_ ），使用参数 ``--align`` 或 ``-a`` 指定优化，一般可以使用 ``optimal`` 由parted自动处理对齐功能。

显示磁盘分区::

   (parted) print
   Model: ATA INTEL SSDSC2KW51 (scsi)
   Disk /dev/sda: 512GB
   Sector size (logical/physical): 512B/512B
   Partition Table: gpt
   Disk Flags: 
   
   Number  Start   End     Size    File system  Name    Flags
    1      1049kB  512MB   511MB   fat16                boot, esp
    2      512MB   51.7GB  51.2GB  ext4

增加分区3::

   mkpart primary btrfs 51.7GB 351.7GB

对新增分区命名::

   name 3 docker

增加分区4::

   mkpart primary btrfs 351.7GB 100%
   name 4 data

磁盘分区完成后，检查结果::

   (parted) print
   Model: ATA INTEL SSDSC2KW51 (scsi)
   Disk /dev/sda: 512GB
   Sector size (logical/physical): 512B/512B
   Partition Table: gpt
   Disk Flags: 
   
   Number  Start   End     Size    File system  Name    Flags
    1      1049kB  512MB   511MB   fat16                boot, esp
    2      512MB   51.7GB  51.2GB  ext4
    3      51.7GB  352GB   300GB   btrfs        docker
    4      352GB   512GB   160GB   btrfs        data


Btrfs部署
================

- 采用的btrfs非常简单的卷，单盘。首先创建根卷 ``docker`` 和 ``data``  ::

   mkfs.btrfs -L docker /dev/sda3
   mkfs.btrfs -L data /dev/sda4

- 挂载btrfs的分区

设置 ``/etc/fstab`` ::

   /dev/sda3  /var/lib/docker  btrfs  defaults,compress=zstd  0 1
   /dev/sda4  /data            btrfs  defaults,compress=zstd  0 1

然后挂载磁盘分区::

   mkdir /var/lib/docker
   mount /var/lib/docker

   mkdir /data
   mount /data

.. note::

   参考 `Btrfs Zstd Compression Benchmarks On Linux 4.14 <https://www.phoronix.com/scan.php?page=article&item=btrfs-zstd-compress&num=4>`_ 采用 ``Zstd`` 压缩方式挂载btrfs，可以获得性能和压缩率的较好平衡。

libvirt和docker数据迁移到btrfs(可选)
=======================================

.. note::

   根据Docker官方文档，在使用btrfs卷的时候，有自己独特的卷管理方式，是可以直接操作btrfs子卷的。我最初是采用本段落的手工迁移卷方法，但是从官方文档来看，似乎用官方方法直接让docker来管理后端卷比较好。所以本段落仅供参考，后续将修改成采用官方btrfs方法来管理docker卷。

   另外，我准备部署 :ref:`ceph_docker_in_studio` ，并且将Ceph存储输出给libvirt使用，作为底层存储，这样就不再使用本段落的btrfs子卷对应libvirt存储，本段落仅供参考。

- 创建btrfs的子卷，分别对应libvirt和home

创建子卷::

   btrfs subvolume create /data/libvirt
   btrfs subvolume create /data/home

检查子卷::

   btrfs subvolume list /data

显示输出::

   ID 257 gen 7 top level 5 path libvirt
   ID 258 gen 8 top level 5 path home

.. note::

   如果已经安装了libvirt软件包，在将btrfs子卷挂载前需要先停止libvirtd服务，并且需要要做数据迁移

.. note::

   详细可以参考 `使用Btrfs部署KVM <https://github.com/huataihuang/cloud-atlas-draft/blob/master/virtual/kvm/startup/in_action/deploy_kvm_using_btrfs.md>`_

- 停止libvirt服务::

   systemctl stop libvirtd
   systemctl stop libvirtd-admin.socket
   systemctl stop libvirtd-ro.socket
   systemctl stop libvirtd.socket
   systemctl stop virtlogd.socket
   systemctl stop virtlogd-admin.socket
   systemctl stop virtlockd-admin.socket
   systemctl stop virtlockd.socket
   # 停止libvirt使用的dnsmasq
   ps aux | grep dnsmasq | grep -v grep | awk '{print $2}' |  sudo xargs kill

.. note::

   在做数据迁移之前，务必确保没有任何进程在访问 ``/var/lib/libvirt`` 目录，以便能够移动和重新挂载这两个目录::

      lsof | grep libvirt

- 将源目录重命名::

   cd /var/lib
   mv libvirt libvirt.bak

注意检查目录的属主和权限::

   drwxr-xr-x  7 root          root          4.0K 2月  26 17:38 libvirt.bak

- 将btrfs子卷挂载到目标目录

创建目录::

   mkdir /var/lib/libvirt
   chmod 755 /var/lib/libvirt

修改 ``/etc/fstab``  添加::

   /dev/sda4  /var/lib/libvirt  btrfs subvol=libvirt,defaults,noatime,compress=zstd  0 1

挂载目录::

   mount /var/lib/libvirt

- 数据迁移::

   rsync -a /var/lib/libvirt.bak/ /var/lib/libvirt/

- 恢复服务::

   systemctl start libvirtd

Home目录迁移
==============

Home目录迁移比较麻烦一些，首先要退出所有使用 ``/home`` 目录的普通用户帐号，然后切换到root用户才可以进行目录挂载和数据迁移。

- 在 ``/etc/fstab`` 中添加::

   /dev/sda4  /home             btrfs subvol=home,defaults,noatime,compress=zstd  0 1

- 退出所有普通用户帐号，切换到root用户执行以下命令::

   mv /home /home.bak
   mkdir /home
   mount /home
   rsync -a /home.bak/ /home/

.. note::

   按照上述操作步骤，完整的 ``/etc/fstab`` 内容如下::

      /dev/sda3    /data    btrfs    defaults,compress=zstd   0    1
      /dev/sda3    /var/lib/libvirt   btrfs  subvol=libvirt,defaults,noatime   0   1
      /dev/sda3    /var/lib/docker    btrfs  subvol=docker,defaults,noatime    0   1

   最后挂载的 btrfs 文件系统内容如下::

      /dev/sda3       186G   17M  185G   1% /data
      /dev/sda3       186G   17M  185G   1% /var/lib/libvirt
      /dev/sda3       186G   17M  185G   1% /var/lib/docker

   可以看到btrfs的最大特点：存储容量是一个完整的"池"被各个存储卷共享，所以不需要担心某些卷预分配过多或锅烧。



.. note::

   可以重启一次操作系统验证是否都工作正常。

其他btrfs卷(可选)
===================

由于常用的用户目录会存储较多的文件，也可以考虑迁移到btrfs中。这里把 ``/home`` 目录迁移

- 创建btrfs子卷home::

   btrfs subvolume create /data/home

检查创建的子卷::

   btrfs subvolume list /data

- 将 ``/home`` 目录重命名成 ``/home.bak`` ::

    mv /home /home.bak

- 修改 ``/etc/fstab`` 添加::

   /dev/sda3    /home              btrfs  subvol=home,defaults,noatime      0   1

- 创建并挂载 ``/home`` 目录::

   mkdir /home
   mount /home

- 同步和恢复 ``/home`` 目录内容::

   rsync -a /home.bak/ /home/

参考
==========

- `ArchLinux Parted <https://wiki.archlinux.org/index.php/Parted>`_
- `ArchLinux Btrfs <https://wiki.archlinux.org/index.php/btrfs>`_
- `Create partition aligned using parted <https://unix.stackexchange.com/questions/38164/create-partition-aligned-using-parted>`_
