.. _macos_homebrew_nginx:

==============================
macOS中使用Homebrew部署nginx
==============================

在 :ref:`macos` 中通过 :ref:`homebrew` 构建 :ref:`macos_studio` ，在 ``brew`` 安装完 ``nginx`` 软件包之后，通过简单配置就可以开始 :ref:`sphinx_doc` 笔记:

默认的 :ref:`homebrew` 安装 ``nginx`` 的配置目录是 ``/opt/homebrew/etc/nginx`` ，默认WEB目录是 ``/opt/homebrew/var/www`` ，默认端口 ``8080``

- 修改 ``/opt/homebrew/etc/nginx/nginx.conf`` :

.. literalinclude:: macos_homebrew_nginx/nginx.conf
   :language: bash
   :caption: 简单修订 /opt/homebrew/etc/nginx/nginx.conf 启用 Sphinx doc 访问
   :emphasize-lines: 13,16,17

.. note::

   我安装部署 :ref:`mobile_cloud_arm` ( :ref:`apple_silicon_m1_pro` MacBook Pro ) 和 在Intel的 MacBook Pro笔记本上部署 :ref:`macos_studio` ，发现homebrew安装目录不同( 基础目录是 ``/usr/local`` ，所以得到的 ``nginx.conf`` 是 ``/usr/local/etc/ngnix/ngnix.conf`` )，并且对应的NGINX的配置文件目录以及具体配置也有差异。

   所以要获取NGINX默认配置，请使用 :ref:`get_nginx_default_config` 方法

- :ref:`homebrew` 提供了惯例服务启动的功能，所以可以通过以下命令重启 NGINX :

.. literalinclude:: macos_homebrew_nginx/brew_restart_nginx
   :language: bash
   :caption: 使用brew重启nginx

.. note::

   ``brew services run XXXX`` 是直接运行服务，但是不会注册成登陆时自动运行；但是 ``brew services start XXXX`` 则会在登陆或启动时自动运行服务

.. note::

   nginx实际运行进程的pid就是用户自己的id，例如我的用户名是 ``huatai`` ，则启动nginx服务之后可以看到是 ``huatai`` pid，就能够读取自己用户目录下的文件; ``nginx.conf`` 配置中也提供了 ``user nobody;`` 选项，但是默认没有激活


