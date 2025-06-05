# SCF Doc

## Queue Buffer

### 数据传递

原理类似 virtio queue，在一个共享页上存放：
- meta data：锁、容量、req_index、rsp_index。
- req ring：请求的环形队列
- rsp ring：回复的环形队列
- desc：存放数据（参数和返回值）  

#### RTOS 到 Linux

- RTOS 发送（内核态）
    - 分配一个 desc 槽
    - 获取一个关联线程和对应 desc 槽的 token（CondVar）
    - 获取 queue 的锁
    - 向 desc 槽中写入参数
    - 将 desc 槽的 idx push 到 req_ring
    - 释放 queue 的锁  
- Linux 接收（用户态）
    - 获取 queue 的锁
    - 比较维护的 req_last_index 和实际的 req_index，如果不一致，说明有请求
    - 从 req_ring pop 出 idx
    - 找到 desc 中的参数
    - 释放 queue 的锁

#### Linux 到 RTOS

- Linux 发送（用户态）
    - 获取 queue 的锁
    - 向 desc 槽中写入参数
    - 将 desc 槽的 idx push 到 rsp_ring。
    - 释放 queue 的锁  
- RTOS 接收（内核态）
    - 获取 queue 的锁
    - 从 rsp_ring 中 pop 出 idx
    - 读取 desc 槽的返回值
    - 释放 desc 槽
    - 释放 queue 的锁  
    - 将 token（CondVar）设置为 true，并且写入返回值。

SyscallCondVar 类似信号，被用于在 Nimbos 中内核和线程的交流。

### 通知机制

#### RTOS 到 Linux

1. 如果是第一次通知，通过 `sendipi` 进行，并且发送 Uintr 初始化的请求，获取 Linux 程序 UPID 地址
2. Uintr 初始化之后，都通过 `senduipi` 进行。

#### Linux 到 RTOS

1. 第一次：
   1. 先 `ioctl` 到 driver（内核态） 中
   2. driver 通过 IPI 进行通知
2. Uintr 初始化后，用户态通过 `senduipi` 一个注册的虚拟的 receiver，将 Notification vector 修改成和 RTOS 约定好的 vector，从而借助 uintr 的通知机制来直接发送 IPI。

## Syscall 设计

### 同步映射

Syscall 中传递指针的时候，需要两侧系统对应虚拟地址的映射一致，解决的方法是同步两侧应用的映射
- 在 RTOS 应用映射发生修改时，通过 `syncmap`/`syncunmap` 告知 Linux 应用（Shadow Process），同步修改（map/unmap）

虽然听起来比较简单，但是实现中有一些细节需要注意：
1. `syncmap` 并不能算一个系统调用，应该被视作一个 kernel function。
   1. RTOS 第一个进程创建过程中，就需要 `syncmap`，但是此时还没有任何 Task 被创建，此时 RTOS 只能通过轮询来等待 Linux 的回复。
   2. `syncmap` 只能通过 patch 原 kernel 来进行调用，在 Nimbos 中的调用发生在页表类的 `map` 函数中。
2. OS 的虚拟地址映射实现的机制：
   1. 如果 OS 保证连续的虚拟地址映射到连续的物理地址（如原文献中的 Nuttx），那么实现起来会更加简单，也更有效率。
   2. Nimbos 中的映射函数不保证这一点，所以实现时只能一页一页地映射，比较麻烦，效率也更低。
3. 碰撞问题：Shadow Process 的数据应避开 RTOS 的应用
   1. 用户栈：RTOS 中的用户栈和 Linux 的用户栈必须错开，不能使用相同的虚拟地址。
      - 目前通过修改 RTOS 的栈的基址来避开 Shadow Process 的栈。
   2. elf：由于 RTOS 中往往限制了用户程序的大小（如 `1G`），所以可以在编译 Shadow Process 加入链接选项，将 Shadow Process 的所有段放到 `1G` 以上的位置。

### Syscall 分类

Syscall 大致可以分为三类：
1. local：RTOS 本地处理的 Syscall
2. remote：交给 Linux 全权处理的 Syscall
3. dual：需要两边联动处理的 Syscall

local syscall 不用解释，下面列出 SCF 目前实现的 dual syscall 和 remote syscall。

#### Remote Syscall

1. `open`/`close`：远程打开和关闭 Linux 中的文件。
2. `read`/`write`：远程读写 Linux 中的文件。
3. `stat`：对 Linux 文件的 stat。

理论上说这些 syscall 可以直通给 Linux，因为 Nimbos 没有 file system，但是在一个有 fs 的操作系统上可能需要特殊处理。

#### Dual Syscall

1. `fork`：
   1. RTOS 发送 `syncfork`，给 Shadow Process
   2. Shadow Process 调用 linux fork
      1. 子进程向 driver 申请一个 SCF slot（slot_num）
      2. 子进程将 slot_num 通过 pipe 发送给父进程
   3. 父进程返回 slot_num 给 RTOS
   4. RTOS `syncfork` 返回，接收到 slot_num
   5. RTOS 调用 RTOS fork，创建 RTOS 的子进程，将 slot_num 分配给子进程。
   6. 返回。
2. `exit`：
   1. RTOS 发送 `exit` 给 Shadow Process
   2. Shadow Process 调用 linux exit，退出
   3. RTOS 调用 RTOS exit 退出
3. `exec`：
   1. RTOS 通过 remote syscall `stat` 获取 elf 的大小 size
   2. RTOS 为程序 elf 分配空间 elf_data
   3. RTOS 通过 remote syscall `open` 获取 elf 的文件描述符 fd
   4. RTOS 通过 remote syscall `read` 远程读取 elf 文件到 elf_data。
   5. RTOS 调用内核函数，exec 对应 elf_data。
   6. RTOS 释放 elf_data
4. `clone`：因为 Nimbos 的线程机制比较不完善，所以实现
   1. RTOS 通过 remote syscall `clone` 通知 Shadow Process
   2. Shadow Process 增加引用计数
   3. RTOS 调用 clone，创建子线程
   4. 注：RTOS process 的多个线程共用一个 scf 资源（Buffer，中断号）

