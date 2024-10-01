.. _lfs_prepare:

==================
LFS准备工作
==================

.. note::

   中文社区翻译的LFS文档在2024年3月1日发布翻译版本12.1已经紧跟官方文档，非常方便实践，主要参考

.. note::

   LFS分为两个分支:

   - `Linux From Scratch 版本 12.1-systemd-中文翻译版 发布于 2024 年 3 月 1 日 <https://lfs.xry111.site/zh_CN/12.1-systemd/>`_
   - `Linux From Scratch 版本 12.1-中文翻译版 发布于 2024 年 3 月 1 日 <https://lfs.xry111.site/zh_CN/12.1/>`_

   考虑到我的底座系统是为了运行 :ref:`kvm` 和 :ref:`docker` ，没有复杂的主机服务，纯粹是运行环境，所以我目前采用 sysv 版本

.. warning::

   实际上官方文档包括中文版已经非常严谨，理论上按照文档一步步走下来必然能够完成构建。所以我不是完整摘抄文档，而是做一些笔记，以便后续能够自己重复完成，并且构建自动完成系统，定制能够充分发挥我的现有硬件性能的系统。

   请参考管官方文档为准，本章节是一些笔记和自己的补充信息

目标架构
=============

LFS的目标架构是 AMD/Intel 的 x86(32位) 和 x86_64(64位) CPU (需要修订一些指令才能适用于Power PC和 :ref:`arm` 架构的CPU)。

构建LFS至少需要一个现有的Linux系统:

- 可以是一个已经正在运行的Linux系统，现代Linux发行版(需要较新版本，对内核有和工具链有要求)，我在 :ref:`lfs_mba` 中先安装一个 :ref:`arch_linux` 作为编译基础环境(同时也是向arch linux发行版学习软件组合和配置方法)
- 也可以是某个发行版的LiveCD运行一个编译环境，例如我就使用 :ref:`fedora` 的一个LiveCD来完成，具体是通过 :ref:`create_vm` 运行Fedoraa LiveCD系统来编译构建

32位 vs. 64位
----------------

- LFS官方文档是构建纯粹的64位系统，也就是值运行64位可执行程序 
- 如果要构建 ``multi-lib`` 系统(同时支持32位和64位)则需要两次编译很多应用(考虑到现代硬件需要巨大的内存，远超4GB，所以我也按照官方文档构建 **纯粹64位系统** 以便获得一个精简的能够用于物理主机host底座的OS)

软件包选择
=============

**LFS的目标是构建一个完整且基本可用的系统** ，但是LFS并不是最小可用系统，因为LFS中有一些软件包并不是必须安装的(具体列表见 `LFS vi. 本书选择软件包的逻辑 <https://lfs.xry111.site/zh_CN/12.1/prologue/package-choices.html>`_ )

勘误和安全公告
======================

- `LFS 12.1 勘误表列出的所有修正项 <https://www.linuxfromscratch.org/lfs/errata/12.1/>`_
- `LFS 12.1 手册发布后发现的安全缺陷列表 <https://www.linuxfromscratch.org/lfs/advisories/>`_ 请根据安全公告建议在构建过程中对相关操作进行修正

  - **要将 LFS 系统实际用作桌面或服务器系统，那么即使在 LFS 系统构建完成后，也要继续关注安全公告并修复列出的所有安全缺陷**

准备宿主机
=============

宿主机(也就是用来编译构建LFS的物理主机):

- 建议至少4个CPU，内存8GB
- `LFS 宿主系统需求 <https://lfs.xry111.site/zh_CN/12.1/chapter02/hostreqs.html>`_ 列出了基本软件包版本要求(大多数发行版都能满足)，并且提供了一个检查脚本

.. literalinclude:: lfs_prepare/version-check.sh
   :caption: 执行检查主机环境
   :language: bash

- 如果和我一样使用 :ref:`lfs_vm` ，并且是 :ref:`fedora` ，可以先安装基本开发工具链:

.. literalinclude:: lfs_prepare/dnf_install_env
   :caption: 在Fedora环境安装LFS编译环境

不过，这里还有一个报错::

   ERROR: yacc is NOT Bison

参考 `How can i change yacc to bison (LFS) <https://stackoverflow.com/questions/48919466/how-can-i-change-yacc-to-bison-lfs>`_ 执行以下脚本命令:

.. literalinclude:: lfs_prepare/yacc
   :caption: 设置yacc
   :language: bash

分阶段构建LFS
=====================

LFS被设计成在一次会话中构建完成，也就是架设整个编译过程中，系统不会关闭或重启。不过，并非要求严格地一气呵成，需要注意如果重启后继续编译LFS，根据进度不同，可能需要再次进行某些操作。

分区
======

LFS对分区没有硬性规定，但是有一些建议:

