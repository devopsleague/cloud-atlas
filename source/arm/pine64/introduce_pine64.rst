.. _introduce_pine64:

===============
Pine64简介
===============

`Pine64 <https://www.pine64.org>`_ 的目标是提供64位ARM设备，这是一个非常迷人的目标。

因为我们的周围遍布ARM设备(例如iPhone和Android手机)，然而很不幸，绝大多数都是商业闭源设备。即使是基于Linux的Android，出于商业目的依然有不能开源的服务或者驱动。这使得移动设备的可Hack/Geek的乐趣少了很多。

Pine64通过社区驱动方式来设计和开发基于ARM的移动设备，目前已经推出了性价比较高的 $99 :ref:`pinebook` ，并且即将在2020年初推出 $199 的 PineBook Pro。2020年。这可以ARM芯片的笔记本哦!

2020年初，Pine64还推出了 :ref:`pinephone` 开发者版本 ``Brave Heart Edition`` ，我觉得这是目前最有可能成功的开源Linux手机。

.. note::

   Pine64目标是ARM64开源设备和软件集成， :ref:`postmarketos` 则是在10年的旧手机设备运行Linux开源系统。

Pine64理念
=============

``None of us is as smart as all of us ~ Ken H. Blanchard`` (团队的任何人都不如整个团队聪明能干)

Pine64的理念是PINE64是一个社区平台。这个观点的简化理解就是经常在网上所说的"PINE64负责硬件，而社区负责软件"。但这个描述并不精确，只是一个过于简化的说法。实际上PINE64社区驱动的并不限于单向的依赖社区或者所支持的合作软件项目，它也意味着社区实际上在不断打磨改进设备，就像社交平台。这个目标使得ARM64设备深深吸引你并让你愿意参与其中。

在设备发售之前，你就可以参与提交产品功能需求，建议修改以及改进文档。在设备进入工厂生产线之前，硬件开发(不管是成功还是失败)也是完全开放的。可以通过论坛、IRC，记录以及在线交谈记录，不论是硬件还是软件开发，你都可以贡献你的时间和技能。

软件生态
=========

Linux发行版
-----------------

- `Arch Linux ARM on Mobile <https://github.com/dreemurrs-embedded/Pine64-Arch>`_ 聚焦于在移动设备中运行 :ref:`arch_linux` ARM版本，当前仅支持 :ref:`pinephone` 和 PineTab。我注意到这个Arch Linux ARM on Mobile项目采用了 :ref:`waydroid` 作为Andorid兼容层(可以可以使用 :ref:`anbox` )，提供了在ARM设备的Linux标准发行版运行Android程序的思路。

参考
========

- `PINE64 Philosophy <https://www.pine64.org/philosophy/>`_
