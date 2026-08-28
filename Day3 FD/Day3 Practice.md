# Practice

1. Understand the fundamentals of `struct task_struct` and explore process attributes via the `/proc/<pid>/` directory.
2. Learn the basics of **Linux system calls** and use the `man 2` command to research `open()`, `close()`, `read()`, and `write()`.
3. Implement a custom `cat` utility in C using these basic system calls.
4. Summarize your learnings into technical notes and push both the notes and the source code to GitHub.



## 1. Understand the fundamentals of `struct task_struct` and explore process attributes via the `/proc/<pid>/` directory.

### **task_struct 是什么**

`struct task_struct` 是 Linux 内核中描述一个**进程/线程**的核心数据结构，定义在 `include/linux/sched.h`。内核用它来管理调度、内存、文件、信号等一切进程资源。在 Linux 里**线程也被表示为 task_struct**（轻量级进程），只是共享部分资源。

这个结构体非常庞大（数百个字段，占几 KB 内存），由 slab 分配器分配。下面按功能分类看主要字段。

**简化后的结构**

```c
struct task_struct {
    /* 1. 标识信息 */
    pid_t               pid;          // 线程 ID（内核视角的 PID）
    pid_t               tgid;         // 线程组 ID（用户视角的 PID）
    char                comm[TASK_COMM_LEN];  // 进程名，如 "sshd"

    /* 2. 进程状态 */
    volatile long       __state;      // TASK_RUNNING / TASK_INTERRUPTIBLE 等
    unsigned int        __flags;      // PF_EXITING、PF_KTHREAD 等标志位

    /* 3. 内核栈与线程信息 */
    void                *stack;       // 指向该任务的内核栈
    struct thread_info  thread_info;

    /* 4. 亲缘关系：构成进程树 */
    struct task_struct  *real_parent; // 真实父进程
    struct task_struct  *parent;      // 接收信号的父进程
    struct list_head    children;     // 子进程链表
    struct list_head    sibling;      // 兄弟链表
    struct task_struct  *group_leader;// 线程组组长

    /* 5. 调度相关 */
    int                 prio;         // 动态优先级
    unsigned int        policy;       // SCHED_NORMAL / SCHED_FIFO / SCHED_RR ...
    const struct sched_class *sched_class; // 调度类（CFS/RT）
    struct sched_entity se;           // CFS 调度实体，含 vruntime

    /* 6. 资源指针：线程间共享与否的关键 */
    struct mm_struct    *mm;          // 地址空间（页表、堆栈区域）
    struct files_struct *files;       // 打开文件表（fd 数组）
    struct fs_struct    *fs;          // 根目录、当前工作目录
    struct nsproxy      *nsproxy;     // 命名空间
    struct signal_struct *signal;     // 共享的信号处理相关信息
    struct sighand_struct __rcu *sighand; // 信号处理函数表
    sigset_t            blocked;      // 被阻塞信号（线程私有）

    /* 7. 权限凭证 */
    const struct cred   *cred;        // uid/gid/capabilities 等

    /* 8. 统计信息 */
    u64                 utime;        // 用户态运行时间
    u64                 stime;        // 内核态运行时间
    u64                 start_time;   // 启动时间

    struct list_head    tasks;        // 链入全局任务链表
};
```

**关键字段详解**

**1. pid 与 tgid —— 为什么有两个？**

```text
用户态看到的 PID  = task_struct.tgid （线程组长）
内核态内部的 PID  = task_struct.pid  （每个线程唯一）

多线程进程示例：
+---------------------------+
| 主线程   pid=100 tgid=100 | <- group_leader，用户看到 PID=100
| 线程 A   pid=101 tgid=100 |
| 线程 B   pid=102 tgid=100 |
+---------------------------+
```

`getpid()` 返回 `tgid`，`gettid()` 返回 `pid`。

**2. 进程状态 __state**

```c
TASK_RUNNING          // 就绪或正在 CPU 上运行
TASK_INTERRUPTIBLE    // 可中断睡眠（等待 IO/信号）
TASK_UNINTERRUPTIBLE  // 不可中断睡眠（常为磁盘 IO）
__TASK_STOPPED        // 被 SIGSTOP 暂停
EXIT_ZOMBIE           // 已退出，等待父进程 wait()
```

