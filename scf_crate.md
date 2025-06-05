# SCF Crate 设计问题和后续规划

## 设计问题

SCF 中会用到很多和操作系统其他部件有所耦合的功能：
1. APIC 支持：sendipi 和 handler 注册
2. sync 相关：需要用到 Arc、Mutex、Spinlock
3. task 相关：
    1. 发送等待时需要调用 `yield_now`。
    2. 如果使用 uintr，必须要维护相关 Task Context。
4. alloc：需要使用 vec、Box。

### 暂时的解决和可能的优化方法

目前 Crate 的设计比较简单，尽量减少依赖，更多地是把需要的功能集成。

#### APIC

目前暂时将 APIC 相关的代码集成，放在 `interrupt` module 中，但是有一些潜在问题：
1. 多核：APIC 需要 percpu，如果后续需要多核，可能会产生依赖。
2. 虚拟地址问题：需要合理安置 lapic 和 ioapic 的虚拟地址

另外一个可能的方法是使用移植目标 OS 的中断支持，只需要注意将 APIC 的 IPI destination mode 设置为 Logical 即可，不过会增加耦合度。

#### sync 相关

目前暂时集成，如果移植到 starry/arceos，可以使用 `axsync` 的实现。

#### task 相关

这一部分比较棘手，目前通过接受回调的方式来实现：
1. 在 `init` 函数中加入一个参数 `scf_callbacks`，初始化时需要传入两个回调：
    1. `yield_now` 回调
    2. `current_uintr_context` 回调
2. 在 crate 内部逻辑使用时，调用这些回调函数。

问题在于 uintr 的 Context 究竟应该如何维护，可能的实现：
1. 集成在 Crate 中：理由有
    1. SCF 需要的 uintr 通知功能不会使用到全部的 uintr 功能，没有必要完整实现
    2. SCF 需要的 uintr 比较特殊，是 cross-os 的一个 uintr，与一般的 uintr 有所区别。
2. 单独实现一个 uintr Crate，支持 uintr 功能：
    1. 解耦、复用性更高
    2. 然而还是无法避免需要 patch 目标 OS

#### alloc

目前通过 extern crate 来引入，Starry 中组件也是这么做的。


## 后续规划（可能的话）

1. 线程机制的优化：由于 Nimbos 没有线程机制，所以目前的实现比较简陋，可能的优化：
    - 每一个线程独占一条信道：
        - 信道可以不设置 buffer，因为一个线程一次只能发送一个 syscall
        - 将一个进程的 syscall buffer 分槽划分给线程。
        - shadow process 和 driver 需要维护线程和 slot 的对应
2. 通知的优化：优化成只分配一个中断号
    - RTOS 的第一个进程通过这个中断号通知 Linux
    - 反向通知也使用这个中断号（复用）
    - 有 uintr 的情况：
        - 其他进程都由 fork 产生，可以即可分配 uintr 资源，从而避免使用普通中断
        - uintr 实现为 per thread
    - 没有 uintr 的情况：
        - 在 Syscall Buffer 中设计一个记录发送请求的线程
        - 在 driver 中找到对应的进程并 signal
        - 在 shadow process 维护的线程中找到精确的线程
3. kernel syncmap 的解决：
    1. 动机：在 exec 中，需要将一个 linux 中的 elf 加载到 RTOS 的内存中：
    2. 问题：加载到 RTOS 的内核内存还是用户内存？
        1. 内核内存：
            1. 加载到 kernel 的 heap 中，需要 kernel heap 足够大。
            2. 无法利用用户态映射的同步，指针无法直接传递：
                1. 最好的方法是保证 RTOS 和 Linux 的 kernel 的虚拟地址不重叠，然后把 kernel 在用户程序中的映射也同步给 Linux。（文献的实现）
                2. 目前暂时没有办法解决这一点，因为 Linux 的高地址不太能够使用。目前的解决方法是将 RTOS 的 kernel virt address 平移到一个低地址映射到 Linux，传递指针的时候需要特殊处理。
        2. 用户内存：
            1. 可以提前 mmap 出一片空间
            2. read 到这一片空间后，加载程序
            3. munmap 这一片空间
    3. 总结：如果加载到内核内存（更安全），则需要实现 kernel syncmap，让 Shadow process 可见 RTOS 的 kernel。


