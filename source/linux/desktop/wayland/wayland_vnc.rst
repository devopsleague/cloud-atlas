.. _wayland_vnc:

=================
Wayland环境VNC
=================

VNC服务器
===========

``wayvnc`` 是 ``wlroots-based`` Wayland compositors 的VNC server，通过附加到一个运行的Wayland会话，创建虚拟输入设备，以及通过RFB协议输出一个单一显示，来实现VNC。不过， ``wayvnc`` 不支持 Gnome 和 KDE 。好在我目前主要使用 :ref:`sway` ，所以使用 ``wayvnc`` 正好。

安装wayvnc
---------------

- 我在 :ref:`asahi_linux` 上使用 :ref:`sway` ，安装方法和 :ref:`arch_linux` 相同::

   pacman -S wayvnc

启动
--------

有2种方式启动wayvnc服务器:

无头模式(headless)
~~~~~~~~~~~~~~~~~~~~

- 无头模式(headless): 在终端启动:

.. literalinclude:: wayland_vnc/wayvnc_headless_server
   :caption: 无头模式(headless)启动sway的VNC服务器

这里我遇到一个报错(以普通用户huatai启动)::

   libEGL warning: MESA-LOADER: failed to open vgem: /usr/lib/dri/vgem_dri.so: cannot open shared object file: No such file or directory (search paths /usr/lib/dri, suffix _dri)

   00:00:00.003 [wlr] [EGL] command: eglInitialize, error: EGL_NOT_INITIALIZED (0x3001), message: "DRI2: failed to load driver"
   libEGL warning: NEEDS EXTENSION: falling back to kms_swrast
   MESA-LOADER: failed to open vgem: /usr/lib/dri/vgem_dri.so: cannot open shared object file: No such file or directory (search paths /usr/lib/dri, suffix _dri)
   failed to load driver: vgem
   KMS: DRM_IOCTL_MODE_CREATE_DUMB failed: Permission denied
   00:00:04.558 [wlr] [render/allocator/gbm.c:114] gbm_bo_create failed
   00:00:04.558 [wlr] [render/swapchain.c:109] Failed to allocate buffer

上述报错参考 `arch linux: OpenGL Switching between drivers Mesa <https://wiki.archlinux.org/title/OpenGL#Mesa>`_ 以及 `The Mesa 3D Graphics Library: Enviroment Variables <https://docs.mesa3d.org/envvars.html>`_ :

Mesa可以通过环境变量修改使用不同的驱动，默认Mesa在 ``/lib/dri`` 目录下搜索驱动，驱动名类似 ``driver_dri.so`` ，如果Mesa找不到特定驱动，会fall back到 ``llvmpipe`` 。 可以在环境变量中配置OpenGL software rasterizer::

   LIBGL_ALWAYS_SOFTWARE=true
   GALLIUM_DRIVER=driver

这里 ``driver`` 可以是 ``softpipe`` , ``llvmpipe`` 或者 ``swr`` 。大多数情况下 ``llvmpipe`` 和 ``swr`` 比 ``softpipe`` 要快。

我查看 ``/usr/lib/dri`` 目录下有 ``etnaviv`` 和 ``zink`` 驱动，所以参考 `The Mesa 3D Graphics Library: Enviroment Variables <https://docs.mesa3d.org/envvars.html>`_ 尝试::

   export MESA_LOADER_DRIVER_OVERRIDE=zink

然后再次执行:

.. literalinclude:: wayland_vnc/wayvnc_headless_server
   :caption: 无头模式(headless)启动sway的VNC服务器

.. note::

   很不幸，我还没有找到适合 Apple Silicon M1Pro处理器图形卡的驱动方法，虽然上述MESA的环境变量设置驱动思路看起来可行，但是选择了zink等几个驱动都提示驱动不兼容。之前在 :ref:`asahi_linux` 发布文档看显卡驱动尚未就绪。后续再做尝试

.. note::

   其实简单的服务器端启动方法就是::

      wayvnc 0.0.0.0

- 如果服务器端启动正常，可以在客户端使用::

   ssh -FL 9901:localhost:5900 <user>@<SERVER_IP> sleep 5; vncviewer localhost:9901

在sway内部启动wayvnc
~~~~~~~~~~~~~~~~~~~~~~

- 在 :ref:`sway` 内部启动wayvnc:

.. literalinclude:: wayland_vnc/wayvnc_inside_sway
   :caption: 在sway内部启动wayvnc

VNC客户端
==========

Wayland的VNC客户端可以采用 `wlvncc <https://github.com/any1/wlvncc>`_ 。WayVNC 0.5支持使用OpenH268 RFB协议扩展的H.264编码。

- 编译依赖::

   GCC/clang
   meson
   ninja
   pkg-config
   wayland-protocols

- 编译和运行::

   git clone https://github.com/any1/aml.git
   git clone https://github.com/any1/wlvncc.git

   mkdir wlvncc/subprojects
   cd wlvncc/subprojects
   ln -s ../../aml .
   cd ..  #在wlvncc目录

   meson build
   ninja -C build

   ./build/wlvncc <address>

.. note::

   使用体验: 能够访问和连接 :ref:`macos` 共享的桌面，但是中文输入法切换存在问题，即使我避开了Win键，看起来能够输入中文，但是用空格键确认会卡住。后续我再尝试一下RDP方式访问桌面程序看看能否解决。

.. note::

   `waypipe <https://gitlab.freedesktop.org/mstoeckl/waypipe>`_ 实现了类似 ``ssh -X`` 的远程服务Wayland应用本地显示功能，有机会要实践一下，应该非常有用。

参考
=======

- `GitHub: wayvnc <https://github.com/any1/wayvnc>`_
- `Wayvnc server <https://n.ethz.ch/~dbernhard/wayvnc-server.html>`_
- `WayVNC 0.5 VNC Server For wlroots-Based Wayland Compositors Released <https://www.phoronix.com/news/WayVNC-0.5-Released>`_
