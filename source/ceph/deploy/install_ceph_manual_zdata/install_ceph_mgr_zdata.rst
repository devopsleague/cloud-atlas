.. _install_ceph_mgr_zdata:

=======================
安装 ceph-mgr (zdata)
=======================

手工安装Ceph的第一阶段工作 :ref:`install_ceph_mon` 完成后，需要在 ``每个`` ``ceph-mon`` 服务的运行节点，在安装一个 ``ceph-mgr`` daemon

可以设置 ``ceph-mgr`` 来使用诸如 ``ceph-ansible`` 工具。

- 创建服务的认证key::

   ceph auth get-or-create mgr.$name mon 'allow profile mgr' osd 'allow *' mds 'allow *'

.. note::

   官方文档这里写得很含糊，我参考 `iris ceph <http://www.hep.ph.ic.ac.uk/~dbauer/cloud/iris/ceph.html>`_ 大致理解 ``$name`` 指的是管理服务器名字，所以实践操作我采用了第一台服务器 ``z-b-data-1`` 名字

实际操作::

   sudo ceph auth get-or-create mgr.z-b-data-1 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -c /etc/ceph/zdata.conf

此时会提示::

   [mgr.adm]
        key = XXXXXXXXXXXXXXXX

将上述输出内容存放到集群对应名字( ``zdata`` )的 ``z-b-data-1`` 路径中，对于我的 ``zdata`` 集群，目录就是 ``/var/lib/ceph/mgr/zdata-adm/`` 。参考 :ref:`install_ceph_mon` 有同样的配置 ``ceph-mon`` 存放的密钥是 ``/var/lib/ceph/mon/zdata-z-b-data-1/keyring`` 内容类似如下::

   [mon.]
       key = XXXXXXXXXXX
       caps mon = "allow *"


所以类似 ``ceph-mgr`` 的key存放就是 ``/var/lib/ceph/mgr/zdata-z-b-data-1/keyring`` ::

   [mgr.adm]
        key = XXXXXXXXXXXXXXXX

上述命令可以合并起来(不用再手工编辑 ``/var/lib/ceph/mgr/zdata-z-b-data-1/keyring`` )::

   sudo mkdir /var/lib/ceph/mgr/zdata-z-b-data-1
   sudo ceph auth get-or-create mgr.z-b-data-1 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -c /etc/ceph/zdata.conf | sudo tee /var/lib/ceph/mgr/zdata-z-b-data-1/keyring

- 然后还需要修订文件属性::

   sudo chown ceph:ceph /var/lib/ceph/mgr/zdata-z-b-data-1/keyring
   sudo chmod 600 /var/lib/ceph/mgr/zdata-z-b-data-1/keyring

- 然后通过systemd启动::

   sudo systemctl start ceph-mgr@z-b-data-1

- 然后检查::

   sudo ceph -s -c /etc/ceph/zdata.conf

可以看到 ``ceph-mgr`` 已经注册成功::

   cluster:
     id:     53c3f770-d869-4b59-902e-d645eca7e34a
     health: HEALTH_WARN
             OSD count 0 < osd_pool_default_size 3
   
   services:
     mon: 1 daemons, quorum z-b-data-1 (age 4h)
     mgr: z-b-data-1(active, since 50s)
     osd: 0 osds: 0 up, 0 in
   
   data:
     pools:   0 pools, 0 pgs
     objects: 0 objects, 0 B
     usage:   0 B used, 0 B / 0 B avail
     pgs:

使用模块
===========

- 查看 ``ceph-mgr`` 提供了哪些模块::

   sudo ceph mgr module ls -c /etc/ceph/zdata.conf

可以看到大量提供的模块以及哪些模块已经激活。

Ceph提供了一个非常有用的模块 ``dashboard`` 方便管理存储集群。对于发行版，可以非常容易安装::

   sudo apt install ceph-mgr-dashboard

然后通过 ``sudo ceph mgr module ls -c /etc/ceph/zdata.conf`` 就会看到这个模块

- 通过 ``ceph mgr module enable <module>`` 和 ``ceph mgr module disable <module>`` 可以激活和关闭模块::

   sudo ceph mgr module enable dashboard -c /etc/ceph/zdata.conf

- 激活 ``dashboard`` 模块后，可以通过 ``ceph-mgr`` 的服务看到它::

   sudo ceph mgr services -c /etc/ceph/zdata.conf

详细配置见 :ref:`ceph_dashboard` 提供了非常丰富的管理功能，并且能够结合 :ref:`prometheus` 和 :ref:`grafana` 。

参考
=====

- `ceph-mgr administrator’s guide: MANUAL SETUP <https://docs.ceph.com/en/pacific/mgr/administrator/#mgr-administrator-guide>`_
- `iris ceph <http://www.hep.ph.ic.ac.uk/~dbauer/cloud/iris/ceph.html>`_ 这篇笔记非常实用，补充了ceph官方文档的缺失
