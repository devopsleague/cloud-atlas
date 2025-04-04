.. _sphinx_readthedocs_yaml:

================================================
Sphinx结合readthdocs.yaml实现定制Read The Docs
================================================

.. _sphinx_typeerror:

Build Filed: TypeError: 'generator'...
================================================

我在使用readthdocs.io平台构建sphinx文档时，在2021年10月开始遇到build fail。报错同 `Build Failed. TypeError: 'generator' object is not subscriptable #8616 <https://github.com/readthedocs/readthedocs.org/issues/8616>`_ ::

   ...
   TypeError: 'generator' object is not subscriptable

   Exception occurred:
     File "/home/docs/checkouts/readthedocs.org/user_builds/diadocsdk-1c/envs/dev/lib/python3.7/site-packages/sphinx/domains/std.py", line 638, in note_labels
       n = node.traverse(addnodes.toctree)[0]
   TypeError: 'generator' object is not subscriptable

原因是对于2020年10月之前创建的Sphinx项目，ReadTheDocs(RTD)会使用Sphinx<2的版本，此时如果你更新过pip环境， ``docutils-0.18`` 就会不兼容。所以上传到 RTD 平台编译就会失败。不过，在我本地编译的时候，安装的是 ``docutils==0.17.1`` ，所以没有任何问题。

解决方法是明确指定RTD环境，参考 `RTD eproducible Builds <https://docs.readthedocs.io/en/stable/guides/reproducible-builds.html>`_ 特别是 `RTD pinning dependencies <https://docs.readthedocs.io/en/stable/guides/reproducible-builds.html#pinning-dependencies>`_

- 在项目目录下添加一个配置文件 ``.readthedocs.yaml`` :

.. literalinclude:: sphinx_readthedocs_yaml/readthedocs.yaml
   :language: yaml

这个配置文件会指定项目需要的安装依赖包

- 在源代码 ``sources`` 目录下添加 ``requirements.txt`` :

.. literalinclude:: sphinx_readthedocs_yaml/requirements.txt

- 在源代码 ``sources`` 目录下添加 ``environment.yaml`` 这个文件非常重要，会强制 ReadTheDocs(RTD) 安装和使用指定pip包，这样就能能够避免 ``docutils`` 版本冲突:

.. literalinclude:: sphinx_readthedocs_yaml/environment.yaml
   :language: yaml
   :emphasize-lines: 9

- 然后 ``make html`` 没有问题后， ``git push`` 到github，再观察 ReadTheDocs(RTD) 就会发现build成功了。

python版本指定升级以适配Sphinx
===============================

指定graphviz依赖
================

:ref:`` 运行需要系统中安装 ``graphviz`` 工具，所以也需要对 ``readthdocs.yaml`` 设置，否则Read The Docs平台编译会WARNING:

.. literalinclude:: sphinx_readthedocs_yaml/graphviz_err
   :caption: 环境中缺乏 ``graphviz`` 工具时RTD报错

类似上述指定python版本，我这里也指定graphviz的安装版本 ``2.42`` :

.. literalinclude:: sphinx_readthedocs_yaml/readthedocs_now.yaml
   :language: yaml
   :caption: 指定环境安装 ``graphviz``




参考
=======

- `Build Failed. TypeError: 'generator' object is not subscriptable #8616 <https://github.com/readthedocs/readthedocs.org/issues/8616>`_
