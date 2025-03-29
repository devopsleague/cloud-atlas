.. _etcd:

======================
etcd - 分布式kv存储
======================

etcd是Kubernetes后端存储所有集群数据的高可用key value存储。由于etcd是Kubernetes集群的核心数据存储，所以数据备份是非常关键的运维工作。

本章节专注于etcd的原理、性能、优化、故障排查；:ref:`deploy_etcd` 则专注于etcd集群安装

.. note::

   详细操作可以参考 `etcd docs <https://etcd.io/docs/>`_ 以及：

   - `Operating etcd clusters for Kubernetes <https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/>`_
   - `etcd Operations guide <https://etcd.io/docs/v3.4.0/op-guide/>`_

.. toctree::
   :maxdepth: 1

   intro_etcd.rst 
   auth/index
   config/index
   maintain/index
   recovery/index
