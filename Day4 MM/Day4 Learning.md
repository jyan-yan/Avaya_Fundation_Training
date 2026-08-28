# Virtual Memory

一个进程的地址分配用虚拟内存Virtual Memory。虚拟内存大小与实际内存无关，与操作系统和CPU位数有关。

**32 位 Linux（x86）**

```text
0x00000000 ────────────── 4 GB 总空间 ────────────── 0xFFFFFFFF
+------------------------+-----------------------------------+
|   用户空间 0 ~ 3GB      |       内核空间 3GB ~ 4GB          |
|   （进程私有）          |   （所有进程共享同一份内核映射）   |
+------------------------+-----------------------------------+
              TASK_SIZE = 0xC0000000 分界（默认 3G/1G）
```

**64 位 x86_64（4 级页表，48-bit 虚拟地址，最常见）**

理论 64 位只有低 48 位有效，且地址分上半/下半，中间是“非规范区”空洞：

```text
0x0000000000000000
        ├──────────── 用户空间（进程可用）：0 ~ 0x00007FFFFFFFFFFF
        │             大小 = 128 TB
        │
0x00007FFFFFFFFFFF  ── 上半不可寻址（canonical 空洞）
0xFFFF800000000000
        │
        ├──────────── 内核空间（全进程共享）：0xFFFF800000000000 ~ 0xFFFFFFFFFFFFFFFF
        │             大小 = 128 TB
0xFFFFFFFFFFFFFFFF
```

要点：

- **用户空间上限 = `0x00007FFFFFFFFFFF`**（128 TB 减 1），这才是进程“自己”能用的范围
- **内核空间高半区**也出现在该进程的页表里，但普通用户代码不能访问，是系统共用的同一份映射
- 中间的空洞是地址格式规定（canonical address），访问它会触发段错误 Segment Fault

**新 CPU 开了 5 级页表（LA57，57-bit）**

```text
用户空间扩展到 0x00FFFFFFFFFFFFFF（128 PB 级）
内核空间 = 0xFF00000000000000 ~
```

**怎么看当前系统**

```bash
# 是多少位
getconf LONG_BIT                 # 64

# 进程实际地址区间（只展示用户空间布局，内核部分不显示）
cat /proc/self/maps

# 地址位数 / 是否 LA57
grep -o 'la57' /proc/cpuinfo
cat /proc/sys/kernel/osrelease   # 结合 arch 判断

# 用户空间典型最高地址示例
cat /proc/self/maps | tail -1    # 末尾一般是 vdso/stack，接近 0x7fff...
```

`/proc/self/maps` 看到的都是 **0x…~0x7ffff…** 范围内的段（代码、堆、库、栈、mmap），印证用户空间封顶在 `0x00007FFFFFFFFFFF`。

**内存布局（用户空间内部，128 TB 内的排布）**

```text
低地址
0x0000000000000000
├── 代码段 / 数据段（程序本身）
├── 堆（向上增长）
├── 内存映射区 mmap（库、文件映射，向下增长）
├── 栈 stack（靠近 0x7fff... 顶端，向下增长）
└── vdso / vsyscall
高地址 0x00007FFFFFFFFFFF
```

**重要澄清：范围固定 ≠ 用满**

- 地址“范围”由硬件/内核决定（上面这些值）
- 但进程实际只占其中一小块，`/proc/<pid>/maps` 显示的就是它在 128 TB 里具体用到的片段
- ASLR 会让各段起始位置随机化，但都落在这个固定区间内

一句话总结：

> 32 位进程可用 `0 ~ 3GB`（默认 3G/1G 划分），64 位 x86_64 进程用户空间为 `0x0000000000000000 ~ 0x00007FFFFFFFFFFF`（128 TB），内核高半区虽映射进页表但不可被用户态访问、且所有进程共享。



# Segmentation fault

**本质：MMU 拦截了一次非法访问**

CPU 访问内存时，MMU 查页表。如果出现下面情况之一，硬件触发**页错误**，内核判断无法补救就给进程发 `SIGSEGV`：

```text
1. 地址根本没映射（没有对应的页表项 / VMA）
     → 空指针解引用、野指针、已 free 的内存
2. 权限不对
     → 写只读区（代码段、字符串常量）、执行不可执行页、用户态访问内核空间
3. 栈溢出撞上 guard page（栈底的保护页，未映射）

int *p = NULL;
*p = 5;              // 解引用 0x0 → 无映射 → SIGSEGV

char *s = "hello";
s[0] = 'H';          // 写只读 rodata → SIGSEGV

int a[4];
a[1000000] = 1;      // 越界到未映射区 → 可能 SIGSEGV（也可能踩到别人）
```



**Segmentation Fault和OOM的区别：** 

```text
“空间不够”   → 分配失败 / 被 OOM 杀（SIGKILL）
“访问非法地址”→ SIGSEGV（与剩余空间多少无关，哪怕地址空间几乎全空也会崩）
```



# Kernal Space in Virtual Memory

