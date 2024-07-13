操作系统有两个主要工作，一个是计算，另一个就是I/O。本文从内核态到用户态，自底至上地介绍操作系统的I/O模型。

## 整体架构

以读取数据为例：

用户态->系统调用->内核态->通过I/O设备控制器读取设备-> 内核缓冲区->拷贝到用户态缓冲区

Q：为什么不能直接I/O设备写到用户指定的地址？

A: https://www.quora.com/Why-we-need-copy_from_user-as-the-kernel-can-access-all-the-memory-If-we-see-the-copy_from_user-implementation-again-we-are-copying-data-to-the-kernel-memory-using-memcpy-Doesnt-it-an-extra-overhead

## 硬件

i/o硬件：port、bus、controller

<img src="https://oss.seineo.cn/images/202407092359344.png" alt="截屏2024-07-09 23.59.00" style="zoom:50%;" />

北桥芯片处理高速设备，如显卡、内存；南桥芯片处理相对低俗的设备，如常见的I/O设备。

## 内核态视角

I/O设备各式各样，但是作为操作系统，我们会希望有统一的接口来管理它们，这些接口称为**设备驱动**。当然不会有一个接口适配于所有设备，存储设备、传输设备、人机交互设备等等都有各自类别的设备驱动。操作系统的内核代码可以像本地调用代码一样使用设备驱动程序的接口，而设备驱动程序是面向设备控制器的代码，它发出操控设备控制器的指令后，就可以操作设备控制器来管理I/O设备。



I/O操作从不同维度有不同的分类：

-   如何对设备寻址：
    -   Port-mapped I/O：适合小数据量，写到I/O寄存器里，不占用内存空间。
    -   Memory-mapped I/O：适合大数据量，写到指定物理地址，占用了内存空间。
-   如何与设备交互：
    -   Busy-waiting：基本消失了。
    -   Interrupt
-   数据传输的控制方：
    -   CPU
    -   DMA



DMA：

<img src="https://oss.seineo.cn/images/202407070044020.png" alt="截屏2024-07-07 00.43.56" style="zoom: 50%;" />

### 零拷贝技术

可以结合这个：https://www.xiaolincoding.com/os/8_network_system/zero_copy.html#%E6%80%BB%E7%BB%93 

根据[维基百科的定义](https://en.wikipedia.org/wiki/Zero-copy)：“零拷贝”描述了CPU不执行将数据从一个存储区域复制到另一存储区域的任务或避免不必要的数据复制的计算机操作。

对于需要文件传输的服务，会面临需要多次的文件数据传输，磁盘<-> 内存 <-> 网卡

以读磁盘文件然后写到响应体返回为例：

```c
read();
write();
```

这里就有四次拷贝操作：

-   read：磁盘设备控制器的缓冲区 由DMA 写到 内核缓冲区， 内核缓冲区拷贝到用户缓冲区
-   Write：用户缓冲区写到内核缓冲区， 内核缓冲区由DMA写到网卡设备控制器的缓冲区。

为了减少拷贝次数，我们很容易想到共享内存，也就是映射内核缓冲区到用户缓冲区，这样例子中cpu参与拷贝次数就少了一次（read）。但是写到网卡的时候还是需要从用户拷贝到内核，为什么继续减少拷贝次数，那么我们就需要直接让数据不进入用户态，这样就不用拷贝进去又拷贝出来，`sendfile`函数就是这么做的，可以直接把内核缓冲区里的数据拷贝到 socket 缓冲区里。那么这样做其实还是3次拷贝，只是不用切换到用户态了，看图我们会想到，为啥要拷贝呢？直接一个设备写到另一个设备就好了呗，但目前还没有这样的管理设备。但我们是不是可以读进来到内核缓冲区后，直接写到另一个设备？可以的，这样我们就只需要两次数据拷贝，且CPU不参与数据拷贝操作。

应用：kafka、nginx

## 用户态视角

我们这里对统一的设备来说，分为阻塞与非阻塞。 （磁盘类没有同步异步的概念，网络这些才有，可以提到网络的时候说一嘴就行）

程序员一般是同步思维，

read/write

select/poll/epoll/kqueue 不过一般用于网络i/o。

为什么epoll不支持磁盘io：https://www.pulpcode.cn/2021/04/03/regular-files-with-epoll/

## 参考

-   无意间刷到的国立清华教授课程，讲得很清晰：https://ocw.nthu.edu.tw/ocw/index.php?page=chapter&cid=141&chid=3025