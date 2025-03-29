.. _alpine_install_pi_usb_boot:

========================================
树莓派环境安装Alpine Linux到USB磁盘启动
========================================

我在 :ref:`alpine_install_pi` ，系统是安装在TF卡上，性能有限。所以和 :ref:`usb_boot_ubuntu_pi_4` 相似，准备采用USB SSD磁盘来运行。我最初安装Alpine Linux是采用TF卡安装 :ref:`alpine_install_pi` ，然后通过 :ref:`alpine_pi_clone` 可以完成相同硬件环境的Alpine Linux复制安装。不过，这个方式我实践下来，只在同样使用TF卡的 :ref:`pi_3` 上成功，改到使用USB SSD存储的 :ref:`pi_4` 上启动失败。

经过再次尝试实践，我最终采用以下方法成功:

- 直接使用USB SSD磁盘的 :ref:`pi_4` 安装 Alpine Linux，采用 ``sys`` 模式安装，可以正常启动并工作
- 将上述直接安装于USB SSD的Alpine Linux系统，采用类似clone方式复制系统，是可以成功启动完成多个系统安装的

本文就是在USB磁盘启动环境中安装Alpine Linux的实践记录，完成后，再使用 :ref:`alpine_pi_usb_boot_clone` 完成后续树莓派安装。

下载和镜像
=============

下载 `alpine-rpi-3.15.0-aarch64.tar.gz <https://dl-cdn.alpinelinux.org/alpine/v3.15/releases/aarch64/alpine-rpi-3.15.0-aarch64.tar.gz>`_ 

准备
===========

- 在USB磁盘上划分2个分区:

  - 第一个分区是 ``fat32`` ，只需要 256MB ，需要设置分区为 ``boot`` 和 ``lba`` 标记
  - 第二个分区是 ``ext4`` 分区，SD卡的剩余空间

::

   fdisk /dev/sda
   
.. literalinclude:: alpine_install_pi/fdisk_sda_p.txt
   :language: bash
   :linenos:
   :caption: USB移动硬盘sys安装模式alpine linux分区

- 磁盘文件系统格式化::

   sudo partprobe
   sudo mkdosfs -F 32 /dev/sda1
   sudo mkfs.ext4 /dev/sda2

复制Alipine系统
=================

- 当前是通过USB转接，显示为 ``sda`` ::

   sudo mount /dev/sda1 /mnt

- 解压缩::

   cd /mnt
   sudo tar zxvf /home/huatai/alpine-rpi-3.15.0-aarch64.tar.gz

- 卸载挂载::

   sudo umount /mnt

.. note::

   我采用交互方式完成初始安装，所以没有使用 :ref:`alpine_install_pi` headless 方式安装

启动
=========

将部署好初始系统的USB接口的SSD移动硬盘插入 :ref:`pi_3` ，然后加电启动，可以看到只要树莓派配置了USB接口存储启动，就可以顺利启动diskless模式的Alipine Linux。这样，就可以继续完成 ``sys`` 模式安装

安装
===========

采用 ``sys`` 模式，参考 `Classic install or sys mode on Raspberry Pi <https://wiki.alpinelinux.org/wiki/Classic_install_or_sys_mode_on_Raspberry_Pi>`_

- 执行 ``setup-alpine`` 命令

  - 主机名设置( :ref:`pi_4` 上运行的第一个工作节点 ): ``x-k3s-n-2.edge.huatai.me``
  - 提示有2个网卡 ``eth0 wlan0`` ，默认选择 ``eth0`` ，并配置固定IP地址 ``192.168.7.22`` ，默认网关 ``192.168.7.200``

  - 需要注意 :ref:`alpine_pi_clock_skew` ，如果系统启动时钟不准确是无法同步的，所以一定要先执行一次::

     chronyd -q 'server pool.ntp.org -iburst'

  同步好系统时间后，再次启动 ``chronyd`` 服务来维护时钟::

     /etc/init.d/chronyd restart

  - 注意，最后要在 ``save config`` 时回答 ``none`` ; 然后还要 ``save cache`` ::

     Which disk(s) would you like to use? (or '?' for help or 'none') [none]
     Enter where to store configs ('floppy', 'sda1', 'usb' or 'none') [sda1] none
     Enter apk cache directory (or '?' or 'none') [/var/cache/apk]

