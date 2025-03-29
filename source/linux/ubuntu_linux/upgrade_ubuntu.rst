.. _upgrade_ubuntu:

========================
升级Ubuntu
========================

.. warning::

   如果你使用LTS版本，不建议升级到release版本，会导致系统稳定性降低。从我个人经验来看，特别是 nvidia 显卡可能带来问题很多。我个人在笔记本硬件（显卡，网卡等驱动）上投入了大量的时间和精力，但是依然难以获得稳定可靠的工作环境。最终，我决定回退到Ubuntu 18.04 LTS Server版本，并且完全采用字符终端来避免在Mac硬件上遇到的各种奇葩的问题。专注于服务器技术，专注于集群、虚拟化、容器等数据中心相关技术。

   另外，PC硬件对Linux相对友好，我准备将另外一台旧ThinkPad笔记本作为Linux桌面，体验更为激进的前沿技术。

.. note::

   本段落仅仅是因为目前尚未发布2020.4 LTS版本（预计每2年Ubuntu会发布一个LTS），当前我安装的Ubuntu 18.10是release版本，非LTS，2019年6月已经不再持续更新，所以准备切换到最新的release版本 19.04 。

   注意：建议你使用 2018.4 LTS 或者等待2020.4 LTS，作为服务器稳定可靠为主，不要轻易尝试relase版本作为生产环境。本文只是个人折腾，不足为凭。

对于不折腾难受的人来说，闲下来消磨时光的最好方式就是折腾。由于2019.4 release发布之后，Ubuntu已经停止上一个release 18.10继续升级（仅有安全补丁），所以准备切换到最新的release 19.04 ，并期待通过不断半年滚动一次，直到下一个稳定LTS版本。

版本升级准备
================

- 首先建议调整sshd服务，确保SSH会话保持，所以编辑 ``/etc/ssh/sshd_config`` 添加以下配置::

   ClientAliveInterval 60

- 然后重启SSH服务::

   sudo systemctl restart ssh

- 升级现有版本所有软件::

   sudo apt update && sudo apt dist-upgrade

- 确保已经安装了 ``update-manager-core`` ::

   sudo apt install update-manager-core

版本升级
=============

- 编辑 ``/etc/update-manager/release-upgrades`` ，将最后一行的 ``Prompt`` 值从 ``lts`` 修改成 ``normal`` ::

   Prompt=normal

- 执行升级::

   do-release-upgrade

.. note::

   不建议通过网络连接服务器升级，不过如果通过ssh登陆服务器升级，上述升级命令会再开启一个ssh端口 1022 避免升级过程网络断开。我实际升级过程是通过远程登陆后，启动screen窗口执行的升级任务。

升级过程需要按照引导交互完成。一旦完成，重启系统。

- 检查升级后版本::

   lsb_release -a

参考
=========

- `2 Ways to Upgrade Ubuntu 18.04/18.10 To Ubuntu 19.04 (GUI & Terminal) <https://www.linuxbabe.com/ubuntu/upgrade-ubuntu-18-04-18-10-to-ubuntu-19-04>`_
