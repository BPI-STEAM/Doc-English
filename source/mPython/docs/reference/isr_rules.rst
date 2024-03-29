.. _isr_rules:

编写中断处理程序
==========================

在适当硬件上，MicroPython提供了使用Python编写中断处理程序的功能。中断处理程序-又称中断服务程序（ISR），定义为回调函数。
这些函数都是作为对诸如计时器触发或引脚上的电压变化等事件的回应而执行的。这些事件可能在程序代码的执行的任何时间点出现。
这就带来了重大影响，其中有些是特定于MicroPython语言的。其他的则对所有可回应实时事件的系统都适用。本文件中涵盖了与语言相关的问题，
并对实时编程进行简短介绍。

介绍中使用了一些诸如"慢""尽可能快"等模糊语言，这并非无意为之，因为速度取决于应用程序。ISR可接受的持续时间取决于多个
因素：中断发生的速度、主程序的性质和其他并发事件的存在。

提示与推荐练习
------------------------------

此处总结了下面详细介绍的几点，并给出了中断处理程序代码的主要建议。

* 尽量使代码短而简单。
* 避免内存分配：无附加列表或插入字典，无浮点数。
* 在ISR返回多个字节的情况下，使用预先分配的 ``bytearray`` 。若在ISR何主程序之间共享多个整数，则使用数组（ ``array.array`` ）。
* 在主程序和ISR之间共享数据的情况下，考虑在主程序中访问数据前禁用中断，并在此后立即重新启用（请参阅Critical Sections）。
* 分配紧急异常缓冲区（见下）。


MicroPython问题
------------------

紧急异常缓冲区
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

若在ISR中出现错误，MicroPython无法生成错误报告，除非为此创建一个特殊缓冲区。若使用中断的任何程序中包含以下代码，则调试将被简化。

.. code:: python

    import micropython
    micropython.alloc_emergency_exception_buf(100)

简化
~~~~~~~~~~

由于各种原因，保持ISR代码尽可能简短十分重要。其应在事件发生后；自己执行：可延迟的操作应委托给主程序循环。
典型情况下，一个ISR将处理引起中断的硬件设备，为下一个中断做好准备。ISR将与主循环通信，通过更新共享数据来表明中断已发生，
并返回。ISR应尽快将控制权返还给主循环。这并非典型的MicroPython问题，以下有详细介绍（ :ref:`below <ISR>`）。

ISR和主程序间的通信
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

通常，ISR需与主程序通信。最简单的通信方式是通过一个或多个共享数据对象，申明为全局或通过一类共享（见下）。
但是这种方法有很多局限性和危害，下面将进行详细介绍。整数、 ``bytes`` 和 ``bytearray`` 对象以及数组（来自数组模块，可储存多种数据类型）通常用于此目的。

对象方法用作回调
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MicroPython支持这一启用ISR来与底层代码共享实例变量的强大技术。它还使得实现设备驱动的类支持多种设备实例。以下示例即为使两个LED以不同速率闪烁。

.. code:: python

    import pyb, micropython
    micropython.alloc_emergency_exception_buf(100)
    class Foo(object):
       def __init__(self, timer, led):
          self.led = led
          timer.callback(self.cb)
       def cb(self, tim):
          self.led.toggle()

    red = Foo(pyb.Timer(4, freq=1), pyb.LED(1))
    greeen = Foo(pyb.Timer(2, freq=0.8), pyb.LED(2))

在此示例中， ``red`` 实例将计时器4与LED 1联系起来：当计时器4中断发生时，则调用 ``red.cb()`` ，以使LED 1改变状态。
``green`` 实例的操作与之相似：计时器2的中断引发 ``green.cb()`` 的执行，并切换LED 2。实例方法的使用有两大益处。其一，
单个类使得代码可在多个硬件实例间共享。其二，作为绑定方法，回调函数的首个参数为 ``self`` 。这就使得回调可访问实例数据，
并在连续回调间保存状态。例如，若上类在构造函数中将变量 ``self.count``
设置为0， ``cb()`` 会递增计数器。 ``red`` 和 ``green`` 实例将保存每个LED已经改变状态的次数的独立计数。

创建Python对象
~~~~~~~~~~~~~~~~~~~~~~~~~~

ISR无法创建Python对象的实例。这是由于MicroPython需从称为堆的空闲内存块的存储中为对象分配内存。
这在中断处理程序中是不允许的，因为堆分配并非可重入的。换言之，当主程序正在执行分配时，
中断可能发生-为保持堆的完整性，解释器不允许ISR代码中的内存分配。

其影响之一为ISR无法使用浮点数算法；这是因为浮点数为Python对象。类似地，ISR无法附加项目到列表中。
在实际操作中，很难精准确定哪个代码结构将尝试执行内存分配并引发错误信息：使ISR代码尽可能简短的另一原因。

避免此类问题的一个方法是ISR使用预分配缓冲区。例如，一个类构造函数创建一个 ``bytearray`` 实例和一个布尔标志。
ISR方法将数据分配到缓冲区中的 位置并设置标志。当实例化对象时，内存分配在主程序代码中实现，而非在ISR中。

