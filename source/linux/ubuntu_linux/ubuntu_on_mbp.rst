.. _ubuntu_on_mbp:

===========================
MacBook Pro上运行Ubuntu
===========================

安装
=========

现代的Linux操作系统对于硬件的支持较好，除了由于Licence限制无法直接加入私有软件需要特别处理，几乎可以平滑运行在常用的电脑设备上。不过，对于MacBook Pro支持也相对差一些，需要小心调整。

如果你参考我的 :ref:`base_os` 选择，详细的安装如下：

- :ref:`ubuntu_server`
- :ref:`ubuntu_desktop`

.. note::

   推荐阅读 `archlinux的MacBookPro11,x文档 <https://wiki.archlinux.org/index.php/MacBookPro11,x>`_

- 准备安装U盘

下载iso镜像之后，可以在macOS上转换成安装U盘镜像::

   hdiutil convert -format UDRW -o ubuntu-18.04.2-live-server-amd64.img ubuntu-18.04.2-live-server-amd64.iso

卸载U盘::

   diskutil umountDisk /Volumes/U

将镜像写入U盘::

   sudo dd if=ubuntu-18.04.2-live-server-amd64.img.dmg of=/dev/rdisk2 bs=10m

然后就可以把U盘拿到目标主机上进行安装

安装提示
------------

- MacBook Pro/Air使用了UEFI（取代了传统的BIOS），这要求硬盘上必须有一个FAT32分区，标记为EFI分区

.. note::

   在Ubuntu安装过程中，磁盘分区中需要先划分的primary分区，标记为EFI分区。这步骤必须执行，否则安装后MacBook Pro无法从没有EFI分区的磁盘启动。

   Ubuntu Desktop版本安装比较灵活，可以自定义EFI分区大小，例如，可以设置200MB。但是，如果使用Ubuntu Server版本，则会强制设置512MB（大小不可调整），而且是系统自动选择创建在 ``/dev/sda1`` 。

- Ubuntu操作系统 ``/`` 分区默认采用 ``Ext4`` 文件系统
  - 设置了 50G 空间，这样其余的空间预留给 :ref:`btrfs_in_studio` (提供灵活的卷管理以及磁盘压缩功能) 和 :ref:`ceph_docker_in_studio` (通过LVM卷模拟多块磁盘)

.. note::

   在UEFI系统中，使用XFS系统分区无法启动Ubuntu，这是因为GRUB的XFS支持的bug。变通的方法是：

   - 创建独立的 ``/boot`` 分区使用ext4文件系统，这样就避免了GRUB读取XFS，可以绕过这个问题。建议 ``/boot`` 分区至少500MB。
   - 挂在ESP到 ``/boot`` : 这个变通类似前一个方法，但是Ubuntu
     installer会拒绝这个磁盘划分方法，需要采用其他变通方法安装，然后在emergency启动系统后调整为这个配置。警告：Ubuntu有时候(但不总是)在安装新内核时候会临时创建软连接，而FAT文件系统(在ESP中使用FAT)不支持软链接，就会导致失败。对于使用Mac，可以在ESP上使用HFS+文件系统，即如果在MacBook笔记本上安装双系统启动Linux，可以直接采用macOS使用的HFS+作为ESP文件系统，这样再将这个文件系统挂载到
     ``/boot`` 就可以同时满足条件来使用XFS作为Ubuntu的rootfs。
   - 使用rEFInd作为boot loader : 实际上GRUB不是Linux唯一的EFI boot loader，可以使用 `efifs <http://efi.akeo.ie/>`_ 提供的XFS驱动，即同时安装rEFInd和efifs的XFS驱动就可以工作。

   以上解决方案参考 `Ubuntu 17.04 will not boot on UEFI system with XFS system partition <https://askubuntu.com/questions/945337/ubuntu-17-04-will-not-boot-on-uefi-system-with-xfs-system-partition>`_ ，我觉得在MacBook上可以采用方案二，在ThinkPad X220上则可以采用方案一。
  
.. note::

   实践发现和Fedora不同，采用 Btrfs 作为根文件系统（ 特别配置了 ``XFS`` 的 ``/boot`` 分区）无法启动Ubuntu。

.. note::

   MacBook Pro笔记本（物理主机）上安装的Host操作系统，我命名为 ``xcloud`` ，后续文档中会引用这个Host主机名。

安装后升级系统
==================

初次安装完操作系统，请务必先做系统更新，确保所有软件都保持最新::

   sudo apt update
   sudo apt upgrade

安装显卡驱动
===================

MacBook Pro使用的显卡是NVIDIA GeForce GT 750M Mac Edition ，默认安装的显卡驱动是开源的 ``nouveau`` ，但是开源驱动一方面性能较差，另一方面我希望在 :ref:`nvidia-docker` 以便能够支持 :ref:`build_tensorflow` 进行机器学习。所以，安装完基础操作系统之后，将开源显卡驱动替换成Nvidia官方的GPU驱动。

- 安装 ``ubuntu-drivers`` 工具包::

   sudo apt install ubuntu-drivers-common

