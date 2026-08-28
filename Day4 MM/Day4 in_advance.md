Hi Team,

看 GitHub 上提交 3-FD note/code 的人不多，今天就不讲了。下次 Training 计划在下周四（8/13）
要是觉得吃力，只做 Practice 部分就行，在AI的帮助把自己提交到github的内容搞懂就行，不用卷细节。

Linux 这个章节我设计的topic不多，顺序大概是：FD > MM > Process > Signal > Network > Epoll > Multi-threading （目前更到了 Process，后面确实会难一点），最后手写个线程池或连接池就算结束。

这块搞懂了，再看 CM/CMS/SM-Asset/EP/IPO 这些项目代码会轻松一大截，大家加油！

**3-FD 要掌握的**

1. 进程的本质是一个task_struct结构体
2. FD 是进程用来找打开文件的引用 (handler)
3. 用户程序靠 System Call 跟Linux内核打交道
4. 写一个简单的cat

**4-MM 要预习的** https://avaya.atlassian.net/wiki/spaces/DLBBEWIKI/pages/2448850963/1-4+MM

1. 物理内存 vs 虚拟内存
2. 内核空间 vs 用户空间
3. 进程的虚拟内存结构
4. Memory Leak 和 Out of Memory