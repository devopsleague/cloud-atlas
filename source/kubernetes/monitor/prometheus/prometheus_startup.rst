.. _prometheus_startup:

==================
Prometheus快速起步
==================

.. note::

   :ref:`install_grafana` 采用社区 :ref:`apt` 仓库安装，更为方便。配合本文，实现 :ref:`hpe_server_monitor`

安装
====

`Prometheus官方网站提供下载 <https://prometheus.io/download/>`_ ，可以获得不同平台 （macOS, Linux, Windows）的版本:

* prometheus
* alertmanager
* 不同的exporter

提供给了二进制程序和 `Prometheus Docker镜像 <https://hub.docker.com/u/prom>`_

ARM环境安装
================

我的实践在 :ref:`arm_k8s` 的 :ref:`raspberry_pi` 主机上完成，所以安装的是 ``arm64`` 版本。

- 下载软件包并解压缩:

.. literalinclude:: prometheus_startup/arm_install_prometheus
   :caption: 在ARM环境( :ref:`raspberry_pi` )安装Prometheus

- 检查::

   prometheus --version

macOS环境安装
==============

- 在 :ref:`macos` 环境下，Prometheus可以通过 :ref:`homebrew` 安装::

   brew install prometheus

安装后提示信息::

   When run from `brew services`, `prometheus` is run from
   `prometheus_brew_services` and uses the flags in:
      /opt/homebrew/etc/prometheus.args
   
   To restart prometheus after an upgrade:
     brew services restart prometheus
   Or, if you don't want/need a background service you can just run:
     /opt/homebrew/opt/prometheus/bin/prometheus_brew_services 

.. _zcloud_host_install_prometheus:

zcloud物理主机安装Prometheus
===============================

我在 :ref:`priv_cloud_infra` 重新在一台二手服务器 :ref:`hpe_dl360_gen9` 部署大规模虚拟化集群。为了结合 :ref:`zdata_ceph` 的 :ref:`ceph_dashboard` 以及对整个基础架构进行监控，我重新部署 ``prometheus + grafana`` 到两台KVM虚拟机 ``z-b-mon-1`` 和 ``z-b-mon-2`` 。

.. note::

   实践 :ref:`hpe_server_monitor` ，在 ``zcloud`` 物理服务器上部署

- 操作系统: :ref:`ubuntu_linux` 22.04 LTS

- 准备用户账号:

.. literalinclude:: prometheus_startup/add_prometheus_user
   :language: bash
   :caption: 在操作系统中添加 prometheus 用户

用户ID < 1000则为系统用户ID，这个ID是从 999 开始递减的，对于刚安装好的Ubuntu系统， ``systemd-coredump`` 组的ID是 999，则上述两个命令创建的 ``prometheus`` 的GID和UID都是 998

- 创建配置目录和数据目录:

.. literalinclude:: prometheus_startup/mkdir_prometheus
   :language: bash
   :caption: 在操作系统中创建prometheus目录

- 下载工具::

   sudo apt update
   sudo apt -y install wget curl vim

- 下载最新prometheus二进制程序:

.. literalinclude:: prometheus_startup/ubuntu_install_prometheus
   :language: bash
   :caption: 在Ubuntu环境安装Prometheus

简餐配置(归档,不推荐)
----------------------

在解压缩的Prometheus软件包目录下有一个默认配置文件 ``prometheus.yml`` ，这个初始配置复制到 ``/etc/prometheus`` 目录下然后简单配置就可以启动::

   sudo mkdir -p /etc/prometheus
   sudo cp prometheus.yml /etc/prometheus/

默认配置启动运行在 ``localhost:9090`` 并且只监听本机::

   scrape_configs:
     - job_name: 'prometheus'
       static_configs:
       - targets: ['localhost:9090']

对于较为安全的内部局域网，可以将配置修改成监听网络IP地址::

       - targets: ['192.168.6.221:9090']

启动::

   sudo prometheus --config.file "/etc/prometheus/prometheus.yml"

