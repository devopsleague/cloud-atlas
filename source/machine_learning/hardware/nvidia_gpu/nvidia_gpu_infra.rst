.. _nvidia_gpu_infra:

===============
NVIDIA GPU架构
===============

NVIDIA的GPU产品：

* GeForce系列 - 娱乐, 分为桌面平台与移动平台，移动平台此系列主要应用到笔记本电脑上的显卡，一般后面带个M或其他标识
* Quadro系列 - 专业可视化
* Tesla系列 - 利用图形处理器进行高性能运算，部分型号无显示输出接头。
* GoForce与Tegra系列 - Tegra(图睿)是系统单片机，替代GoForce系列。应用于智能手机、便携式媒体播放器和平板电脑等。每个 Tegra 内置ARM架构的处理器核心、基于GeForce的图形处理器等。

NVIDIA GPU微架构：

* Tesla
* Fermi
* Kepler
* Maxwell
* Pascal
* Volta

总体来说，Tesla架构的GPU计算能力为1.*, Fermi架构的GPU计算能力为2.*，Kepler架构的GPU计算能力为3.*，Maxwell架构的GPU的计算能力为5.*，Pascal架构的GPU计算能力为6.*，Volta架构的GPU计算能力为7.*。

此外，从 Volta 微架构开始，NVIDIA 引入了 :ref:`tensor_cores` 特殊设计的GPU核心，目前已经经历了四代

更详细的产品，计算能力参见 `NVIDIA官网-推荐开发者使用的 GPU <https://developer.nvidia.com/zh-cn/cuda-gpus>`_

2012年，NVIDIA推出了NVIDIA Virtual GPU ( :ref:`vgpu` ) 可以让多个虚拟机(VMs)同时直接访问一块单一的物理GPU

参考
======

- `NVidia产品和微架构 <http://juniorprincewang.github.io/2018/01/13/NVidia%E4%BA%A7%E5%93%81%E5%92%8C%E5%BE%AE%E6%9E%B6%E6%9E%84/>`_
- `NVIDIA官网-推荐开发者使用的 GPU <https://developer.nvidia.com/zh-cn/cuda-gpus>`_
- `NVIDIA GPU的一些解析（一） <https://zhuanlan.zhihu.com/p/258196004>`_
