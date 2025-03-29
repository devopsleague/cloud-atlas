.. _gentoo_on_gentoo:

============================
在Gentoo上运行Gentoo(容器)
============================

在 :ref:`gentoo_docker` 部署完成后，我期望运行一个非常精简的Gentoo系统底座，而将各种开发、测试等工作搬迁到Gentoo容器中完成:

- 保持物理服务器操作系统尽可能精简稳定
- 容器之间相互隔离
- 方便不断重新实现完全一致的开发测试环境

:ref:`gentoo_image`
=====================

通过 :ref:`gentoo_image` 构建提供ssh服务并且能够安装不同开发环境的基础容器

:ref:`gentoo_docker` 之后进入容器，首先按照 :ref:`install_gentoo_on_mbp` 完成一些必要调整，也就是以下这些手工执行的命令。在完成验证之后，就可以按图索骥定制 :ref:`gentoo_image`

修改gentoo镜像地址
-----------------------

- :ref:`install_gentoo_on_mbp` 已经提示过，海外的镜像实际上几乎无法访问，所以需要将在 ``/etc/portage/make.conf`` 配置使用国内镜像网站:

.. literalinclude:: gentoo_on_gentoo/gentoo_mirrors
   :language: bash
   :caption: 配置使用阿里云镜像网站
   :emphasize-lines: 2

这步对墙内用户非常重要，否则下载几乎难以完成

.. note::

   为方便修改配置，此时可以先 ``emerge --ask app-editors/vim`` ，当然也可以在后续批量安装

选择profile
------------

- 查看系统当前使用的 ``profile`` :

.. literalinclude:: install_gentoo_on_mbp/eselect_profile_list
   :language: bash
   :caption: 检查当前使用的 ``profile``

可以看到Gentoo容器中默认的 ``profile`` 是最基本的配置:

.. literalinclude:: gentoo_on_gentoo/profile
   :caption: Gentoo的Docker中默认profile是最基本配置
   :emphasize-lines: 1

这里不用修改

更新 @world set
-----------------

更新系统的 ``@world`` set ，以便建立 ``base`` ，这样系统可以应用自构建 ``stage3`` 以来出现的任何更新或 ``USE flag`` 更改以及任何配置文件选择:

.. literalinclude:: install_gentoo_on_mbp/update_world_set
   :language: bash
   :caption: 更新 @world set

配置USE变量
-------------

.. note::

   获取CPU特性，设置对应USE变量

- 安装工具包:

.. literalinclude:: install_gentoo_on_mbp/install_cpuid2cpuflags
   :language: bash
   :caption: 安装 ``cpuid2cpuflags`` 工具包

- 执行:

.. literalinclude:: install_gentoo_on_mbp/cpuid2cpuflags
   :language: bash
   :caption: 运行 ``cpuid2cpuflags``

- 将输出结果添加到 ``package.use`` :

.. literalinclude:: install_gentoo_on_mbp/cpuid2cpuflags_output_package.use
   :language: bash
   :caption: 运行 ``cpuid2cpuflags`` 输出添加到 ``package.use``

(可选)配置 ``ACCEPT_LICENSE`` 变量
-------------------------------------

考虑到容器内部暂时不需要编译特定私有代码，所以我暂时没有修订默认接受的licenses。不过，后续如果要支持 :ref:`gentoo_nvidia` 来实现类似 :ref:`nvidia-docker` 可能还是需要接受厂商协议，可能需要执行以下命令:

.. literalinclude:: gentoo_on_gentoo/gentoo_licenses
   :language: bash
   :caption: 配置接受特定licenses
   :emphasize-lines: 2

配置时区
----------

配置时区写在 ``/etc/timezone`` 文件:

.. literalinclude:: install_gentoo_on_mbp/timezone
   :language: bash
   :caption: 配置OpenRC的timezone配置

然后重新配置 ``sys-libs/timezone-data`` 软件包，这个软件包会更新 ``/etc/localtime`` :

.. literalinclude:: install_gentoo_on_mbp/timezone-data
   :language: bash
   :caption: 重新配置 sys-libs/timezone-data

配置本地化(语言支持)
-----------------------

- 修订 ``/etc/locale.gen``

.. literalinclude:: install_gentoo_on_mbp/locale.gen
   :language: bash
   :caption: 在 /etc/locale.gen 中添加 UTF-8 支持

- 根据 ``/etc/locale.gen`` 生成所有本地化支持:

.. literalinclude:: install_gentoo_on_mbp/locale-gen
   :language: bash
   :caption: 运行 ``locale-gen`` 命令，根据 /etc/locale.gen 生成本地化支持

选择本地化
============

- 显示可支持的 ``lcoale`` :

.. literalinclude:: install_gentoo_on_mbp/eselect_locale_list
   :language: bash
   :caption: 查看当前locale

- 修订本地化为 ``en_US.utf8`` :

.. literalinclude:: install_gentoo_on_mbp/eselect_locale_set
   :language: bash
   :caption: 设置locale

- 重新加载环境:

.. literalinclude:: install_gentoo_on_mbp/env-update
   :language: bash
   :caption: 更新加载环境

安装必要工具
----------------

- 安装工具参考 :ref:`install_gentoo_on_mbp` 做了一些精简:

.. literalinclude:: gentoo_on_gentoo/option_tools
   :language: bash
   :caption: 安装可选工具

升级Gentoo
=============

在上述构建部署完成后，最后做一次仓库更新和系统整体升级:

- 更新Gentoo仓库，这时需要使用工具 ``emaint`` :

.. literalinclude:: upgrade_gentoo/emaint_short
   :language: bash
   :caption: 使用emaint更新仓库 

- 完成仓库更新之后，使用 ``emerge`` 升级整个系统:

.. literalinclude:: upgrade_gentoo/emerge_world_short
   :language: bash
   :caption: 使用emerge升级整个系统(简化参数)

构建 :ref:`gentoo_image`
==========================

以上完成一系列调整定制后，就可以参考上文步骤来定制 :ref:`gentoo_image`