- 最小的系统大约需要10G空间来存储源代码并编译所有软件包，不过作为日常Linux系统，建议使用30G空间以便后续添加功能
- 内存RAM需要8G以上，不建议使用SWAP(会影响性能)而是建议如果出现使用swap情况则扩容内存
- LFS分区建议使用成熟的文件系统，例如 :ref:`ext` 或 :ref:`xfs` ，对于后续存储数据可以使用更为高级的文件系统，如 :ref:`zfs` (这是我的建议)
- 如果启动磁盘采用GUID分区表(GPT)，则必须创建一个小的大约1MB的分区，这个分区不能格式化，但是必须被启动引导器GRUB发现，且这个分区在 ``fdisk`` 下显示为 ``BIOS Boot`` 分区，而使用 ``gdisk`` 命令则显示分区类型代号位 ``EF02``

常用分区
-------------

以下是建议使用的分区(作为参考):

- ``/boot`` 分区，存储内核以及引导信息。为了减少大磁盘可能引起的问题，建议将 ``/boot`` 分区设置为第一块磁盘的第一个分区，建议分配 512MB 或 1GB (我的建议，因为需要存储不同内核进行测试；原文建议200MB，对于我的实践经验来看这样大小的分区可能不能存储2个或以上内核)
- ``/boot/efi`` EFI系统分区，对于使用UEFI引导系统是必要的
- ``/home`` 分区，这个分区我准备后续用 :ref:`zfs` 来构建
- ``/usr`` 分区通常不用独立划分(可选，我没有使用独立分区)
- ``/opt`` 分区(可选，我没有使用独立分区)
- ``/tmp`` 分区，一般是配置瘦客户机时使用，如果系统有足够内存，可以在 ``/tmp`` 挂载一个 ``tmpfs`` 以加速访问临时文件
- ``/usr/src`` 用于存储BLFS源代码，并且可以在多个LFS系统之间共享；可以用来编译BLFS软件包，通常划分 30-50 GB

LFS假设根文件系统 ``/`` 采用 ext4 文件系统 (考虑到简化内核且根分区并没有极端的性能要求，我目前按照官方文档采用默认的 ext4 文件系统)

我的分区
----------------

- ``/dev/sda1`` vfat 格式， ``EFI System`` ，设置200M
- ``/dev/sda2`` :ref:`ext` 4 文件系统，挂载为 ``/boot`` 分区，设置512M
- ``/dev/sda3`` :ref:`ext` 4 文件系统，直接挂载 ``/`` 根分区
- 如果是物理服务器 :ref:`hpe_dl360_gen9` ，我会把大容量SSD磁盘再划分一个 ``/dev/sda4`` 作为 :ref:`zfs` 存储(虚拟机则单独配置一块虚拟磁盘来构建ZFS)

分区实践:

.. literalinclude:: lfs_prepare/parted
   :caption: 对LFS分区

完成后执行 ``parted /dev/vdb print`` 输出信息类似如下:

.. literalinclude:: lfs_prepare/parted_output
   :caption: 对LFS分区后分区信息
   :emphasize-lines: 8-10

设置环境变量
=================

- 在 ``/etc/profile`` 中配置:

.. literalinclude:: lfs_prepare/env
   :caption: ``/etc/profile`` 中添加LFS环境变量

挂载分区
============

- 创建 ``/mnt/lfs`` 目录，并挂载分区

.. literalinclude:: lfs_prepare/mount
   :caption: 挂载分区

- 完成后使用 ``df -h`` 检查

.. literalinclude:: lfs_prepare/mount_output
   :caption: 挂载分区情况

准备工作已经完成，现在可以开始准备软件包了

软件包下载
================

- 创建软件包和补丁的存放目录，按照文档，建议 ``$LFS/sources`` 目录:

.. literalinclude:: lfs_prepare/sources
   :caption: 创建 ``$LFS/sources`` 目录用于存放源代码

- 官方 `wget-list-sysv <https://www.linuxfromscratch.org/lfs/view/stable/wget-list-sysv>`_ 作为 :ref:`wget` 命令的输入来下载所有软件包和补丁:

.. literalinclude:: lfs_prepare/wget
   :caption: 下载所有 wget-list-sysv 列出的软件包和补丁

使用LFS提供的 `md5sums <https://www.linuxfromscratch.org/lfs/view/stable/md5sums>`_ 文件校验下载的软件包:

.. literalinclude:: lfs_prepare/md5sum
   :caption: 使用LFS提供的 md5sums 文件校验下载的软件包

.. note::

   对于LFS稳定版，可以从 `LFS官方镜像网站直接下载打包好的软件包文件 <https://www.linuxfromscratch.org/lfs/mirrors.html#files>`_ 

.. note::

   部分文件可能会因为上游下载源更改无法下载，需要手工处理

在LFS文件系统中创建有限目录布局
==================================

- 以root身份执行以下命令创建所需目录布局:

.. literalinclude:: lfs_prepare/mkdir
   :caption: 创建目录布局
   :language: bash

- 此外为交叉编译器准备一个专用目录，使得其和其他程序分离:

.. literalinclude:: lfs_prepare/tools
   :caption: 创建交叉编译器目录
   :language: bash