MicroPython库I/O方法通常提供使用预分配缓冲区的选项。例如， ``pyb.i2c.recv()`` 可接受一个可变缓冲区作为其首个参数：这使其可在ISR中使用。

不使用类或全局变量创建对象的方法如下所示:

.. code:: python

    def set_volume(t, buf=bytearray(3)):
       buf[0] = 0xa5
       buf[1] = t >> 4
       buf[2] = 0x5a
       return buf

首次加载函数时，编译程序实例化默认 ``buf`` 参数（通常在其所在模块被导入时）。

使用Python对象
~~~~~~~~~~~~~~~~~~~~~

由于Python对象的运行方式，对对象产生了进一步的限制。当执行 ``import`` 语句时，Pyton代码编译为字节代码。
当运行代码时，解释器读取每一字节代码，并将其作为一组机器代码指令执行。鉴于在机器代码指令之间的任何时刻都可能发生中断，
Python代码的原始行可能只是部分执行。其结果就是，类似一个在主循环中修改的组、列表或库可能在中断发生时缺少内部一致性。

典型结果如下。在极少数情况下，ISR将在对象部分更新时在精确时间运行。当ISR尝试读取对象时，会导致崩溃。
由于此类问题仅在极少数且随机的情况下出现，因而很难诊断。有很多方法可以避开这一问题，请参见
:ref:`Critical Sections <Critical>` below.

了解对象的更改的组成很重要。对字典等内置类型的更改问题重重。更改数组或字节数组的内容则相对容易。
这是由于字节或词作为可中断的单一机器代码写入：按照实时编程的说法，写入是原子的。用户定义的对象可能会实例化整数、数组或字节数组，
主循环和ISR都可修改其内容。

MicroPython支持任意精度的整数。介于2**30 -1和-2**30之间的值将储存在单一机器词中。更大的值则储存为Python对象。
因此，不认为对长整数的修改是原子的。在ISR中使用长整数并不安全，因为变量值更改时，可能会尝试分配内存。

克服浮点数限制
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

总而言之，最好避免在ISR中使用浮点数：硬件通常在主循环中处理整数并转换为浮点数。但是，有一些需要浮点数的DSP算法。
在具有硬件浮点数的平台中（例如Pyboard），内联的ARM Thumb汇编程序可用来避免此限制。这是由于处理器将浮点值储存在机器字中；
该值可通过一组浮点数在ISR和主程序代码中共享。

异常
----------

若ISR出现异常，该异常不会传播到主循环中。除非由ISR代码处理异常，否则中断将被禁用。

一般问题
--------------

这是对实时编程的简短介绍。初学者应注意：实时编程的设计错误可能导致极难诊断的故障。这是由于它们可能极少发生且其发生的时间间隔是完全随机的。
保证最初的设计准确无误并在问题发生前预估问题至关重要。中断处理程序和主程序都需在设计时考虑到以下问题。

.. _ISR:

中断处理程序设计
~~~~~~~~~~~~~~~~~~~~~~~~

如上所述，ISR的设计应尽量简单，它们应在较短的、可预计的时间段内返回。这很重要，当ISR运行时，主循环并未运行：主循环不可避免地会在代码中的随机处暂停。
这种暂停可能导致较难诊断的故障，尤其是在暂停的持续时间较长或可变时。为理解ISR的运行时间，则要求对中断优先级基本了解。

中断通过一个优先级方案进行组织。ISR代码本身可能被更高优先级的中断而中断。若两个中断共享数据（参见下面的Critical Sections），则产生一定影响。
若这种中断发生，则在ISR代码中插入延迟。若在ISR运行时发生更低优先级的中断，则较低优先级的中断将失效。慢ISR的另一问题是：在执行中同一类型的中断第二次出现。
第二个中断将会在第一个中断终止后处理。然而，若后续的中断的速率仍旧超过ISR所能容纳的数值，则结果将不容乐观。

因此，应避免或最小化循环结构。应避免对除中断设备外的其他设备进行I/O：如磁盘存取、 ``print`` 语句和UART访问等相对较低，其持续时长各不相同。
此处另一问题是文件系统函数不可重入：在ISR或主程序中使用文件系统I/O 可能会遇到许多问题。重要的是，ISR不应等待事件。若确保代码在可预计时间内返回，
如切换引脚或LED，则I/O为可接受的。通过I2C或SPI访问中断设备可能很有必要，但应计算这些访问所花费的时间，并评估其对应用程序的影响。

通常需要在ISR和主循环间共享数据。可通过全句变量或类或实例变量来实现共享。变量通常为整数或布尔类型、整数或字节数组（一个预分配的整数数组比列表访问更快）。
在ISR修改多个值时，有必要考虑主程序访问了部分值（而非全部值）时发生中断的情况。这会导致不一致性。

