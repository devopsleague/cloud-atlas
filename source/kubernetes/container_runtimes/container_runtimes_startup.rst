.. _container_runtimes_startup:

====================
容器运行环境快速起步
====================

在Kubernetes的Pod中运行容器，系统需要具备一个container runtime。并且Kubernetes支持多种contrainer runtime:

- Docker
- CRI-O
- Containerd
- 其他CRI runtimes: frakti

默认情况下，Kubernetes使用 ``Container Runtime Interface(CRI)`` 来和选择的容器运行时交互。如果没有指定运行时，则 :ref:`kubeadm` 自动尝试检测安装的容器运行

.. note::

   Docker Engine没有直接实现CRI，所以在Kubernetes中使用 :ref:`docker` 需要一个附加的服务 ``cri-dockerd`` 来使用传统的Docker Engine。

Kubernetes通过以下约定来检测支持的endpoints:

.. csv-table:: kubernetes检测runtime运行时endpoints
      :file: container_runtimes_startup/runtime_socket.csv
      :widths: 50, 50
      :header-rows: 1

Cgroup驱动
===========

在使用systemd作为init的Linux发行版中，init进程生成和控制了一个根控制组(root control group)并且systemd作为一个cgroup manager。systemd已经紧密结合了cgroups并且为每个进程分配cgroup。systemd可以配置container runtime 和 kubelet 来使用 ``cgroupfs`` 。注意，如果独立于systemd来使用 ``cgroupfs`` 意味着系统有两套不同的cgruop manager。 

控制组(control groups)用于进程使用的资源。一个单一的cgroup manager可以简化资源分配的视图并且天然就具备可用及使用资源的一致性视图。所以如果我们使用两个cgroup manager就不得不面对这些资源的两种视图：需要检查通过 ``cgroupfs`` 配置的kubelet和Docker的配置的字段，同时又要检查节点上剩下的由 ``systemd`` 配置的进程。这种情况会导致系统不稳定。

要修改容器运行时环境和kubelet使用 ``systemd`` 作为cgroup驱动以便系统更佳稳定。请注意下文中设置Docker的配置 ``native.cgroupdriver=systemd`` 选项。

.. warning::

   请不要修改已经加入集群节点的cgroup驱动：如果kubelet已经使用了一种cgroup driver创建了Pod，修改容器运行时环境使用另一种cgroup driver会导致对已经存在Pods重新创建PodSandbox时出错，甚至重新启动kubelet也不能解决这个错误。如果要修改节点cgroup driver，建议先从集群移除节点，然后修改cgroup driver再重新加入集群。

Cgroup v2
------------

- 与 :ref:`cgroup_v1` 不同的是， :ref:`cgroup_v2` 只有一个层次结构，而不是每个控制器有一个不同的层次结构。
- 尽管内核支持混合配置，即其中一些控制器由 cgroup v1 管理，另一些由 cgroup v2 管理， Kubernetes 仅支持使用同一 cgroup 版本来管理所有控制器。
- 如果 systemd 默认不使用 cgroup v2，你可以通过在内核命令行中添加 ``systemd.unified_cgroup_hierarchy=1`` 来配置系统去使用它。
- 切换到 cgroup v2 时，用户体验不应有任何明显差异， 除非用户直接在节点上或在容器内访问 cgroup 文件系统。 为了使用它，CRI 运行时也必须支持 cgroup v2。


Ubuntu 20.10 才开始默认使用cgroup v2，原因是snap系统尚未支持。不过，对于我不使用snap，且想要实践最新的 :ref:`cgroup_v2` 来更精细化管控资源，所以 :ref:`enable_cgroup_v2_ubuntu_20.04` 。以下修订在每个节点上完成:

- 修改 ``/etc/default/grub`` 配置在 ``GRUB_CMDLINE_LINUX`` 添加参数::

   systemd.unified_cgroup_hierarchy=1

- 修订grub并重启::

   sudo update-grub
   sudo reboot

容器运行时(container runtimes)
=================================

Kubernetes不需要(现在甚至不太支持)完整的Docker，只需要提供 Container Runtime Interface(CRI)的容器运行时(runtime)

:ref:`containerd`
-------------------

完整步骤待整理，见 :ref:`install_containerd_official_binaries`

Docker(适用于K8s 1.24版本之前)
=================================

