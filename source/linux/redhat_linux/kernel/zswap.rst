.. _zswap:

==============
zswap
==============

zswap是一项内核功能，可以为交换页面提供压缩的RAM缓存: 换出到磁盘的页面被压缩并存储到RAM的内存池中。一旦内存池写满或者RAM耗尽，则最近最少使用(LRU)页面就会被解压缩并写入到磁盘，就好像没有被拦截一样。在页面被解压缩到交换缓存之后，可以释放内存池中的压缩版本。

与 :ref:`zram` 的区别是， zswap 需要和交换设备一起工作，而 ``zram`` 是RAM中的交换设备，不需要后端支持的swap设备。

参考
======

- `arch linux: zswap <https://wiki.archlinux.org/title/zswap>`_
- `Use Zswap to Improve Performance on Linux PC with Low Amounts of RAM <https://www.maketecheasier.com/use-zswap-improve-old-linux-performance/>`_
