# 1-5 Process



# Process

Based on our previous topic, we know that:

- A program is a static text file written in a programming language for human readers.
- An executable file is a compiled, static file on disk (usually in ELF format).
- A process is a dynamic, living instance of an executing program in memory.
- Inside the kernel, a process is represented by a `task_struct` structure.

## The Parent-Child Structure

In Linux, processes do not exist in isolation. Instead, they are organized in a strict hierarchical tree structure where every process has a parent and can have zero or more children.

At the root of this entire process tree is **PID 1** (usually `init` or `systemd`), which is directly or indirectly the ancestor of every process on the system. This family relationship is established the moment a parent process creates a child (typically via system calls like `fork()` or `clone()`).

### Core attributes

To manage this complex tree, the kernel tracks identification and family pointers within the `task_struct`:

```C
// https://elixir.bootlin.com/linux/v4.18/source/include/linux/sched.h#L593 
struct task_struct {    
    /* --- Identification --- */ 
    pid_t                       pid;            /* Thread ID (Unique in the kernel) */ 
    pid_t                       tgid;           /* Thread Group ID (What user space sees as "PID") */ 
    
    /* --- Family Relationship Pointers --- */ 
    struct task_struct __rcu    *real_parent;   /* Points to the process that created it */ 
    struct task_struct __rcu    *parent;        /* Points to the parent that receives signals like SIGCHLD */ 
    
    struct list_head            children;       /* Head of the list containing all of this process's children */ 
    struct list_head            sibling;        /* Node linking this process to its parent's children list */ 
};
```

### Visualizing the Tree

You can view this exact parent-child hierarchy in a running system using the `ps` command with the `--forest` parameter.

```shell
$ ps -eo pid,ppid,cmd --forest    
	PID    PPID CMD      
	  1       0 /usr/lib/systemd/systemd --system --deserialize=138 
3517258       1 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups 
3861456 3517258  \_ sshd: dadmin [priv] 
3861481 3861456      \_ sshd: dadmin@pts/0 
3861482 3861481          \_ -bash 
3861638 3861482              \_ ps -eo pid,ppid,cmd --forest
```

## Process State

Once a process is part of the tree, it moves through several states during its lifecycle. The kernel uses these states to track what the process is currently doing.

### Core States

- `TASK_RUNNING (R)`: The process is either currently executing on the CPU or sitting in the run queue waiting for its turn.
- `TASK_INTERRUPTIBLE (S)`: The process is sleeping, waiting for an event or resource (like user input). It can be awakened by system signals.
- `TASK_UNINTERRUPTIBLE (D)`: The process is in a deep sleep, typically waiting for hardware I/O or kernel locks. It ignores signals and cannot be killed (even with kill -9).
- `EXIT_ZOMBIE (Z)`: The process has finished executing, but its parent has not yet read its exit status to release its remaining resources.

### Concepts

The state transitions of a process are driven by creation, interrupts, and termination. These events often introduce specific concepts:

- **Copy-on-Write (COW)**: During creation (`fork()`), the process enters the `R` state. To optimize performance, Linux does not immediately copy the parent's memory. Instead, both share the same pages in read-only mode. Memory is only duplicated when either process attempts to write to it.
- **Zombie Processes**: When a child calls `exit()`, it transitions to the `Z` state. It releases memory but keeps its `task_struct` so the parent can collect its "death certificate" via `wait()`. If the parent fails to do this, the child remains a Zombie.
- **Orphan Processes**: If a parent dies before its child, the child becomes an "Orphan." The kernel automatically reparents orphans to PID 1, which takes on the responsibility of reaping them when they eventually exit.

### State Attributes in Code

```c
// https://elixir.bootlin.com/linux/v4.18/source/include/linux/sched.h#L593 
struct task_struct {    
    volatile long               state;          /* Current execution state (R, S, D, etc.) */    
    int                         exit_state;     /* Exit state (e.g., EXIT_ZOMBIE) */    
    int                         exit_code;      /* The return value passed to exit() */    
    int                         exit_signal;    /* Signal sent to parent upon death */ 
};
```

## The Process Scheduler

When multiple processes are in the `TASK_RUNNING` state, the Process Scheduler decides which one gets to use the CPU and for how long. The kernel uses different algorithms based on the process type:

- **Normal Processes (CFS)**: Managed by the **Completely Fair Scheduler (CFS)**. CFS uses a Red-Black Tree to organize processes according to their `vruntime` (virtual runtime). The CPU always picks the leftmost node—the process that has received the least CPU time.
- **Real-Time Processes**: Used for time-sensitive tasks requiring low latency. They run under `SCHED_FIFO` (First-In, First-Out) or `SCHED_RR` (Round-Robin). Real-time processes always preempt normal processes.

### Scheduling Attributes in Code

```c
// https://elixir.bootlin.com/linux/v4.18/source/include/linux/sched.h#L593 
struct task_struct {    
    int                         prio;           /* Dynamic scheduling priority */    
    int                         static_prio;    /* Base priority for normal processes (derived from nice value) */    
    int                         normal_prio;    /* Priority based on static_prio and scheduling policy */    
    unsigned int                rt_priority;    /* Real-time priority */    
    unsigned int                policy;         /* Scheduling policy (e.g., SCHED_NORMAL, SCHED_FIFO) */    
    const struct sched_class    *sched_class;   /* Pointer to the specific scheduler class handling this task */ 
};
```

### Understanding Process Priority

Internally, the Linux kernel uses a priority scale from 0 to 139:

- **Real-Time processes**: 0 to 99 (Lower number = higher priority).
- **Normal processes**: 100 to 139 (Default is 120).

For normal processes, users can adjust the priority using the **NICE (NI)** value, which ranges from -20 to 19. (**Priority** = 120 + NICE).

You can monitor scheduling policies and priorities using tools like `top` and `chrt`:

```shell
$ top    
	PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND 
3507252 root      20   0 1734536 477680  75892 S   9.2   5.9     27,28 kube-apiserver     
	 49 root     -51   0       0      0      0 S   0.0   0.0   0:00.00 watchdogd 
	 
$ chrt -p 3507252 
pid 3507252's current scheduling policy: SCHED_OTHER 
pid 3507252's current scheduling priority: 0 

$ chrt -p 49 
pid 49's current scheduling policy: SCHED_FIFO 
pid 49's current scheduling priority: 50 

```

## System Calls

The life cycle of a process is driven by three essential system calls:

- `fork()`: Creates a new child process by duplicating the parent's `task_struct`. Instead of copying physical memory immediately, the parent and child share the same physical memory pages. The memory is only duplicated when one of the processes attempts to write to it. Upon creation, the child's `real_parent` pointer is set to point to the parent.
- `execlp()`: Replaces the current process's entire memory space (including its code, data, heap, and stack) with a new executable program. It is a convenient library wrapper around the raw `execve()` system call that automatically searches your system's `PATH`.
- `waitpid()`: Called by the parent process to read the child's exit status. This is crucial for reaping Zombie processes. Once the parent retrieves this status, the kernel finally deletes the child's task_struct from memory.

```c
#include <stdio.h> 
#include <unistd.h> 
#include <sys/types.h> 
#include <sys/wait.h> 

int main() {    
    printf("[PARENT] PID is %d\n", getpid());    
    pid_t pid = fork();    
    if (pid == 0) {        
        printf("[CHILD]  PID is %d, PPID is %d.\n", getpid(), getppid());  
        
        execlp("ls", "ls", "-lahtr", NULL);    
    } else {        
        printf("[PARENT] Created child PID %d.\n", pid);        
        
        int status;        
        pid_t reaped_pid = waitpid(-1, &status, 0);        
        
        printf("\n[PARENT] Child %d has exited. (Raw status: %d)\n", reaped_pid, status);        
        printf("[PARENT] Done.\n");    
    }    
    return 0; 
}
```

## Practice

1. Understand the process state machine (R, S, D, Z) and the family tree concept.
2. Explore the `struct task_struct` fields, paying special attention to `pid`, `state`, and `real_parent`.
3. Understand how CFS uses a Red-Black tree and `vruntime` to achieve fair scheduling, and contrast it with real-time process scheduling.
4. Implement the process lifecycle example in C.
5. Summarize your learnings into technical notes, and push both the code and notes to GitHub.