考虑以下设计。ISR将输入数据储存到字节对象，将接收字节的数量添加到准备处理的总字节数量的整数中。主程序读取字节数量，处理字节，并清除准备就绪的字节数。
在主程序读取字节数并出现中断后，此过程才开始运行。ISR将添加的数据放入缓冲区并更新接收的数字，但主程序已读取了数字，因此处理原来接收的数据。
新的等待接收的字节就丢失了。

有许多避免此问题的方法，最简单的是使用环形缓冲器。若无法使用具有固有线程安全性的结构，则下面将介绍其他方法。

可重入性
~~~~~~~~~~

若一个函数或方法在主程序与一个或多个ISR间或在不同ISR间共享，则可能引发一个潜在问题。函数本身可能被中断，该函数的另一个实例运行。
若此问题出现，函数须设计为可重入。如何实现这一设计是超出本文范围的高级任务

.. _Critical:

临界区
~~~~~~~~~~~~~~~~~

代码的临界区的示例是访问多个变量，这些变量受ISR影响。若中断在对单个变量的访问间发生，则其值将会不一致。
这是一种叫作"竞态条件"的问题的实例：ISR和主程序循环争相修改变量。为避免不一致性，必须采取一种方法来确保ISR不会在临界区持续过程中修改值。
实现此目的的方式之一是在临界区开始前发出 ``pyb.disable_irq()`` ，并在其结束时发出 ``pyb.enable_irq()`` 。这是此方法的示例:

.. code:: python

    import pyb, micropython, array
    micropython.alloc_emergency_exception_buf(100)

    class BoundsException(Exception):
       pass

    ARRAYSIZE = const(20)
    index = 0
    data = array.array('i', 0 for x in range(ARRAYSIZE))

    def callback1(t):
       global data, index
       for x in range(5):
          data[index] = pyb.rng() # simulate input 模拟输入
          index += 1
          if index >= ARRAYSIZE:
             raise BoundsException('Array bounds exceeded')

    tim4 = pyb.Timer(4, freq=100, callback=callback1)

    for loop in range(1000):
       if index > 0:
          irq_state = pyb.disable_irq() # Start of critical section 临界区的开始
          for x in range(index):
             print(data[x])
          index = 0
          pyb.enable_irq(irq_state) # End of critical section 临界区的结束
          print('loop {}'.format(loop))
       pyb.delay(1)

    tim4.callback(None)

临界区可包含一行代码和一个变量。思考以下的代码碎片。

.. code:: python

    count = 0
    def cb(): # An interrupt callback 一个中断回调
       count +=1
    def main():
       # Code to set up the interrupt callback omitted 设置省略的中断回调的代码
       while True:
          count += 1

此示例说明了故障的潜在原因。主循环中的 ``count += 1`` 行携带了一个称为"读-修改-写"的特定的竞态条件问题。这是实时系统中故障的典型原因。
在主循环中，读取 ``t.counter`` 值，将其增加1，并写回。在少数情况下，中断发生在读取后、写入前。中断更改 ``t.counter`` ，但其改变在ISR返回时被主循环覆盖。
在实时系统中，这可能会导致极少的、难以预测的故障。

如上所述，若在主代码中修改了Python内置类型的实例或在ISR中访问实例，则应多加注意。执行更改的代码应被视为临界区，以确保ISR运行时实例处于有效状态。

若在不同ISR间共享数据集，则应特别注意。此处的问题在于较低优先级的中断部分地更新了共享数据时，较高优先级的中断可能在此时发生。
处理这种情况是超出本文范围的高级任务，但下面的互斥对象有时可使用。

在临界区间内禁用中断是常用也是最简单的方法，但是其禁用了所有中断，甚至包括并不会引发问题的中断。通常我不希望长时间禁用中断。
在计时器中断的情况下，其将可变性引入到回调发生的时刻。在设备中断时，其可导致设备服务太晚，可能会丢失数据或使设备硬件出现超限错误。
如ISR，主代码中的临界区的持续时长应较短且可预测。

处理临界区（彻底减少禁用中断的时间）的一个方法是使用名为"互斥体"的对象（得名于互相排斥的概念）。主程序在运行临界区前螺钉互斥体，
并在结束时解锁。ISR测试互斥体是否锁定。若锁定，则其避开临界区并返回。此设计的难题在于，在访问临界变量被拒绝时，如何定义ISR应做出的行为。
此处提供互斥体的简单示例：
`here <https://github.com/peterhinch/micropython-samples.git>`_。注意：互斥体代码禁用了中断，但其禁用仅限于8个机器指令期间。
此方法的优点是几乎不影响其他其他中断。

中断和REPL
~~~~~~~~~~~~~~~~~~~~~~~

中断处理程序（如与计时器相关的中断处理程序）可在程序结束后继续运行。这可能会产生意想不到的结果，在此种情况下，您可能期望引发回调的对象已超出范围。
例如在Pyboard中:

.. code:: python

    def bar():
       foo = pyb.Timer(2, freq=4, callback=lambda t: print('.', end=''))

    bar()

此代码将持续运行，除非计时器被显式禁用或使用 ``ctrl D`` 重置板。