.. _introduce_flink:

============
Flink简介
============

.. note::

   `Flink中文社区视频教程 <https://github.com/flink-china/flink-training-course>`_ 提供入门和进阶的指南，可以参考学习。

   GitHub上 zhisheng17 有一个非常详尽的Flink学习指南 `flink-learning <https://github.com/zhisheng17/flink-learning>`_ 是目前我看到最完善的学习资料。

在大规模流数据处理中，需要对不断无限产生对数据实时计算分析。在这个领域， :ref:`spark` , strom 和 Flink 都提供了各自的解决方案。

流数据处理难题
===============

流数据处理是极具挑战的技术：

- 在大型分布式系统中，数据一致性对事件发生顺序有限制
- 大多数企业常用的数据架构是假设数据是有头有尾的有限集