.. warning::

   从 Kubernetes 1.24 开始 :strike:`已经移除了Docker支持，默认不再需要使用Docker` (依然可以使用 ``Docker Engine`` ，但是需要安装Mirantis公司(这家公司收购了企业版Docker Engine)提供的 ``cri-dockerd`` ，建议直接采取原生CRI的 :ref:`container_runtimes` ，如 :ref:`containerd` 。

.. note::

   :strike:`我的个人实践目前仅限于Docker作为Kubernetes的runtime，其他容器运行环境可能会在某些需要的时候尝试实践。` 我之前实践是采用Docker Engine作为runtime，但是从 Kubernetes 1.24 开始，我改为直接使用 :ref:`containerd` 部署。

Ubuntu 16.04
----------------

::

   # Install Docker CE
   ## Set up the repository:
   ### Install packages to allow apt to use a repository over HTTPS
   apt-get update && apt-get install apt-transport-https ca-certificates curl software-properties-common
   
   ### Add Docker’s official GPG key
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
   
   ### Add Docker apt repository.
   add-apt-repository \
     "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
     $(lsb_release -cs) \
     stable"
   
   ## Install Docker CE.
   apt-get update && apt-get install docker-ce=18.06.2~ce~3-0~ubuntu
   
   # Setup daemon.
   cat > /etc/docker/daemon.json <<EOF
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2"
   }
   EOF
   
   mkdir -p /etc/systemd/system/docker.service.d
   
   # Restart docker.
   systemctl daemon-reload
   systemctl restart docker   

Ubuntu 20.04
-----------------

在 :ref:`priv_cloud_infra` 部署中采用 :ref:`ubuntu_linux` 20.04 LTS版本，采用了发行版提供的 ``docker.io`` 软件包部署::

   sudo apt update && sudo apt upgrade
   sudo apt install docker.io -y

同样需要调整daemon::

   # Setup daemon.
   cat > /etc/docker/daemon.json <<EOF
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2"
   }
   EOF
   
   mkdir -p /etc/systemd/system/docker.service.d
   
   # Restart docker.
   systemctl daemon-reload
   systemctl restart docker   

.. note::

   实际部署我调整为采用 :ref:`docker_btrfs_driver` ，见 :ref:`prepare_z-k8s`

CentOS/RHEL 7.4+
-------------------

::

   # Install Docker CE
   ## Set up the repository
   ### Install required packages.
   yum install yum-utils device-mapper-persistent-data lvm2
   
   ### Add Docker repository.
   yum-config-manager \
     --add-repo \
     https://download.docker.com/linux/centos/docker-ce.repo
   
   ## Install Docker CE.
   # 我部署的测试环境采用默认docker-ce未指定版本
   yum update && yum install docker-ce-18.06.2.ce
   
   ## Create /etc/docker directory.
   mkdir /etc/docker
   
   # Setup daemon.
   cat > /etc/docker/daemon.json <<EOF
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2",
     "storage-opts": [
       "overlay2.override_kernel_check=true"
     ]
   }
   EOF
   
   mkdir -p /etc/systemd/system/docker.service.d
   
   # Restart Docker
   systemctl daemon-reload
   systemctl restart docker

   # Enable Docker
   systemctl enable docker

.. note::

   强烈推荐采用pssh工具来并发执行安装，例如将上述所有主机IP地址保存为 ``kube`` 文件，然后执行以下命令批量安装更新::

      pssh -ih kube 'sudo yum install yum-utils device-mapper-persistent-data lvm2 -y'
      pssh -ih kube 'sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo'
      pssh -ih kube 'sudo yum update && sudo yum install docker-ce -y'
      pssh -ih kube 'sudo mkdir /etc/docker'

      cat > daemon.json <<EOF
      {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
          "max-size": "100m"
        },
        "storage-driver": "overlay2",
        "storage-opts": [
          "overlay2.override_kernel_check=true"
        ]
      }
      EOF

      pscp.pssh -h kube daemon.json /tmp/daemon.json
      pssh -ih kube 'sudo mv /tmp/daemon.json /etc/docker/daemon.json'

      pssh -ih kube 'sudo mkdir -p /etc/systemd/system/docker.service.d'

      pssh -ih kube 'sudo systemctl daemon-reload'
      pssh -ih kube 'sudo systemctl restart docker'
      pssh -ih kube 'sudo systemctl enable docker'

CentOS 8.2
==============

近期 :ref:`upgrade_centos_7_to_8` 再 :ref:`install_docker_centos8` ，过程会比较折腾，但是可以在最新的内核和发行版上部署完整的Kubernetes环境。

我将补充CentSO 8平台部署Kubernetes和之前在CentOS 7上部署 :ref:`ha_k8s` 的不同。

::

   # 添加docker-ce仓库
   sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

   # 检查所有仓库列表
   sudo dnf repolist -v
   # 将所有docker-ce版本列出
   dnf list docker-ce --showduplicates | sort -r

   # 安装containerd.io - 2021年初验证，Docker已经直接提供了CentOS 8版本 containerd.io ，不需要强制指定安装
   # sudo dnf install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.13-3.2.el7.x86_64.rpm

   # 安装docker-ce
   sudo dnf install docker-ce

参考
=======

- `安装 kubeadm <https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>`_
