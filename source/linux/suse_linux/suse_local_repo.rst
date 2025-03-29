.. _suse_local_repo:

===================
SUSE本地软件仓库
===================

和RHEL/CentOS相似，我们可以通过构建 :ref:`centos_local_repo` 和 :ref:`centos_local_http_repo` 来加速整个网络海量服务器的软件包更新。对于维护SUSE服务器，同样有这个需求。

SUSE采用的是Red Hat相同的rpm包管理，所以实际上构建软件仓库非常相似。有两种类型软件仓库:

- 产品介质软件仓库: 产品介质软件仓库就是安装介质(CD/DVD)的副本仓库，也就是把iso镜像复制到管理服务器上，然后 ``loop-mounted`` ，或者通过NFS从一个远程服务器挂载。这种静态软件仓库不需要修改也不更新

- 更新和池软件仓库：更新和池软件仓库是由SUSE客户中心提供的，包含产品和扩展的所有更新和布丁。为了能够提供给本地局域网使用，需要从SUSE客户中心镜像这个软件仓库。由于更新仓库是定期更新的，所以必须保持和SUSE客户中心同步。对于这种方式，SUSE提供了订阅管理工具(Subscription Management Tool, SMT) 或 SUSE Manager。

复制产品介质仓库
===================

在产品介质仓库中的文件是固定不变的(从DVD复制)，不需要从远程源同步，只需要复制文件，然后通过NFS挂载产品仓库，或者直接挂载安装介质iso镜像文件就可以。

.. note::

   SUSE Linux Enterprise Server product repository必须直接从本地直接访问，不可以创建目录的符号软链接，否则会导致通过PXE启动失败。

- 产品介质必须复制到特定目录:

  - SUSE Linux Enterprise Server 12 SP4 DVD #1: 复制到 ``/srv/tftpboot/suse-12.4/x86_64/install`` 目录
  - SUSE OpenStack Cloud Crowbar 9 DVD #1: 复制到 ``/srv/tftpboot/suse-12.4/x86_64/repos/Cloud``

.. note::

   我的实践是 SLES 12 sp3 ，所以我复制目录是 ``/srv/tftpboot/suse-12.3/x86_64/install``


- 在服务器端创建目录并挂载ISO镜像进行(只读)::

   mkdir -p /srv/tftpboot/suse-12.3/x86_64/install
   mount -o loop SLE-12-SP3-Server-DVD-x86_64-GM-DVD1.iso /srv/tftpboot/suse-12.3/x86_64/install

- 服务器端安装NFS支持::

   yum install nfs-utils

- 服务器端启动NFS服务::

   systemctl enable nfs-server
   systemctl start nfs-server

- 在服务器端创建NFS输出，即编辑 ``/etc/exports`` 添加内容::

   /srv/tftpboot/suse-12.3/x86_64/install *(ro,sync,no_root_squash,no_subtree_check)

- 服务器端输出配置的NFS共享::

   exportfs -a

- 需要NFS服务器输出的软件仓库的SUSE客户机执行以下命令挂载远程服务器NFS::

   mkdir -p /srv/tftpboot/suse-12.3/x86_64/install/
   mount -t nfs 192.168.1.10:/srv/tftpboot/suse-12.3/x86_64/install/ /srv/tftpboot/suse-12.3/x86_64/install/

挂载以后，在客户机上执行 ``df -h`` 可以看到挂载的目录::

   192.168.1.10:/srv/tftpboot/suse-12.3/x86_64/install  3.6G  3.6G     0 100% /srv/tftpboot/suse-12.3/x86_64/install

.. note::

   实际上zypper可以直接添加 nfs 的仓库::

      zypper addrepo nfs://192.168.1.10:/srv/tftpboot/suse-12.3/x86_64/install

更新和池仓库
===============

Update and Pool Repositories是在管理服务器上用于设置和维护所有软件包的仓库，由SUSE Customer Center提供，包含所有更新和补丁。在部署大型应用软件，例如部署SUSE OpenStack，就需要使用最新软件版本的更新和池仓库。

- :ref:`suse_deploy_smt`

参考
=====

- `Software Repository Setup <https://documentation.suse.com/soc/9/html/suse-openstack-cloud-crowbar-all/cha-depl-repo-conf.html>`_
- `Creating a Local Repository on SUSE <https://docs.datafabric.hpe.com/61/AdvancedInstallation/CreatingLocalReposSUSE.html>`_
- `How to configure local customised repository for zypper based installation in SuSE Enterprise Linux <https://www.golinuxhub.com/2018/06/how-to-configure-local-custom-repo-zypper-sles/>`_
- `4 Installing and Setting Up an SMT Server on the Administration Server (Optional)  <https://documentation.suse.com/soc/9/html/suse-openstack-cloud-crowbar-all/app-deploy-smt.html>`_
