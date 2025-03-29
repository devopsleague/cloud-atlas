.. _containerd_centos_systemd:

======================================
Containerd环境CentOS容器运行systemd
======================================

在 :ref:`docker_systemd` 实践中，可以实现容器中以 :ref:`systemd` 为init。不过， `CentOS DockerHUB官方镜像 <https://hub.docker.com/_/centos>`_ 提供了启用 ``systemd`` 的Dockerfile。在官方 ``Dockerfile`` 基础上，还需要解决 :ref:`nerdctl` 实践时遇到的无法更新yum仓库的问题(CentOS 8已经于2021年底终止更新，需要切换到CentOS Stream分支)。

本文实践采用 :ref:`nerdctl` 完成，原因是部署 :ref:`k8s_dnsrr` 最新Kubernetes已经完全切换到 :ref:`containerd` ，摈弃了复杂的 :ref:`docker` 。

- 在本地目录下创建 ``DockerFile`` 和 用于ssh的 ``authorized_keys`` 文件: ``Dockerfile`` 内容如下:

.. literalinclude:: containerd_centos_systemd/centos-stream-8-systemd_fail
   :language: dockerfile
   :caption: CentOS Stream 8 启用systemd(失败)

- 构建镜像::

   sudo nerdctl build -t centos-stream-8-systemd .

- 启动容器::

   sudo nerdctl run -d -p 1122:22 --hostname centos-stream-8 --name centos-stream-8 local:centos-stream-8-systemd

- 启动后，检查确实看到容器启动::

   sudo nerdctl ps

输出::

   CONTAINER ID    IMAGE                                             COMMAND                   CREATED           STATUS    PORTS                   NAMES
   a2f1a44e9899    docker.io/local/centos-stream-8-systemd:latest    "/usr/sbin/init"          40 minutes ago    Up        0.0.0.0:1122->22/tcp    centos-stream-8 

但是我登陆容器内部::

   sudo nerdctl exec -it centos-stream-8 /bin/bash

发现无法启动服务::

   systemctl start sshd

输出报错::

   Failed to connect to bus: Host is down

- 通过 ``top`` 命令检查，可以看到系统启动了 ``systemd`` ::

   Tasks:   3 total,   1 running,   2 sleeping,   0 stopped,   0 zombie
   %Cpu(s):  4.6 us,  2.4 sy,  0.0 ni, 92.7 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
   MiB Mem :   3929.8 total,    186.8 free,   1074.2 used,   2668.8 buff/cache
   MiB Swap:      0.0 total,      0.0 free,      0.0 used.   2567.7 avail Mem
   
       PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
         1 root      20   0   89152   7392   6376 S   0.0   0.2   0:00.04 systemd
        44 root      20   0   14984   3784   3232 S   0.0   0.1   0:00.07 bash
        72 root      20   0   52080   4300   3696 R   0.0   0.1   0:00.00 top 

这说明systemd是启动了，但是没有传递 ``--privated`` 参数来运行容器，所以改为以下命令启动::

   sudo nerdctl run -d --privileged -p 1122:22 --hostname centos-stream-8 --name centos-stream-8 local:centos-stream-8-systemd
  
但是还是没有解决 

我感觉可能是 ``nerdctl`` 的bug，上述方式在 ``docker`` 中应该是可行的。考虑到 ``nerdctl`` 的help提到 ``-it`` 参数和 ``-d`` 参数目前是冲突的，所以我尝试用交互方式 ``-it`` 来运行，看看是什么报错::

   sudo nerdctl run -it --privileged -p 1122:22 --hostname centos-stream-8 --name centos-stream-8 local/centos-stream-8-systemd

果然出现了报错::

   Failed to mount cgroup at /sys/fs/cgroup/systemd: Operation not permitted
   [!!!!!!] Failed to mount API filesystems, freezing.
   Freezing execution.

不过，此时看容器是启动的，因为::

   sudo nerdctl ps

显示::

   CONTAINER ID    IMAGE                                             COMMAND                   CREATED          STATUS    PORTS                   NAMES
   2b75615718c8    docker.io/local/centos-stream-8-systemd:latest    "/usr/sbin/init"          3 minutes ago    Up        0.0.0.0:1122->22/tcp    centos-stream-8

看到这里的报错 ``Failed to mount cgroup at /sys/fs/cgroup/systemd: Operation not permitted`` ，而 ``Dockerfile`` 配置内容::

   VOLUME [ "/sys/fs/cgroup" ]