这就是 `ps` 里 `S`、`R`、`D`、`Z` 状态的来源，对应 `/proc/<pid>/status` 的 `State:` 字段。

**3. 亲缘关系字段 —— 进程树的实现**

```text
init(1) ── children ──┬── sshd(800) ──┬── bash(1200)
                      │               └── bash(1300)
                      └── systemd-...
```

所有 task_struct 通过 `tasks` 字段链成全局双向链表；通过 `children/sibling` 构成树。`ps -ef` 就是遍历这条链。

**4. 资源指针 —— 进程和线程的本质区别**

线程创建时（`clone()` 带 `CLONE_VM | CLONE_FILES ...` 标志）：

```text
进程（不共享）              线程（共享）
p1->mm != p2->mm            t1->mm == t2->mm   共享地址空间 ✓
p1->files != p2->files      t1->files == t2->files  共享 fd 表 ✓
各自独立 Stack               Stack 在各自 mm 中独立分配
```

这也解释了前面聊过的：同一进程的多线程共享 Heap/全局变量，但各有各的用户态栈（栈本身是 `mm->mmap` 区域里的一段映射）。

**5. mm 与 active_mm**

```c
mm         // 用户进程持有自己的地址空间
active_mm // 内核线程没有自己的 mm，借用被切出进程的页表
```

所以内核线程（如 kworker）的 `/proc/<pid>/` 下看不到有意义的 maps——它没有自己的地址空间。

**6. sched_class 与 se —— CFS 调度**

```c
sched_class  ->  fair_sched_class (CFS) 或 rt_sched_class (实时)
se.vruntime  ->  虚拟运行时间，CFS 总挑 vruntime 最小的任务运行
```

**如何观察 task_struct 的存在**

```bash
# status 文件就是内核从 task_struct 格式化输出的
cat /proc/self/status
# Name:, State:, Pid:, PPid:, Threads:, Uid:, Gid: ...

# 每个线程一个子目录 = 每个线程一个 task_struct
ls /proc/<pid>/task/

# 内核调试器中直接查看
crash> task_struct <address>
```

**一句话总结**

> `task_struct` 是 Linux 对“执行流”的完整抽象：标识（pid/tgid）、状态、调度信息、亲缘树，以及指向各类资源的指针（`mm` 地址空间、`files` fd 表、`fs` 文件系统信息）。**进程与线程的区别不在结构体本身，而在这些指针是否与其他任务指向同一份资源**——共享即线程，独立即进程。



### **/proc 是什么**

`/proc` 是一个**虚拟文件系统**（procfs），由内核在内存中动态生成，不占用磁盘空间。它是内核向用户态暴露**进程信息**和**内核状态**的接口。文件的内容是读取时实时生成的。

```bash
$ mount | grep proc
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
```

**顶层结构：两大类内容**

```text
/proc
├── 1/          <- 数字目录：每个运行中的进程一个（PID 命名）
├── 42/
├── 1234/
│   ...
├── acpi/
├── cpuinfo     <- 非数字文件：系统全局信息和内核参数
├── meminfo
├── ...
```

**第一部分：进程目录 /proc//**

每个数字目录对应一个进程，目录名就是 PID：

| 文件/目录 | 内容                                                   |
| --------- | ------------------------------------------------------ |
| `cmdline` | 启动命令行，以 `\0` 分隔                               |
| `comm`    | 进程名                                                 |
| `status`  | 可读格式的进程状态（State、Pid、PPid、内存、线程数等） |
| `stat`    | 机器可读的进程状态（ps 的数据来源）                    |
| `environ` | 环境变量                                               |
| `cwd`     | 符号链接 → 当前工作目录                                |
| `exe`     | 符号链接 → 可执行文件的路径                            |
| `fd/`     | 该进程打开的所有文件描述符（socket、pipe、普通文件）   |
| `maps`    | 进程的内存映射区域布局（代码段、堆、栈、共享库）       |
| `smaps`   | 更详细的内存映射统计                                   |
| `task/`   | 每个线程一个子目录 `/proc/<pid>/task/<tid>/`，结构同上 |
| `net/`    | 该进程可见的网络信息（网络命名空间）                   |
| `limits`  | 进程的资源限制                                         |
| `io`      | 进程的 I/O 统计                                        |
| `stack`   | 内核栈（需 root）                                      |
| `syscall` | 当前正在执行的系统调用                                 |

