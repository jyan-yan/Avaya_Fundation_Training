# Task_struct

- “进程”在内核眼里 = 一组共享资源的 task_struct 集合
- “线程”在内核眼里 = 一个 task_struct（调度的基本单位）

- 调度器调度的是 **task_struct**，不是“进程”
- 单线程进程：1 个 task_struct（主线程），它的 `pid == tgid`
- 多线程进程：N 个 task_struct，它们**同属一个线程组**（`tgid` 相同），但各自 `pid` 不同

```text
进程 PID=100（用户视角）

task_struct 主线程  pid=100 tgid=100  <- group_leader
task_struct 线程A   pid=101 tgid=100
task_struct 线程B   pid=102 tgid=100
```

对应关系：

| 用户态接口                             | 返回                  | 内核字段              |
| -------------------------------------- | --------------------- | --------------------- |
| `getpid()`                             | 100（整个进程）       | `task_struct->tgid`   |
| `gettid()`                             | 101 / 102（具体线程） | `task_struct->pid`    |
| `ps` 一列 / `/proc/<pid>/task/` 子目录 | 每个线程一个          | 每个 task_struct 一个 |

**共享与独立**

同一组的 N 个 task_struct 之所以是一个“进程”，是因为它们共享关键资源：

```text
共享（指向同一份）：
  mm_struct（地址空间）   <- 同一堆/全局变量/代码
  files_struct（fd 表）
  fs_struct、signal_struct、nsproxy

独立（各自一份）：
  pid / tgid
  内核栈 stack
  调度信息 se / 优先级 / 状态
  阻塞信号掩码 blocked
  寄存器上下文
```

所以前面聊的“多线程共享 Heap、各有各的栈”就是这个机制：`mm` 共享 → Heap 共享；`stack` 字段各自独立 → 内核栈（以及用户态栈在 `mm` 中各占一段映射）独立。

一句话总结：

> 内核里**没有“进程”对象，只有 task_struct**。一个进程就是一组 `tgid` 相同的 task_struct；单线程进程只有一个，多线程进程每个线程各有一个。