参考 `Docker (CentOS 7 with SYSTEMCTL) : Failed to mount tmpfs & cgroup <https://stackoverflow.com/questions/36617368/docker-centos-7-with-systemctl-failed-to-mount-tmpfs-cgroup>`_ 实际上，现在挂载卷命令应该是只读挂载，所以修订成::

   sudo nerdctl run -it --tmpfs /tmp --tmpfs /run -v /sys/fs/cgroup:/sys/fs/cgroup:ro -p 1122:22 --hostname centos-stream-8 --name centos-stream-8 local/centos-stream-8-systemd
  
此时提示进了一步::

   Detected virtualization docker.
   Detected architecture x86-64.

   Welcome to CentOS Stream 8!

   Set hostname to <centos-stream-8>.
   Failed to create /init.scope control group: Read-only file system
   Failed to allocate manager object: Read-only file system
   [!!!!!!] Failed to allocate manager object, freezing.
   Freezing execution.

注意，此时可以执行以下命令登陆到容器控制台::

   sudo nerdctl exec -it centos-stream-8 /bin/bash

并且可以看到挂载文件系统::

   # df -h
   Filesystem      Size  Used Avail Use% Mounted on
   overlay         9.4G  3.9G  5.5G  42% /
   tmpfs            64M     0   64M   0% /dev
   shm              64M     0   64M   0% /dev/shm
   tmpfs           2.0G     0  2.0G   0% /tmp
   tmpfs           2.0G  4.0K  2.0G   1% /run
   /dev/vda2       6.3G  5.1G  1.2G  81% /etc/hosts
   tmpfs           2.0G     0  2.0G   0% /proc/acpi
   tmpfs           2.0G     0  2.0G   0% /sys/firmware
   tmpfs           2.0G     0  2.0G   0% /proc/scsi

而改为::

   sudo nerdctl run -it --privileged -p 1122:22 --hostname centos-stream-8 --name centos-stream-8 local/centos-stream-8-systemd

则::

   # df -h
   Filesystem      Size  Used Avail Use% Mounted on
   overlay         9.4G  3.9G  5.5G  42% /
   tmpfs            64M     0   64M   0% /dev
   shm              64M     0   64M   0% /dev/shm
   /dev/vda2       6.3G  5.1G  1.2G  81% /sys/fs/cgroup
   tmpfs           2.0G     0  2.0G   0% /run

解决
=======

我万万没有想到，或者说我忘记了 ``nerdctl`` 实际上 **部分** clone 了 ``docker`` 命令:

在 ``nerdctl run --help`` 有如下提示::

   --privileged                  Give extended privileges to this container

我一直以为在 ``nerdctl`` 中只要使用 ``--privileged`` 就可以启动特权模式容器。但是，实际上不是的... ``nerdctl`` 的帮助文件迷惑了我。

实际上， ``nerdctl`` 启用 ``privileged`` 模式的命令参数和 ``docker`` 完全一样，必须使用 ``--privileged=true`` 而不是 ``--privileged`` 。遗憾的是，如果使用了错误参数 ``--privileged`` nerdctl 也不报任何错误。

所以，要在容器中启动 ``systemd`` ，正确的命令是::

   sudo nerdctl run -it --privileged=true -p 1122:22 --hostname centos-stream-8 --name centos-stream-8 centos-stream-8-systemd:latest

但是，CentOS 官方镜像关于 systemd 的Dockerfile也不完全正确，单纯采用::

   VOLUME [ "/sys/fs/cgroup" ]

结合 ``--privileged=true`` 依然会报错::

   Failed to mount cgroup at /sys/fs/cgroup/systemd: Operation not permitted
   [!!!!!!] Failed to mount API filesystems, freezing.
   Freezing execution.

但是将::

   VOLUME [ "/sys/fs/cgroup" ]

修改成 :ref:`docker_systemd` 方法则可以解决::

   RUN --mount=type=bind,target=/sys/fs/cgroup \
       --mount=type=bind,target=/sys/fs/fuse \
       --mount=type=tmpfs,target=/tmp \
       --mount=type=tmpfs,target=/run \
       --mount=type=tmpfs,target=/run/lock

折腾了我一天时间，终于还是回到 :ref:`docker_systemd` 最初的方式才解决。以下是修正后正确配置:

.. literalinclude:: containerd_centos_systemd/centos-stream-8-systemd
   :language: dockerfile
   :caption: CentOS Stream 8 启用systemd(正确)

参考
=======

- `[Solved] Docker Error: Failed to connect to bus: Host is down <https://programmerah.com/solved-docker-error-failed-to-connect-to-bus-host-is-down-34967/>`_ 