常用示例：

```bash
# 查看进程启动命令
cat /proc/1234/cmdline | tr '\0' ' '

# 查看进程实际的可执行文件位置
ls -l /proc/1234/exe

# 查看进程打开的所有 fd
ls -l /proc/1234/fd

# 查看进程有几个线程
ls /proc/1234/task

# 查看进程内存布局
cat /proc/1234/maps
```

注意 `/proc/self` 是个特殊链接，始终指向**当前正在读它的那个进程**自己：

```bash
cat /proc/self/status    # 读到的就是 cat 这个进程自己的状态
```

**第二部分：系统级信息文件**

| 文件         | 内容                                |
| ------------ | ----------------------------------- |
| `cpuinfo`    | CPU 型号、核心数、频率等            |
| `meminfo`    | 内存使用情况（free 命令的数据来源） |
| `loadavg`    | 系统负载（uptime 显示的前几项）     |
| `uptime`     | 系统运行时长和空闲时间              |
| `version`    | 内核版本信息                        |
| `mounts`     | 当前挂载的文件系统                  |
| `interrupts` | 中断分配统计                        |
| `partitions` | 磁盘分区                            |
| `swaps`      | swap 使用情况                       |

网络相关子目录：

| 路径                 | 内容                                                  |
| -------------------- | ----------------------------------------------------- |
| `net/dev`            | 网卡流量统计                                          |
| `net/tcp`、`net/udp` | socket 连接表（含 inode，可与 `/proc/<pid>/fd` 对应） |
| `net/route`          | 路由表                                                |

