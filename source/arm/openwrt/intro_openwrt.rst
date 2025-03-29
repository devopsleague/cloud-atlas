.. _intro_openwrt:

=================
OpenWrt简介
=================

OpenWrt是面向嵌入设备的Linux操作系统，通过构建一个单一静态的firmware，OpenWrt提供了支持包管理的完整读写的文件系统。这样你可以自由选择安装软件和配置，可以定制设备来满足运行任何程序。对于开发者而言，OpenWrt就是一个构建应用程序的框架，无需重新构建完整firmware；对于最终用户就是一个完全可定制的设备。

`OpenWrt Table of Hardware <https://openwrt.org/toh/start>`_ 提供了完整的硬件支持列表。

.. note::

   我的目标是采用OpenWrt构建云计算的边缘计算，实现所谓的"雾计算"，即在嵌入式系统中构建云计算:

   - 使用 :ref:`goflex_home` 构建NAS存储+ :ref:`gluster`
   - 使用多网口路由器构建自己的SDN交换机
   - 构建基于OpenWRT的 :ref:`kubernetes` 集群

     - 参考 `discordianfish/k3s-openwrt <https://github.com/discordianfish/k3s-openwrt>`_ 和 `Running Kubernetes on OpenWrt <https://5pi.de/2019/05/10/k8s-on-openwrt/>`_
