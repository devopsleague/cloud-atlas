.. _intro_cockpit:

=====================
Cockpit简介
=====================

Cockpit是Linux服务器的系统管理平台，可以用于管理容器、存储以及配置网络和检查日志。Cockpit提供了一个WEB管理界面，非常容易使用。主流发行版集成了Cockpit，适合部署到服务器上，提供集群服务器管理。

.. note::

   另一个非常著名的WEB管理Linux服务器平台是 `webmin <https://www.webmin.com/>`_ ，提供了类似Apache, Samba, MySQL等服务的配置管理

   我理解Cockpit更适合管理底层设备(存储、网络等)，而webmin则专注于服务配置

快速起步
==========

安装完CentOS 8的标准Server(字符终端模式)后，首次启动，在终端有提示使用cockpit的方法::

   systemctl enable --now cockpit.socket

访问: https://ip-address-of-machine:9090

.. note::

   由于 :ref:`prometheus` 默认也使用 ``9090`` 端口，所以我调整 :ref:`cockpit_port_address` 为 ``9091``

很多主流的Linux发行版都内置支持了Cockpit(当前Arch Linux也内置支持了cockpit，不需要再从第三方社区仓库安装):

.. figure:: ../../../_static/linux/server/cockpit/cockpit_support_linux.png
   :scale: 75

Cockpit集成
============

- Cockpid使用系统现有的API，所以它并没有重新开发子系统或者增加新的工具层
- 默认Cockpit使用系统的普通用户登陆和权限，网络登陆也支持SSO(single-sign-on)以及其他认证技术
- ``重点`` : Cockpit自身不消耗系统资源，如果你不使用它，Cockpit甚至不在后台运行，它是通过systemd socket激活 ``按需运行`` 的。 

安装
=======

CentOS
--------

- 安装::

   sudo yum install cockpit

- 激活::

   sudo systemctl enable --now cockpit.socket

- 如果系统使用了防火墙，则通过以下方式允许访问::

   sudo firewall-cmd --permanent --zone=public --add-service=cockpit
   sudo firewall-cmd --reload

Ubuntu
---------

在 :ref:`real` 的 :ref:`priv_cloud_infra` ，我部署在 :ref:`hpe_dl360_gen9` 二手服务器上模拟云计算的底层操作系统，采用 :ref:`ubuntu_linux` 。 在这个底层物理服务器上，构建采用Cockpit来查看和管理系统。

- 安装软件包::

   sudo apt install cockpit

cockpit集成的运维功能
======================

系统升级
===========

cockpit可以在WEB界面完成系统的软件包升级，替代了传统的 ``yum upgrade`` ，并且能够开启自动更新功能




参考
========

- `Cockpit官方网站 <https://cockpit-project.org/>`_
