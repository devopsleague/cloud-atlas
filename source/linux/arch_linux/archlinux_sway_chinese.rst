.. _archlinux_sway_chinese:

=================================
archlinux Sway图形桌面和中文环境
=================================

.. note::

   本文综合 :ref:`archlinux_sway` 和 :ref:`archlinux_chinese` ，结合最近(2024年10月)实践，力求快速准确完成精简的sway工作环境和中文显示、输入。

安装准备
================

由于 arch linux 稳定版本还没有集成 ``sway-im`` 包所提供的支持 :ref:`foot` 输入框，所以需要先完成 ;ref:`archlinux_aur` 安装，以便能够用 ``sway-im`` 替代 ``sway`` :

.. literalinclude:: archlinux_aur/install_yay
   :caption: 编译安装yay

安装
===========

- 安装 ``sway-im`` :

.. literalinclude:: archlinux_aur/install_sway-im
   :caption: 安装 ``sway-im``

.. note::

   近期没有再折腾，暂时记录一下，后续有机会再补充
