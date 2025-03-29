.. _k3s_squid_arch:

==================
k3s squid架构
==================

我基于 :ref:`k3s` 构建运行在 :ref:`raspberry_pi` 上的 :ref:`edge_cloud_infra` ，目标是在家庭工作室能够实现一个小型的 :ref:`devops` 环境。但是，由于GFW的阻碍，很多软件在线安装困难重重，信息也难以获取。

我基于 :ref:`squid` 构建过 :ref:`apt_proxy_arch` 来实现代理翻墙，不过以往的构建都是采用发行版squid包进行直接安装和配置。既然我已经完成了 k3s 部署，我期望除了少数底层服务采用直接物理主机构建，其他服务都容器化，kubernetes化。

所以，构建 :ref:`edge_cloud_infra` 环境下的 :ref:`squid` :

- 自制镜像: 使用 :ref:`nerdctl` 构建镜像
- 构建镜像仓库: 融合到Kubernetes集群
- 容器化运行 :ref:`squid` 并且实现冗灾(节点故障不影响运行)

参考
======

- `Deploying squid 🦑 in k3s on an RPI4B <https://dev.to/mediocredevops/deploying-squid-in-k3s-on-a-rpi4b-1nic>`_