如果发生异常，则可以使用 ``prometool`` 工具检查配置文件::

   promtool check config prometheus.yml

配置以及systemd运行Prometheus
===============================

- 在解压缩的Prometheus软件包目录下有配置案例以及 console libraries :

.. literalinclude:: prometheus_startup/config_prometheus
   :language: bash
   :caption: 简单配置

- 创建 Prometheus 的 :ref:`systemd` 服务管理配置文件 ``/etc/systemd/system/prometheus.service`` :

.. literalinclude:: prometheus_startup/prometheus.service
   :caption: Prometheus :ref:`systemd` 服务管理配置文件 ``/etc/systemd/system/prometheus.service``

- 启动服务:

.. literalinclude:: prometheus_startup/start_prometheus
   :caption: 启动Prometheus

.. warning::

   如果系统启用了 :ref:`cockpit` ，会遇到端口冲突导致无法启动。请先执行 :ref:`cockpit_port_address` 调整(我设置成 ``9091`` )

Docker运行Prometheus
=======================

官方提供了Prometheus Docker镜像，简单运行以下命令::

   docker run -p 9090:9090 prom/prometheus

不过，上述命令使用默认配置启动，所以需要提供自定义配置和数据存储，所以可以采用如下方法::

   docker run -p 9090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus

访问Prometheus
=================

请注意，上述简单的实践是将Prometheus启动监听在本地回环地址 ``localhost:9090`` ，所以一般外部就不能访问。这样带来一定的安全保护。

我们可以通过ssh端口转发方式实现远程访问，就是在客户端执行以下命令访问服务器 ``192.168.6.221`` 开启端口转发到服务器的回环地址 ``9090`` 端口::

   ssh -L 9090:192.168.6.221:9090 192.168.6.200

或者配置 ``.ssh/config`` 配置::

   Host zcloud
       HostName 192.168.6.200
       User admin
       LocalForward 9090 192.168.6.221:9090

然后直接执行::

   ssh zcloud

通过ssh认证登陆后，在本地通过浏览器访问 http:://127.0.0.1:9090/graph 就可以看到管理配置界面(访问 http://127.0.0.1:9090 也会重定向到 ``/graph`` 路径)

如果要访问Prometheus服务器自己的metrics，则访问 http://127.0.0.1:9090/metrics

表达式浏览
===========

在表达式(也就是graph的查询栏输入需要查询的指标)，系统会自动匹配最接近的参数

例如，我们查询 ``go_gc_duration_seconds`` 指标，输入后点击 ``Execute`` 按钮执行，就看到:

.. figure:: ../../../_static/kubernetes/monitor/prometheus/prometheus_graph_1.png
   :scale: 50

我们点击 ``Graph`` 面板，就可以看到上述 ``Table`` 表格数据在一段时间，例如1小时以内数据变化趋势:

.. figure:: ../../../_static/kubernetes/monitor/prometheus/prometheus_graph_2.png
   :scale: 50

Prometheus的PromQL提供了非常灵活的表达式语言，允许查询和聚合指标。

一个简单查询所有带有 ``quantile="0.5"`` 标签的追表，则输入::

   {quantile="0.5"}

我们可以看到有很多匹配的metrics:

.. figure:: ../../../_static/kubernetes/monitor/prometheus/prometheus_graph_3.png
   :scale: 50

配套安装exporter
===================

我的主要目标是实现 :ref:`hpe_server_monitor` ，所以继续安装以下组件:

- :ref:`node_exporter`

参考
========

- `How To Install and Configure Prometheus On a Linux Server <https://devopscube.com/install-configure-prometheus-linux/>`_
- `Install Prometheus Server on Ubuntu 22.04|20.04|18.04 <https://computingforgeeks.com/install-prometheus-server-on-debian-ubuntu-linux/>`_
- `How to Install Prometheus and Grafana on Ubuntu? <https://antonputra.com/monitoring/install-prometheus-and-grafana-on-ubuntu/>`_ 这篇非常详尽，建议参考
