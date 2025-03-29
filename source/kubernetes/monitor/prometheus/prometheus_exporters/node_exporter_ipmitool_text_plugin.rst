.. _node_exporter_ipmitool_text_plugin:

===================================
Node Exporter ipmitool 文本插件
===================================

通过 :ref:`node_exporter_textfile-collector` 可以将几乎所有文本转换成Prometheus的metrics，也包括 :ref:`ipmi` 。这种方式可以帮助我们监控服务器的硬件:

准备工作
==========

.. note::

   由于 :ref:`ipmitool` 工具命令执行需要 ``root`` 权限，所以类似 :ref:`node_exporter_smartctl_text_plugin` 完成配置

- 创建一个 ``/var/lib/node_exporter/textfile_collector/`` 用于存放 ``--collector.textfile.directory`` 对应的 ``*.prom`` 文件，以便转换成metrics:

.. literalinclude:: node_exporter_textfile-collector/textfile_collector_dir
   :caption: 准备 ``/var/lib/node_exporter/textfile_collector/`` 目录

- Prometheus社区提供了 `node-exporter-textfile-collector-scripts <https://github.com/prometheus-community/node-exporter-textfile-collector-scripts>`_ ，将这些脚本下载到服务器上:

.. literalinclude:: node_exporter_textfile-collector/git_node-exporter-textfile-collector-scripts
   :caption: 下载 ``node-exporter-textfile-collector-scripts`` 到本地( ``/etc/prometheus`` )

这里将使用 ``/etc/prometheus/node-exporter-textfile-collector-scripts/ipmitool`` **脚本** 来转换服务器的 ``ipmitool sensor`` 命令输出成 :ref:`node_exporter_textfile-collector` 可处理的兼容格式

执行脚本
=========

社区推荐使用 ``sponge`` 来自动写输出，所以先切换到 ``root`` 执行以下命令生成一个案例检查(详见 :ref:`node_exporter_smartctl_text_plugin` )::

   # 注意这条命令是切换到root用户身份执行的
   sudo su -
   ipmitool sensor |  sudo -u prometheus /etc/prometheus/node-exporter-textfile-collector-scripts/ipmitool | sudo -u prometheus sponge /var/lib/node_exporter/textfile_collector/ipmitool.prom

然后检查输出内容 ``/var/lib/node_exporter/textfile_collector/ipmitool.prom`` 可以看到类似::

   # HELP node_ipmi_temperature_celsius Temperature sensor reading from ipmitool
   # TYPE node_ipmi_temperature_celsius gauge
   node_ipmi_temperature_celsius{sensor="37-Fuse"} 38.000000
   node_ipmi_temperature_celsius{sensor="15-VR P1 Mem"} 34.000000
   node_ipmi_temperature_celsius{sensor="02-CPU 1"} 40.000000
   node_ipmi_temperature_celsius{sensor="16-VR P1 Mem"} 35.000000
   ...

这个文件就是 :ref:`node_exporter_textfile-collector` 可以处理的标准格式

- 检查 ``/var/lib/node_exporter/textfile_collector/ipmitool.prom`` 内容无误之后，在 ``/etc/cron.d`` 目录下添加配置文件 ``node_exporter_textfile_collector`` :

.. literalinclude:: node_exporter_textfile-collector/node_exporter_textfile_collector
   :language: bash
   :caption: 配置定时执行 ``node_exporter`` 的 ``textfile`` Collectors
   :emphasize-lines: 3

.. note::

   上述配置详情参考 :ref:`node_exporter_smartctl_text_plugin`

配置 ``node_exporter``
==========================

按照 :ref:`node_exporter` 中 :ref:`systemd` 运行服务配置，修订 ``/etc/systemd/system/node_exporter.service`` :

.. literalinclude:: node_exporter_textfile-collector/node_exporter.service
   :caption: ``/etc/systemd/system/node_exporter.service`` 添加 textfile Collector 输出数据目录

重启 ``node_exporter`` 服务

配置 Grafana Dashboard
=========================

在 :ref:`grafana` 中 ``import`` `Grafana Dashboard 13177: IPMI for Prometheus <https://grafana.com/grafana/dashboards/13177-ipmi-for-prometheus/>`_ 

完成后Dashboard:

.. figure:: ../../../../_static/kubernetes/monitor/prometheus/prometheus_exporters/node_exporter_with_ipmitool_text_plugin.png

在 :ref:`grafana` 中 ``import`` `Grafana Dashboard 15765: IPMI Exporter <https://grafana.com/grafana/dashboards/15765-ipmi-exporter/>`_ (这个面板看起来更清晰，不过温度显示我比较喜欢 ``Time series`` 表现方式，所以我添加了一个视图)

完成后Dashboard:

.. figure:: ../../../../_static/kubernetes/monitor/prometheus/prometheus_exporters/node_exporter_with_ipmitool_text_plugin_1.png