内核空间那部分虚拟地址，指向的是**系统级的物理内存/设备资源**，可以分成几类映射。核心思想和用户空间“每个进程私有一份”完全不同：内核空间是**全系统共享、统一映射**。

**一、最大的部分：物理内存的直接映射（direct map / physmap）**

64 位内核会把**整块物理 RAM 线性映射**进内核空间：

```text
内核虚拟地址 = 物理地址 + PAGE_OFFSET

例如物理页框 pa=0x12340000
  → 内核可直接访问 va = 0xffff880012340000
内核空间（x86_64 概念布局）
0xffff800000000000 ┌─────────────────────────────┐
                  │ 直接映射区 (physmap)          │
                  │  把全部物理 RAM 线性搬进来     │
                  │  内核代码、task_struct、       │
                  │  页帧、slab 缓存、页表本身…   │
                  │  都在这片物理内存里，         │
                  │  透过 direct map 访问         │
                  ├─────────────────────────────┤
                  │ vmalloc / ioremap 区          │
                  │  内核模块、非连续内核分配、    │
                  │  设备寄存器 MMIO 映射         │
                  ├─────────────────────────────┤
                  │ vmemmap (struct page 数组)     │
                  ├─────────────────────────────┤
                  │ 内核镜像 text/data/bss         │
                  └─────────────────────────────┘
```

意义：内核要操作任何物理页（分配内存、改页表、访问某进程的用户页），都先通过这片直接映射找到它——这是内核能管理全系统内存的根本。

**二、内核空间具体指向哪些“内容”**

| 内核空间映射       | 指向的物理内容                                               |
| ------------------ | ------------------------------------------------------------ |
| **直接映射区**     | 全部物理 RAM：内核代码/数据、所有进程的页帧、task_struct、slab、页表本身 |
| **内核镜像段**     | 内核自己的 `.text`/`.data`/`.bss` 所在的物理页               |
| **内核栈**         | 每个 task 的 16KB 内核栈（位于物理 RAM，经 direct map 访问） |
| **vmalloc 区**     | 内核非连续虚拟分配：可装载模块、`ioremap` 来的设备 I/O 内存  |
| **ioremap / MMIO** | 映射到**设备寄存器**（不是 RAM），用于驱动操作网卡/显卡等硬件 |
| **vmemmap**        | `struct page` 数组（每物理页一个描述符），也是物理 RAM 里的一块 |
| **fixmap / vDSO**  | 固定映射、给用户态看时间用的内核页                           |

**三、和用户空间“指向什么”对比**

```text
用户空间 va：
  → 映射到“本进程私有”的物理页
  → 来源：匿名页(堆/栈) 或 文件映射(代码/库)
  → 不同进程同一 va 指向不同物理页

内核空间 va：
  → 映射到“系统共享”的物理 RAM（或直接映射全部物理内存）
  → 同一 va 在所有进程页表里都指向同一物理页
  → 外加 MMIO：映射到设备寄存器（非 RAM）
```

**四、一个关键推论（呼应前面的 task_struct）**

前面说 `task_struct` 在内核里。具体说：它被分配在**物理 RAM 的某页**，内核通过 direct map（或 slab 的虚拟地址）访问它。所有进程的内核栈、页表、文件缓存也都在物理 RAM 里，统一被这片内核空间窗口覆盖。

**五、32 位下的对应（呼应 1GB 问题）**

```text
0xC0000000 起的内核 1GB 窗口：
  ├─ 0xC0000000 + 物理地址  → 低 896MB 物理 RAM 的直接映射
  ├─ vmalloc 区            → 模块、ioremap
  ├─ kmap 临时窗口         → 访问 HIGHMEM(>896MB) 用
  └─ 内核镜像 / fixmap
```

所以“内核空间 1GB”里，真正**永久**直接映着物理 RAM 的只有约 896MB（lowmem），其余靠 vmalloc/kmap 临时用。

一句话总结：

> 内核空间虚拟地址主要指向**整个系统的物理内存（经 direct map 线性映射）**、内核自身镜像、各任务的内核栈/ slab / 页表，以及通过 `ioremap` 映射的**设备寄存器**。它和用户空间的本质区别是：用户空间映射到“进程私有页”，内核空间映射到“全系统共享的物理 RAM + 硬件 MMIO”，且所有进程看到的内核映射完全一致。



# mmap

**Memory-Mapped Region**: Used for loading shared libraries and huge files (via `mmap`).

`mmap` 是操作系统**虚拟内存管理**的一部分（POSIX/Linux 系统调用，Windows 对应 `CreateFileMapping`/`MapViewOfFile`）。

**它做什么**：把文件或设备直接映射到进程的**虚拟地址空间**，让文件像内存数组一样被读写，省去 `read`/`write` 的拷贝开销。

**存储/映射的信息**：

- 磁盘文件内容（最常见，如大文件、共享库 `.so`）
- 匿名内存（无文件背景，用作共享内存 `MAP_ANONYMOUS`）
- 设备寄存器/硬件（设备文件映射）
- 进程间共享内存（`MAP_SHARED`）

