.. MicroPython板子 documentation master file, created by
    sphinx-quickstart on Tue Aug 28 17:25:35 2018.
    You can adapt this file completely to your liking, but it should at least
    contain the root `toctree` directive.

MicroPython帮助文档
======================================

欢迎您，使用板子！

MicroPython板子是一块MicroPython微控制器板，它集成ESP32高性能双核芯片，使用当下最流行的Python编程语言，以便您轻松地将代码从电脑传输到板子中，从而体验程序创作的无穷乐趣！

.. image:: /images/掌控-动态.gif



| GitHub开源: https://github.com/labplus-cn/MicroPython
| MicroPython板子文档: https://MicroPython.readthedocs.io
| MicroPythonX IDE 软件编程文档: https://MicroPythonX.readthedocs.io
| MicroPython library(提供各类库): https://MicroPython-lib.readthedocs.io

.. Hint::

    我已将板子项目的软硬件资源分享至GitHub，供各位板子爱好者学习阅览！

.. Attention::

     该项目正在积极开发中。由于ESP32仍然面向开发人员，并非所有外围设备都能完美使用，可能仍然存在一些bug，我们将会不断及时修复和更新。

---------

MicroPython板子资料
---------

.. toctree::
    :maxdepth: 2
    
    board/index.rst


---------

.. toctree::
    :maxdepth: 2
    :caption: 板子教程

    tutorials/basics/index.rst
    tutorials/advance/index.rst

---------

经典案例
++++++

.. toctree::
    :maxdepth: 2


    classic/index.rst

---------

MicroPython类库
----------------

.. toctree::
    :maxdepth: 1
    :caption: MicroPython类库
    :hidden:
    
    library/pythonStd/index.rst
    library/micropython/index.rst


=========================================    ======================================================
 :ref:`Python标准库<pythonStd>`                兼容CPython,内含Python内建函数、常用module
 :ref:`MicroPython类库<microPythonModu>`       MicroPython的ESP32硬件控制层的模块     
=========================================    ======================================================


您可以通过 ``help()`` 发现可用的内置库，在REPL中输入以下内容来导入::

    >>> help('modules')


除了本文档中描述的内置库之外，在 :term:`micropython-lib`  中还可以找到来自Python标准库的更多模块以及对它的进一步微Python扩展。

---------

MicroPython类库
-------------

.. toctree::
    :maxdepth: 1
    :caption: MicroPython类库
    :hidden:
    
    library/MicroPython/index.rst
                                                
======================================    ====================================================================  
 :ref:`MicroPython模块<MicroPython.py>`            提供了MicroPython板子特有拓展类。:ref:`MicroPython.py源码查看 <MicroPython_code>`
 :ref:`music模块<music.py>`                兼容micro:bit music 模块
 :ref:`urequests模块<urequests>`           从CPython移植过来的HTTP客户端的第三方库,提供各种HTTP请求方式
 :ref:`umqtt.simple模块<umqtt.simple>`    提供简单MQTT客户端功能 
 :ref:`servo模块<servo_api>`               舵机驱动 
 :ref:`gui模块<gui>`                        提供GUI类的绘制元素 
 :ref:`audio模块<audio>`                    音频播放录音
 :ref:`radio模块<radio>`                    无线广播
======================================    ====================================================================

---------

.. toctree::
    :maxdepth: 1
    :caption: MicroPython 语法

    reference/index.rst
 
有关MicroPython特定语言功能的语言参考信息



.. toctree::
    :hidden:
    
    release.rst


掌控拓展板
----------

.. toctree::
    :hidden:

    extboard/index.rst

.. image:: /images/extboard/extboard_250.png
  :scale: 80 %
  :target: extboard/index.html


------------------

索引
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