- 更新软件::

   apk update

添加 sda2 分区作为系统分区
============================

由于我们已经在上文中将 ``/dev/sda2`` 格式化成 ext4 文件系统，现在开始挂载并配置成系统分区::

   mount /dev/sda2 /mnt
   export FORCE_BOOTFS=1
   # 这里添加一步创建boot目录
   mkdir /mnt/boot
   setup-disk -m sys /mnt

- 然后重新以读写模式挂载第一个分区，准备进行更新::

   mount -o remount,rw /media/sda1  # An update in the first partition is required for the next reboot.

- 清理掉旧的 ``boot`` 目录中无用文件::

   rm -f /media/sda1/boot/*  
   cd /mnt       # We are in the second partition 
   unlink boot/boot  # Drop the unused symbolink link

- 将启动镜像和 ``init ram`` 移动到正确位置::

   mv boot/* /media/sda1/boot/
   rm -Rf boot
   mkdir media/sda1   # It's the mount point for the first partition on the next reboot

- 创建软连接::

   ln -s media/sda1/boot boot

- 更新 ``/etc/fstab`` ::

   echo "/dev/sda1 /media/sda1 vfat defaults 0 0" >> etc/fstab
   sed -i '/cdrom/d' etc/fstab   # Of course, you don't have any cdrom or floppy on the Raspberry Pi
   sed -i '/floppy/d' etc/fstab
   cd /media/sda1

- 因为下次启动，需要标记root文件系统是第二个分区，所以需要修订 ``/media/sda1/cmdline.txt`` 

原先配置是::

   modules=loop,squashfs,sd-mod,usb-storage quiet console=tty1

执行以下命令添加 ``root`` 指示::

   sed -i 's/$/ root=\/dev\/sda2 /' /media/mmcblk0p1/cmdline.txt

完成修订后 ``/media/sda1/cmdline.txt`` 内容如下::

   modules=loop,squashfs,sd-mod,usb-storage quiet console=tty1 root=/dev/sda2

- 重启系统::

   reboot

这里我遇到一个问题 :ref:`alpine_pi_clock_skew`

系统简单配置
=================

- 添加huatai用户，并设置sudo::

   apk add sudo
   adduser huatai
   adduser huatai wheel
   visudo

- 作为服务器运行，关闭无线功能::

   rc-update del wpa_supplicant boot   

参考
======

- `Installing Alpine Linux on a Raspberry Pi <https://github.com/garrym/raspberry-pi-alpine>`_
- `Alpine Linux Raspberry Pi <https://wiki.alpinelinux.org/wiki/Raspberry_Pi>`_
- `Alpine Linux Raspberry Pi - Headless Installation <https://wiki.alpinelinux.org/wiki/Raspberry_Pi_-_Headless_Installation>`_ 无显示器安装
- `Alpine Linux Classic install or sys mode on Raspberry Pi <https://wiki.alpinelinux.org/wiki/Classic_install_or_sys_mode_on_Raspberry_Pi>`_ 系统安装模式 ``这篇文档是主要的参考``
- `Setting Up a Software Development Environment on Alpine Linux <https://www.overops.com/blog/my-alpine-desktop-setting-up-a-software-development-environment-on-alpine-linux/>`_-
- `Is Alpine a good alternative to Raspberry Pi OS (RPi4) when it comes to running a home server + small website (pretty much all Docker-based)? <https://www.reddit.com/r/AlpineLinux/comments/mrk03f/is_alpine_a_good_alternative_to_raspberry_pi_os/>`_