- 列出建议驱动版本::

   ubuntu-drivers devices

输出::

   == /sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0 ==
   modalias : pci:v000010DEd00000FE9sv0000106Bsd00000130bc03sc00i00
   vendor   : NVIDIA Corporation
   model    : GK107M [GeForce GT 750M Mac Edition]
   driver   : nvidia-340 - distro non-free
   driver   : nvidia-driver-390 - distro non-free recommended
   driver   : xserver-xorg-video-nouveau - distro free builtin
   
   == /sys/devices/pci0000:00/0000:00:1c.2/0000:03:00.0 ==
   modalias : pci:v000014E4d000043A0sv0000106Bsd00000134bc02sc80i00
   vendor   : Broadcom Limited
   model    : BCM4360 802.11ac Wireless Network Adapter
   driver   : bcmwl-kernel-source - distro non-free

这里可以看到默认推荐驱动 ``nvidia-driver-390`` ，现在我们来安装::

   sudo ubuntu-drivers autoinstall

.. note::

   稳定版本 ``nvidia-driver-390`` 不支持 DPMS ，会导致 ``vbetool dpms off`` 关闭屏幕命令无效，所以，我采用添加图形驱动仓库(也是Ubuntu维护)以便安装最新图形驱动(418)::

      sudo add-apt-repository ppa:graphics-drivers/ppa
      ubuntu-drivers devices
      sudo ubuntu-drivers autoinstall

   不过，目前测试 ``418`` 版本的Nvidia驱动，依然没有实现 ``vbetool dpms off`` ，这个问题待解决。

.. _set_ubuntu_wifi:

设置无线网络
================

.. note::

   由于版权原因，默认安装的Ubuntu Server/Desktop都没有安装Broadcom无线网卡驱动

- 安装wifi驱动::

   sudo apt-get --reinstall install bcmwl-kernel-source

推荐使用NetworkManager管理无线网络，现代主流Linux发行版，不论是 CentOS 7 还是 Ubuntu 18.0.4，默认都采用了NetworkManager来管理网络。详情请参考 `NetworkManager命令行nmcli <https://github.com/huataihuang/cloud-atlas-draft/blob/master/os/linux/redhat/system_administration/network/networkmanager_nmcli.md>`_ 。

- 安装NetworkManager::

   sudo apt install network-manager

- 启动NetworkManager::

   sudo systemctl start NetworkManager
   sudo systemctl enable NetworkManager

- 显示所有可以连接的访问热点（AP）::

   sudo nmcli device wifi list

- 新增加一个wifi类型连接，连接到名为 ``HOME`` 的AP上（配置设置成名为 ``MYHOME`` ）::

   nmcli con add con-name MYHOME ifname wlp3s0 type wifi ssid HOME \
   wifi-sec.key-mgmt wpa-psk wifi-sec.psk MYPASSWORD

- 指定配置 ``MYHOME`` 进行连接::

   nmcli con up MYHOME

- 新增加一个公司所用的802.1x认证的无线网络连接，连接到名为 ``OFFICE`` 的AP上（配置设置成 ``MYOFFICE`` ）::

   nmcli con add con-name MYOFFICE ifname wlp3s0 type wifi ssid OFFICE \
   wifi-sec.key-mgmt wpa-eap 802-1x.eap peap 802-1x.phase2-auth mschapv2 \
   802-1x.identity "USERNAME" 802-1x.password "MYPASSWORD"

- 执行连接（例如，使用配置 ``MYOFFICE`` ）::

   nmcli con up MYOFFICE

- 开启或关闭wifi::

   sudo nmcli radio wifi <on|off>

.. note::

   ``nmcli`` 指令有很多缩写的方法，只要命令不重合能够区分，例如 ``connection`` 可以缩写成 ``con`` 。

.. note::

   详细请参考 `Ubuntu在MacBook Pro上WIFI <https://github.com/huataihuang/cloud-atlas-draft/blob/master/os/linux/ubuntu/install/ubuntu_on_macbook_pro_with_wifi.md>`_

.. _studio_ubuntu_setup:

配置
==============

sudo设置
-----------

- 由于长期使用Fedora/CentOS，默认自己的各平台都使用 ``uid=501,gid=20`` ，所以也修订服务器的用户账号 ``huatai`` 确保多平台一致

- 设置账号 huatai 无需密码就可以执行 root 账号命令::

   echo "%sudo   ALL=(ALL:ALL) NOPASSWD:ALL" >> /etc/sudoers

IPv6设置
-----------

- 由于不使用IPv6，所以安装后内核配置参数关闭IPv6；另外，为避免Mac硬件的BIOS默认访问macOS特性，需要在内核参数重添加 ``acpi_osi=!Darwin``

编辑 ``/etc/default/grub`` 设置::

   GRUB_CMDLINE_LINUX="ipv6.disable=1 acpi_osi=!Darwin"

然后执行 ``update-grub`` 并重启系统来关闭IPv6

电源管理设置
---------------

- 设置笔记本屏幕合上时不进入休眠（以便能够作为服务器运行），需要修改 ``/etc/systemd/logind.conf`` 配置如下::

   HandleLidSwitch=ignore
   HandleLidSwitchDocked=ignore