**关键特点**：

- 按需分页（page fault 时才真正读磁盘）
- 多个进程 `MAP_SHARED` 映射同一文件可共享内存
- 修改可由 `msync` 刷回磁盘，或依赖内核自动回写

和你在做的训练相关的典型用途：加载大模型权重、深度学习数据集（`torch.utils.data` 底层常用 mmap 提速）。



**mmap可以通过mmap()的system call进行调用**

 **调用系统调用（C/Unix）：**

```c
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

int fd = open("file.bin", O_RDONLY);
struct stat st; fstat(fd, &st);

void *addr = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
// addr 现在可直接当字节数组访问，如 ((char*)addr)[0]
// 用完：
munmap(addr, st.st_size);
close(fd);
```

**只是磁盘文件的映射，磁盘上文件内容并不一次性全部拷贝到物理内存。**

- 访问 `addr[i]` 时触发 **page fault**，内核才把对应 4KB 页从磁盘读入物理内存。
- 多个虚拟页可指向同一份物理页（共享）；`MAP_PRIVATE` 写时会触发写时复制（COW）。
- 所以"内容"对进程看来就是文件内容，但物理内存只在访问到的部分才真正驻留。



# task_struct和mm_struct

## `task_struct`（节选中）

```c
struct mm_struct      *mm;     // 指向该进程的内存描述符（地址空间）
struct files_struct  *files;  // 指向该进程打开的文件描述符表
```

- 这是进程的"总控块"。一个进程的运行所需的关键资源都以指针形式挂在这里：内存(`mm`)、文件(`files`)、信号处理、PID、状态等。

## `mm_struct`（节选中）

```c
struct vm_area_struct *mmap;   // VMA 链表：描述地址空间里每一段[起始,结束)区域的属性 对应mmap
pgd_t *pgd;                    // 页表顶级指针（Page Global Directory），MMU 寻址根
atomic_t mm_users;             // 引用计数：多少个线程/进程共享此地址空间（clone VM）
unsigned long start_code, end_code;   // 代码段(text)的起止虚拟地址
unsigned long start_data, end_data;   // 已初始化数据段(data)的起止虚拟地址
unsigned long start_brk, brk;         // 堆的起始地址 与 当前堆顶（brk() 系统调用扩展它）
unsigned long start_stack;            // 用户栈的起始地址（栈向低地址增长，这实际是栈底/最高地址）
```



# System Call / Library Call

## 1. Library call 在 link 后去哪了？

**静态链接 (`-static`)：你说得对** 库函数的机器码被**直接复制**进可执行文件的 `.text`。运行时全在进程自己 text 段里。

**动态链接（gcc 默认）：你说得不对** 你的可执行文件里**没有**库函数的机器码，只有：

- 一个 **PLT（过程链接表）stub** + 动态符号表里的一条**引用**。
- 真正的库函数代码在磁盘的 `libc.so` 里。

运行时，加载器 `ld.so` 把 `libc.so` **mmap 映射**进进程虚拟地址空间（形成一个带执行权限的 VMA，挂在 `mm_struct` 下）。所以库函数代码**也在进程地址空间里、也在某个 text 区**，但那是 **libc.so 自己的 text，不是你 exe 文件的 text**——多个进程共享同一份物理页。

```text
进程虚拟地址空间（mm_struct 管理）
┌─────────────────────┐
│ 你的 exe 的 .text    │  ← 只有 main 和你自己写的函数
├─────────────────────┤
│ libc.so 的 .text     │  ← open/read/mmap wrapper 实际在这里（mmap 进来的）
├─────────────────────┤
│ 堆 / 栈 / mmap 区    │
└─────────────────────┘
```

## 2. System call 呢？

一次 `mmap()` 实际分两段，分属两个空间：

```text
用户空间 (ring 3, 进程的 text)
   glibc 的 mmap wrapper:
       mov rax, 9        ; __NR_mmap
       syscall           ; ← 这条指令在用户态 text 里
   ────────── 陷入(trap)，CPU 切到 ring 0 ──────────
内核空间 (ring 0, 内核自己的内存，普通进程用户态不可访问)
   entry_SYSCALL_64 → sys_mmap()  ← 真正建 VMA 的代码在这
       返回 → 回到用户态
```

要点：

- **`syscall` 指令本身**（trap 的那一下）在用户态 text（libc 或 exe 的 text 里）。
- **内核里的真正处理程序 `sys_mmap`** 在**内核空间**，不属于任何进程的用户态地址空间，用户态代码读不到也跳不进去（靠特权级隔离）。
- 所以"system call 的执行机器语言在 text 出现"——只是指**用户态那层薄 wrapper + trap 指令**在你进程 text 里；内核干活的机器码在 kernel image 里，和你的 exe 无关。

## 一句话总结

- Library call：动态链接下代码在 `libc.so` 的 text（运行时 mmap 进来），不在你 exe 里。
- System call：用户态 wrapper（含 `syscall` 指令）在进程 text；内核侧处理程序在**内核空间**，与进程地址空间隔离。
