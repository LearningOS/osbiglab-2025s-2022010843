# 进展文档
## 3.20 进展
### 进展
- 阅读了 cRTOS 文献
- 配置了 qemu、RVM、nimbos，并且运行 base
- 学习了 x86 uintr spec。
## 3.27 进展
### 进展
- qemu-uintr：已经配置好
- RVM/Linux 侧：
  - uintr-linux-kernel 的配置：
    - 内核的编译
    - 在 qemu-uintr 中运行（卡了比较久）
- RTOS 侧：
  -  用 qemu-uintr 运行 nimbos

## 4.3 进展
### 进展
- Linux jailhouse：通过修改 kernel 的方式解决了 jailhouse 的适配问题
- 虚拟机运行问题：
  - 不使用虚拟 CPU：无法正常运行 Syscall forwarding，linux 侧未收到 ipi（原因暂不明）
  - qemu64 设定的 vendor 是 amd，RVM-amd 和 nimbos 不能正常配合。
- 整理了这几周的脚本和代码，push 到了 github

## 4.10 进展
### 进展
- Nimbos 以及 nimbos-driver 上 shadow process 机制的初步实现
  - 在创建 nimbos 中进程时，通过 syncmap 与 linux 的 shadow process 同步映射
    - shadow process 通过链接参数预留低地址区域
    - shadow process 通过调用 mmap，借助 driver 实现的自定义 mmap 函数来实现映射
  - 在 fork 时：
    - shadow process 也 fork 出子进程，和 nimbos 侧的子进程同步
    - 分配新的 syscall queue 和 irq_num，供两个子进程通信
    - 在新的信道上，通过 syncmap 同步两个子进程的映射
### 规划
- 整理代码，让新功能作为一个模块引入 nimbos，提交 PR
- 实现更多的 syscall：
  - sys_clone：nimbos 的线程机制和 linux 不同，需要特殊处理
  - sys_exit：在进程退出时，同步释放 shadow process （需要在 sys_clone 的基础上实现，因为 nimbos 的线程实现的特点）
