.. _chromium_os:

==================
Chromium OS
==================

Chromium OS基于 :ref:`gentoo_linux` 开发，结合了开源的Chromium浏览器，实现了轻量级面向云应用的Linux。

Chromium OS/Chrome OS完全基于因特网运行的轻量级客户端，在网络受限的墙内使用非常痛苦。不过，Chormium OS通过crostini这样的容器技术，可以在PixelBook/ChromeBook上运行Linux应用程序，为本地应用打开了一扇门。

我个人更看好基于Android的系统来融合移动设备操作系统，毕竟Android市场海量的应用目前还是WEB无法超越的。何况，在国内的生态环境，很多工作应用如果不能使用Android，则即使引入Linux容器也无法满足需求。

对于技术爱好者，我觉得更好的方式是学习 Chromium OS ，通过 :ref:`lfs` 来构建自己的操作系统，构建精简的轻量级操作系统，同时构建AnBox来运行Android应用。

.. _chromium_os_arch:

Chromium OS 架构
===================

.. note::

   基于Chromium OS开发定制的Linux操作系统有 `Flint OS <https://flintos.io/>`_ ，以及在国内再定制的 `FydeOS <https://fydeos.com/>`_ (FydeOS提供针对FydeOS定制的硬件，可以视为ChromeBook在国内的翻版)。

   `Flint OS于2018年3月被Neverware收购 <https://www.neverware.com/pressrelease-03-06-2018>`_ ，合并到Neverware的CloudReady产品。Neverware在2020年11月又被Google收购，所以目前这个产品应该是融入Chrome OS作为云计算中远程运行的桌面系统产品。

Chromium OS主要由3部分组成:

- 基于Chromium的浏览器和窗口管理器
- 系统级别软件和用户级别服务：内核，驱动，连接管理器等等
- Firmware

.. figure:: ../../_static/linux/chromium_os/chromium_os_arch.dms
   :scale: 75

Chromium OS关键组件
======================

Firmware
-----------

firmware可以说是Chromium OS的安全核心，因为现代操作系统的硬件底层都是通过firmware来完成的，firmware的安全性是整个系统的基石。通过加固firmware可以使得操作系统更安全，也更快速。

Chromium OS对firmware的改进主要是削减不必要的组件以降低被攻击面，并且支持启动过程的每一步验证。Chromium OS也支持系统恢复进入firmware。为了避免复杂性带来的安全漏洞，Chromium OS的firmware舍弃了很多旧硬件支持，例如，不支持软盘。

Chromium OS firmware可以实现以下功能:

- 系统恢复: recovery firmware可以重新安装操作系统，即使操作系统损坏也可以立即恢复。这个功能有点类似mac电脑的recovery，并且macOS还支持通过Internet进行recovery
- 验证启动: 每次系统启动，Chromium OS都会校验firmware，内核以及系统镜像，以防止恶意修改
- 快速启动: Chromium OS移除了很多传统PC firmware向后兼容不得不支持大量硬件的特性，所以通过精简加快了启动(当然也就限制了硬件兼容)

.. figure:: ../../_static/linux/chromium_os/chromium_os_firmware.dms
   :scale: 75

系统级别和用户级别软件
=======================

在Chromium OS中，提供了 Linux内核，驱动和用户级别服务。

Chromium OS使用Upstart来管理用户级别的服务，这种Upstart进程管理器提供了并行的，应用任务crashi自动重启以及推迟服务来加快启动。

Chromium OS使用了以下开源组件:

- ``D-Bus`` : 浏览器使用D-Bus来和系统其他组件进行交互，包括店址检测和网络探测
- ``连接管理器`` : 提供了一个共用的API来和网络设备交互，提供DNS代理，以及管理网络服务，如3G，无线和以太网
- ``WPA Supplicant`` : 连接无线网络
- ``Autoupdate`` : 使用自动更新服务可以静默安装新的系统镜像
- ``电源管理`` : (在Intel硬件上使用ACPI)管理电能，例如合上笔记本屏幕或按下电源键时休眠
- ``标准Linux服务`` : 如NTP, syslog 和 cron

Chromium和窗口管理器
======================

窗口管理器负责处理用户和多个客户端窗口交互，有点类似X window manager，通过控制窗口的摆放，输入窗口聚焦，以及快捷键操作。结合ICCCM(Inter-Client Communication Conventions Manual, 客户间通讯约定手册) 和 EWHM (Extended Window Manager Hints, 扩展窗口管理器提示) 标准来实现客户端和窗口管理器通讯。

窗口管理器还使用了 XComposite 扩展来重定向客户端窗口到 offscreen pixmaps，这样可以绘出一个最终的合成图形以及其中包含的内容。这样是的窗口可以缩放和混合。窗口管理器包含了一个compositor来通过OpenGL或OpenGL|ES实现窗口动画和渲染。

.. figure:: ../../_static/linux/chromium_os/chromium_os_wm.dms
   :scale: 75

参考
======

- `The Chromium Projects - Software Architecture <http://www.chromium.org/chromium-os/chromiumos-design-docs/software-architecture>`_
