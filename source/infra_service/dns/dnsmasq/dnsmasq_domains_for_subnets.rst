.. _dnsmasq_domains_for_subnets:

====================================================
配置DNSmasq对不同子网提供不同域名扩展(expand-hosts)
====================================================

我最初在构建 :ref:`edge_cloud_infra` (完全架构在 :ref:`edge_pi` )，没有独立建立DNS，而是共用了 :ref:`priv_cloud_infra` 中 :ref:`priv_dnsmasq_ics` 的DNSmasq提供的DNS解析。这个部署 :ref:`deploy_dnsmasq` 方法非常简便，通过 ``/etc/hosts`` 结合 DNSmasq 的 ``expand-hosts`` 功能(也就是自动添加 ``staging.huatai.me`` 域名后缀)，为整个虚拟化集群提供DNS解析。

不过，随着我拆分集群，将 :ref:`raspberry_pi` 设备构建成独立的 :ref:`edge_cloud` ，采用2个网段:

- ``192.168.6.x`` - :ref:`priv_cloud` : 域名 ``staging.huatai.me``
- ``192.168.7.x`` - :ref:`edge_cloud` : 域名 ``edge.huatai.me``
- ``192.168.8.x`` - :ref:`y-k8s` : 域名 ``staging.cloud-atlas.io``

这就带来一个问题:

虽然使用短域名，例如 ``x-k3s-m-1`` 直接 ``ping`` 或者 ``ssh`` 都能解析出 ``192.168.7.11`` ，但是主机的 FQDN 名字却被错误扩展成了 ``x-k3s-m-1.staging.huatai.me`` 而不是我期望的独立域名 ``egde.huatai.me`` 。

仔细阅读 ``/etc/dnsmasq.conf`` 配置文件中的注释，就可以看到:

- 所谓域名扩展 ``expand-hosts`` 是从 ``/etc/hosts`` 读取主机名到IP的解析，然后默认添加的域名后缀
- DNSmasq支持根据不同子网(particular subnet)来提供不同的 ``expand-hosts`` 域名

所以，针对我的需求，可以如下配置DNSmasq:

.. literalinclude:: deploy_dnsmasq/dnsmasq.conf
   :language: ini
   :emphasize-lines: 10,12-14

这样，DNSmasq读取 ``/etc/hosts`` 配置，例如::

   192.168.6.253  z-dev
   ...
   192.168.7.11  x-k3s-m-1
   ...
   192.168.8.116  y-k8s-m-1
   ...

就会根据IP网段，分别扩展解析成::

   192.168.6.253  z-dev.staging.huatai.me
   192.168.7.11  x-k3s-m-1.edge.huatai.me
   192.168.8.116  y-k8s-m-1.staging.cloud-atlas.io

- 验证(nslookup查询反向解析和正向解析)::

   # nslookup 192.168.6.253
   Server:         192.168.7.200
   Address:        192.168.7.200:53
   
   253.6.168.192.in-addr.arpa      name = z-dev.staging.huatai.me
   
   # nslookup 192.168.7.11
   Server:         192.168.7.200
   Address:        192.168.7.200:53
   
   11.7.168.192.in-addr.arpa       name = x-k3s-m-1.edge.huatai.me

   # nslookup 192.168.8.116
   116.8.168.192.in-addr.arpaname = y-k8s-m-1.
   
   # nslookup z-dev.staging.huatai.me
   Server:         192.168.7.200
   Address:        192.168.7.200:53
   
   Name:   z-dev.staging.huatai.me
   Address: 192.168.6.25
   
   # nslookup x-k3s-m-1.edge.huatai.me
   Server:         192.168.7.200
   Address:        192.168.7.200:53
   
   Name:   x-k3s-m-1.edge.huatai.me
   Address: 192.168.7.11

   # nslookup y-k8s-m-1.staging.cloud-atlas.io
   Server:		127.0.0.53
   Address:	127.0.0.53#53
   
   Non-authoritative answer:
   Name:	y-k8s-m-1.staging.cloud-atlas.io
   Address: 192.168.8.116

参考
======

- `archlinux doc - dnsmasq <https://wiki.archlinux.org/index.php/Dnsmasq>`_
