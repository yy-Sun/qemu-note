# 概述

qemu 中   `main_loop`  主要运行在主线程中，以前也叫做 IO 线程，当然现在已经有专门的 IO 线程 即 `io thread`。

我在想为什么 qemu 需要运行 main_loop 循环, 如果没有 main_loop 循环会怎么样？

首先 qemu 的线程模型中必不可少的就是 vcpu 线程，当 vcpu 线程从 kvm 中退出到 qemu 时，即需要完成 IO 设备的模拟，当然设备的模拟可以直接在 vcpu 线程中完成，当IO设备的读写比较简单时，是没有问题的，但是如果 IO操作比较复杂，可能就会造成 vcpu 迟迟不能陷入到 KVM 中，导致 虚拟机中 cpu 得不到调度，可能会造成 guest 内部报错 cpu 软锁。

为了解决上述问题，我们可以在 vcpu 退出到 qemu 时，先建一个线程去实现设备模拟，但是这也有一个问题，对于 Guest 来说设备读写IO是非常频繁的，如果每次Guest读写IO时，我们都创建一个新线程来完成IO操作，当IO操作完成时再销毁该线程，那么线程的创建与销毁也太频繁了，这显然是不可取的。

还有一种方式，即为每个设备都创建一个线程，专门用来执行设备模拟的操作，这样就避免了频繁的对线程进行创建与销毁，但是这样也会有一个问题，那就是虚拟机的IO设备并不是一直在使用的，当 guest 空闲时，可能大部分IO设备都没有被调用，这样就会造成IO线程的浪费，而且还需要考虑这样一种情况，很多虚拟机分配的规格是很小的，比如1核，2核的虚拟机，为这样的虚拟机创建10多个，甚至更多的IO线程真的好吗？

综合以上，将IO操作放到一个线程中是比较合理的操作，即 vcpu 退出到 qemu 后，通知主线程去做相应的IO操作，然后vcpu再快速的陷入到KVM去做自己的事情，这点和中断的上半部与下半部操作有点类似。

实际上，将所有的IO操作都当到主线程中可能也不是那么合适，对于块设备，网卡设备，vnc数据处理，这些操作都比较消耗资源，它们也都被单独放到一个IO线程中调用了。
