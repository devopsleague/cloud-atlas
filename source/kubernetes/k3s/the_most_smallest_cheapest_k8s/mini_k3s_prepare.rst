.. _mini_k3s_prepare:

===================
微型K3s部署准备
===================

硬件: :ref:`pi_1`
=====================

为了能够发挥 :ref:`pi_1` 硬件余热(10年前购买的)，我通过 :ref:`alpine_install_pi_1` 来运行底层系统，一共使用了3个 :ref:`pi_1` :

- :ref:`alpine_install_pi_1` 只需要执行一次，另外两个树莓派通过 :ref:`alpine_pi_clone` 可以快速完成
- 按照 :ref:`edge_cloud_infra` 为3个 :ref:`pi_1` 分配IP地址及主机名，启动后，确保3台主机都能ssh登陆

  - 我发现实际上只有2台是512MB内存的B型 :ref:`pi_1` ，另外一台则只有 256MB 内存 ，所以硬件性能非常有限
  - 只能安装32位操作系统，使用 :ref:`alpine_linux` for ``armhf`` 系统，期望能够将硬件资源要求降到最低
  - :ref:`pi_1` 功率极低，无风扇，将3台设备叠加起来形成一个stack，扔到桌子底下连接到路由器的网口上组成小集群

alpine linux初始化
=======================

- :ref:`alpine_install_pi_1` 安装第一台主机，然后 :ref:`alpine_pi_clone`

最小化alpine linux安装(sys模式)，启动系统后观察可以看到，内存仅占用 42MB 

- 软件仓库激活 ``community`` ，即修改 ``/etc/apk/repositories`` :

.. literalinclude:: ../../../linux/alpine_linux/alpine_apk/repositories
   :language: bash
   :linenos:
   :caption: 激活community仓库
   :emphasize-lines: 3

- 安装工具 :ref:`screen` 并设置 ``~/.screenrc`` ::

   sudo apk add screen

:ref:`alpine_docker`
======================

执行以下步骤完成 :ref:`alpine_linux` 操作系统调整。

.. note::

   默认不需要安装 :ref:`alpine_docker` ，因为 :ref:`k3s` 包含了 :ref:`containerd` 和 :ref:`runc`

   不过 :ref:`build_k3s` 则需要先安装 :ref:`alpine_docker` ，因为所有编译过程都是在容器中完成

- 执行cgroup的fs挂载配置:

.. literalinclude:: ../../../linux/alpine_linux/alpine_docker/cgroup_fstab
   :language: bash
   :caption: 配置cgroup的fs挂载配置 /etc/fstab

- 创建 ``/etc/cgconfig.conf`` :

.. literalinclude:: ../../../linux/alpine_linux/alpine_docker/create_cgconfig.conf
   :language: bash
   :caption: 创建 /etc/cgconfig.conf

- 修订alpine linux内核启动参数，``/media/mmcblk0p1/cmdline.txt`` 添加:

.. literalinclude:: ../../../linux/alpine_linux/alpine_docker/cmdline.txt
   :language: bash
   :caption: /media/mmcblk0p1/cmdline.txt 添加内核参数

- 重启主机，然后检查 ``docker info`` 输出信息