.. warning::

   LFS不适用 ``/usr/lib64`` 目录，一定要确保该目录不存在，否则可能破坏系统。需要经常检查并确认该目录不存在

添加LFS用户
=============

- 创建 ``lfs`` 用户，避免微小错误损坏或摧毁系统:

.. literalinclude:: lfs_prepare/lfs_user
   :caption: 创建lfs用户
   :language: bash

如果是使用 ``root`` 用户身份切换到 ``lfs`` 用户，则不要求 ``lfs`` 用户帐号设置过密码；其他用户如果要切换到 ``lfs`` 用户，则需要为 ``lfs`` 用户设置密码( ``passwd lfs`` )

- 将 ``lfs`` 设置为 ``$LFS`` 中所有目录的所有者，这样 ``lfs`` 就对它们拥有 **完全访问权** :

.. literalinclude:: lfs_prepare/chown_lfs
   :caption: 设置 ``$LFS`` 目录属主为 ``lfs`` 用户
   :language: bash

- 从 ``root`` 切换身份到 ``lfs`` ，后续操作以 ``lfs`` 用户身份来执行(避免出现误操作破坏系统)::

   su - lfs

配置环境
=============

为了配置良好工作环境，需要为bash创建两个新的启动脚本，以下命令以 ``lfs`` 身份执行，创建一个新的 ``.bash_profile`` :

.. literalinclude:: lfs_prepare/bash_profile
   :caption: 创建 ``lfs`` 用户的 ``.bash_profile``
   :language: bash

上述 ``.bash_profile`` 中采用 ``exec env -i .../bin/bash`` 可以确保除了 ``HOME, TERM 以及 PS1`` 之外没有任何环境变量的 shell，防止宿主机环境中有不需要和有潜在风险的环境变量进入构建环境。

- 由于新的shell实例是 **非登录shell** ，所以它不会读取和执行 ``/etc/profile`` 或者 ``.bash_profile`` 中内容，而是读取并执行 ``.bashrc`` ，所以我们现在需要创建一个 ``.bashrc`` 文件:

.. literalinclude:: lfs_prepare/lfs_bashrc
   :caption: 为 ``lfs`` 的非登录shell创建一个独立使用的 ``~/.bashrc``
   :language: bash

``.bashrc`` 中设置含义(部分摘录参考):

- ``set +h`` 关闭bash的散列功能。bash使用一个散列表维护各个可执行文件的完整路径，这样就不用每次都在 ``PATH`` 指定的目录中搜索可执行文件。这是一个非常有用的功能，但是在LFS构建中，我们希望总是使用最新安装的工具，所以关闭散列功能来强制shell在运行程序的时候总是搜索PATH
- ``umask 022`` 将用户的文件创建掩码(umask)设置为 ``022`` ，确保只有文件的所有者可以写新创建的文件和目录，但是其他任何人都可以读取、执行它们
- ``LFS=/mnt/lfs`` LFS 环境变量必须被设定为之前选择的挂载点
- ``C_ALL=POSIX`` 将 LC_ALL 设置为 “POSIX” 或者 “C”(这两种设置是等价的) 可以保证在交叉编译环境中所有命令的行为完全符合预期，而与宿主的本地化设置无关
- ``LFS_TGT=$(uname -m)-lfs-linux-gnu`` LFS_TGT变量设定了一个非默认，但与宿主系统兼容的机器描述符。该描述符被用于构建交叉编译器和交叉编译临时工具链
- ``PATH=/usr/bin`` 许多现代 Linux 发行版合并了 /bin 和 /usr/bin。在这种情况下，标准 PATH 变量应该被设定为 /usr/bin
- ``if [ ! -L /bin ]; then PATH=/bin:$PATH; fi`` 如果 /bin 不是符号链接，则它需要被添加到 PATH 变量中
- ``PATH=$LFS/tools/bin:$PATH`` 将 $LFS/tools/bin 附加在默认的 PATH 环境变量之前，一旦安装了新的程序，shell 就能立刻使用它们
- ``CONFIG_SITE=$LFS/usr/share/config.site`` 如果没有设定这个变量，configure 脚本可能会从宿主系统的 /usr/share/config.site 加载一些发行版特有的配置信息。覆盖这一默认路径，避免宿主系统可能造成的污染。 
- ``export ...`` 设定了一些变量，为了让所有子 shell 都能使用这些变量

对于多CPU core的主机，可以并行执行make，所以在 ``.bashrc`` 添加以下配置(如果不希望所有cpu core被使用，则将 ``$(nproc)`` 替换成希望使用的cpu core数量:

.. literalinclude:: lfs_prepare/makeflags
   :caption: 为make程序设置使用的cpu core数量
   :language: bash

最后确保构建临时工具所需环境就绪，强制bash shell读取刚才创建的配置文件:

.. literalinclude:: lfs_prepare/source_profile
   :caption: 强制shell读取配置文件
   :language: bash

参考
======

- `Linux From Scratch 版本 12.1-中文翻译版 发布于 2024 年 3 月 1 日 <https://lfs.xry111.site/zh_CN/12.1/>`_