**第三部分：可写的内核参数 /proc/sys/**

这个目录下的文件**可以写入**，用于实时调整内核行为，即 sysctl 接口：

```bash
# 允许 IP 转发
echo 1 > /proc/sys/net/ipv4/ip_forward

# 等价命令
sysctl -w net.ipv4.ip_forward=1

# 查看
cat /proc/sys/net/ipv4/ip_forward
```

常见子目录：

| 目录          | 控制                                   |
| ------------- | -------------------------------------- |
| `sys/net/`    | 网络参数（转发、TCP 缓冲区等）         |
| `sys/vm/`     | 内存管理（dirty ratio、swappiness 等） |
| `sys/kernel/` | 内核全局（hostname、pid_max 等）       |
| `sys/fs/`     | 文件系统（最大打开文件数 file-max 等） |

重启后失效；持久化配置写在 `/etc/sysctl.conf` 或 `/etc/sysctl.d/`。

**与用户态工具的关系**

很多常用工具本质上就是在读 `/proc`：

| 工具     | 数据来源                                 |
| -------- | ---------------------------------------- |
| `ps`     | `/proc/<pid>/stat`、`status`             |
| `top`    | 同上 + `/proc/stat`、`meminfo`           |
| `free`   | `/proc/meminfo`                          |
| `lsof`   | `/proc/<pid>/fd`                         |
| `ss -p`  | `/proc/net/tcp` + 各进程的 fd inode 匹配 |
| `uptime` | `/proc/loadavg`、`uptime`                |

一句话总结：

> `/proc` 是内核暴露给用户态的实时信息窗口：数字目录对应每个进程（fd、maps、status 等），非数字文件是全局系统状态，`/proc/sys/` 还能在线调内核参数——它不是真实磁盘文件，而是内核按需生成的内容。



## 2. Learn the basics of **Linux system calls** and use the `man 2` command to research `open()`, `close()`, `read()`, and `write()`.

### **`man 2` 显示的是“系统调用”的帮助手册。**

man 手册共分 9 个章节，`数字` 用于指定查哪一章：

| 章节  | 内容                           | 示例           |
| ----- | ------------------------------ | -------------- |
| 1     | 用户命令                       | `man 1 ls`     |
| **2** | **系统调用（内核提供的接口）** | `man 2 open`   |
| 3     | 库函数（C 标准库）             | `man 3 printf` |
| 4     | 特殊文件（设备文件）           | `man 4 null`   |
| 5     | 文件格式                       | `man 5 passwd` |
| 6     | 游戏                           |                |
| 7     | 杂项（协议、约定）             | `man 7 socket` |
| 8     | 系统管理命令                   | `man 8 mount`  |

**为什么要指定章节？**

因为同名条目可能存在于多个章节，不指定时按顺序显示第一个命中的：

```bash
$ man 2 open      # 系统调用 open() 的原型、参数、错误码
$ man 3 printf    # C 库函数 printf()

# 不指定章节，open 会先命中第 2 章
$ man open        # 等价于 man 2 open

# 查看某名字在哪些章节存在
$ man -f open
open (2)            - open and possibly create a file
openat (2)          - open ... relative to a directory file descriptor
```

**`man 2` 的典型内容结构**

以 `man 2 fork` 为例：

```text
FORK(2)                    Linux Programmer's Manual

NAME        fork - create a child process

SYNOPSIS    #include <unistd.h>
            pid_t fork(void);

DESCRIPTION 系统调用的功能描述

RETURN VALUE 成功/失败的返回值约定

ERRORS      可能返回的错误码（EAGAIN、ENOMEM...）

SEE ALSO    相关调用：clone(2), execve(2), wait(2)
```

**常用技巧**

```bash
man 2 socket       # 看 socket 系统调用
man 2 clone        # 看线程创建（对应前面聊的 task_struct 共享）
whatis read        # 快速列出 read 出现在哪些章节
apropos network    # 按关键词搜索手册
```

一句话总结：

> `man 2 <name>` 查看的是 `<name>` 这个**系统调用**的手册——即内核直接暴露给用户态的接口（如 `fork`、`open`、`socket`），包含函数原型、行为语义、返回值和错误码；而 `man 3` 才是 C 库函数。



## 3. Implement a custom `cat` utility in C using these basic system calls.



```C
root@DA22053684:/tmp# cat jyancat.c
#include <fcntl.h>  // open() 及 O_RDONLY 等标志
#include <unistd.h> // read()/write()/close(), STDOUT_TILENO

int main(int argc,char *argv[]) {
        // argv[0]=程序名, argv[1]=第一个参数（要打开的路径）
        int fd = open(argv[1], O_RDONLY);
        char buffer[1024];
        ssize_t bytes_read;
        while ((bytes_read = read(fd, buffer, 1024))>0) {
                ssize_t bytes_written = write(STDOUT_FILENO, buffer, bytes_read);
        }

        close(fd);
        return 0;

}
```



**argc 和 argv 是什么**

它们是 `main` 函数接收**命令行参数**的方式：

```c
int main(int argc, char *argv[])
```

| 参数   | 含义                                                  |
| ------ | ----------------------------------------------------- |
| `argc` | argument count：参数个数（含程序名本身）              |
| `argv` | argument vector：字符串指针数组，每个元素指向一个参数 |



```int fd = open(argv[1], O_RDONLY);```

man 2 open

```man 2
   int open(const char *path, int flags, ...
              /* mode_t mode */ );
              
   The argument flags must include one of the following access modes: O_RDONLY, O_WRONLY, or O_RDWR.  These request opening the file read-only, write-only, or read/write, respectively.
```



man 2 read / write

```man
NAME
       write - write to a file descriptor

LIBRARY
       Standard C library (libc, -lc)

SYNOPSIS
       #include <unistd.h>

   ssize_t write(size_t count;
                 int fd, const void buf[count], size_t count);
```

```man 2
NAME
       write - write to a file descriptor

LIBRARY
       Standard C library (libc, -lc)

SYNOPSIS
       #include <unistd.h>

       ssize_t write(size_t count;
                     int fd, const void buf[count], size_t count);


```

```man close
NAME
       close - close a file descriptor

LIBRARY
       Standard C library (libc, -lc)

SYNOPSIS
       #include <unistd.h>

   	   int close(int fd);
```

