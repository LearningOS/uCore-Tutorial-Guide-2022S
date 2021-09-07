.. rCore-Tutorial-Book-v3 documentation master file, created by
   sphinx-quickstart on Thu Oct 29 22:25:54 2020.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

uCore-Tutorial-Book 第二版
==================================================

.. toctree::
   :maxdepth: 2
   :caption: Part1 - Just do it!
   :hidden:
   
   chapter0/index
   chapter1/index
   chapter2/index
   chapter3/index
   chapter4/index
   chapter5/index
   chapter6/index
   chapter7/index
   chapter8/index

.. toctree::
   :maxdepth: 2
   :caption: 开发注记
   :hidden:

   setup-sphinx
   rest-example
   log

欢迎来到 uCore-Tutorial-Book 第二版！

指导书简介
----------------------------

该指导书为 `THU` `OS` 课程实验 `C` 版实验指导书，旨在帮助同学们快速熟悉框架并完成书面和编程任务, `配套代码 <https://github.com/deathWish5/uCore-Tutorial-v2>`_ 。

此外，还推荐有余力同学们参考 `rCore-Tutorial 指导书 <https://rcore-os.github.io/rCore-Tutorial-Book-v3/index.html>`_。该书为一本从零开始写一个 OS 的教材，虽然是 rust 语言编写的，但对于 OS 的宏观特征和部分细节有更详细的描述。本指导书大量引用了该书的部分章节。

导读
---------------------

此实验的设计，大致还原了历史上 OS 的演变历程，感兴趣的同学可以参考 `WHAT-IS-OS <https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter0/1what-is-os.html>`_。
 
在正式进行实验之前，请先按照第零章章末的 :doc:`/chapter0/1setup-devel-env` 中的说明完成环境配置，确保能够正常运行 ch1 分支的代码。

此外需要注意指导书章节与实验提交要求的不一致，该指导书有 7 个章节，其中: ``ch1`` ``ch2`` ``ch3`` 对应课程要求 ``lab1``; ``ch4`` 对应 ``lab2``; ``ch5`` ``ch6`` 对应 ``lab3``; ``ch7`` 对应 ``lab4``。此外 ``ch8`` 对可选实验做了一定描述。


项目协作
----------------------

- :doc:`/setup-sphinx` 介绍了如何基于 Sphinx 框架配置文档开发环境，之后可以本地构建并渲染 html 或其他格式的文档；
- :doc:`/rest-example` 给出了目前编写文档才用的 ReStructuredText 标记语言的一些基础语法及用例；
- `该文档仓库文档仓库 <https://github.com/Exusial/uCore-Tutorial-Book>`_
- 时间仓促，本项目还有很多不完善之处，欢迎大家积极在每一个章节的评论区留言，或者提交 Issues 或 Pull Requests，让我们
  一起努力让这本书变得更好！


项目进度
-----------------------

- 2021-09-09: 基本完成初稿。