然后重启 ``logind`` 服务::

   systemctl restart systemd-logind

此时合上笔记本屏幕，就可以放在角落里静静工作。

- 安装cpupower工具::

   sudo apt-get install linux-tools-common linux-tools-generic

检查确认处理器默认采用 ``powersave`` 模式，以便能够降低能耗和温度，详细请参考 :ref:`reduce_laptop_overheat` ::

   sudo cpupower frequency-info

输出应该类似::

   analyzing CPU 0:
     driver: intel_pstate 
     CPUs which run at the same hardware frequency: 0
     CPUs which need to have their frequency coordinated by software: 0
     maximum transition latency:  Cannot determine or is not supported.
     hardware limits: 800 MHz - 3.50 GHz
     available cpufreq governors: performance powersave
     current policy: frequency should be within 800 MHz and 3.50 GHz.
                     The governor "powersave" may decide which speed to use
                     within this range.
     current CPU frequency: Unable to call hardware
     current CPU frequency: 1.81 GHz (asserted by call to kernel)
     boost state support:
       Supported: yes
       Active: yes

.. note::

   请确保系统采用了 ``intel_pstate`` 内核驱动来管理处理器主频动态调整，并且默认设置 ``powersave`` 。

关闭显示器节约电能，设置 ``~/bin/screenoff`` 脚本::

   sudo sh -c 'vbetool dpms off; read ans; vbetool dpms on'

在合上屏幕之前，先执行这个脚本命令关闭屏幕，这样也能降低笔记本温度。

.. note::

   测试发现 ``vbetool dpms off`` 可以关闭 :ref:`ubuntu_on_thinkpad_x220` 的显示屏幕，但是对于MacBook笔记本运行的Ubuntu无效。

   参考 `Turning off screen <https://unix.stackexchange.com/questions/433452/turning-off-screen>`_ 似乎可以通过设置::

      echo 0 | sudo tee /sys/class/backlight/acpi_video0/brightness

   来修改亮度。但是我测试没有效果。这个问题似乎和Nvidia显卡驱动有关 `DPMS not working with DisplayPort monitor <https://devtalk.nvidia.com/default/topic/968059/linux/dpms-not-working-with-displayport-monitor/1>`_ 例如，Nvidia驱动 375.10 不支持DPMS，

   不过，根据最新反馈 `Nvidia v415.22驱动+Kernel 4.19.8可能支持 HardDPMS <https://devtalk.nvidia.com/default/topic/968059/linux/dpms-not-working-with-displayport-monitor/5>`_ ，或许可以尝试一下。

ssh设置
------------

- 将自己的密钥对复制到 xcloud 服务器上，确保后续能够免密码登陆各个虚拟机

- 由于我需要在 xcloud 这台物理主机上作为管理堡垒机来管理整个模拟的计算机集群虚拟机，虽然可以通过密钥登陆，但是每次输入密钥保护密码是非常麻烦的。所以按照之前工作经验 `ssh密钥认证 <https://github.com/huataihuang/cloud-atlas-draft/blob/master/service/ssh/ssh_key.md>`_ 中的 ssh-agent 方法，在个人目录下 ``~/.profile`` 中添加::

   if [ -f ~/.agent.env ]; then
     . ~/.agent.env -s > /dev/null
   
     if ! kill -0 $SSH_AGENT_PID > /dev/null 2>&1; then
       echo
       echo "Stale agent file found.  Spawning new agent..."
       eval `ssh-agent -s | tee ~/.agent.env`
       ssh-add
     fi
   else
     echo "Starting ssh-agent..."
     eval `ssh-agent -s | tee ~/.agent.env`
     ssh-add
   fi

这样首次登陆系统会自动加载 ``ssh-agent`` ，只需要输入一次密钥保护密码就可以免密码登陆需要维护的集群虚拟机。

- SSH提供了一种 ``multiplexing`` （多路传输）机制，可以重用已有的TCP连接，在维护系统时非常方便，加速了ssh登陆，推荐在 ``xcloud`` Host主机上配置客户端，方便登陆各个虚拟机操作。即设置 ``~/.ssh/config`` 配置如下::

   Host *
       ServerAliveInterval 60
       ControlMaster auto
       ControlPath ~/.ssh/%h-%p-%r
       ControlPersist yes

   Host ubuntu18-04
       HostName 192.168.122.2
       User huatai

.. note::

   详细请参考 `ssh多路传输multiplexing加速 <https://github.com/huataihuang/cloud-atlas-draft/blob/master/service/ssh/multiplexing.md>`_

screen设置
-------------

- 在服务器维护过程中，经常需要启用 ``screen`` 以便在服务器端保持工作状态，建议在 ``xcloud`` 主机上添加 ``~/.screenrc`` :

.. literalinclude:: screenrc
   :language: bash
   :linenos:

.. note::

   详细使用screen工具来帮助维护服务器，请参考 `screen <https://github.com/huataihuang/cloud-atlas-draft/blob/master/develop/shell/utilities/screen.md>`_
