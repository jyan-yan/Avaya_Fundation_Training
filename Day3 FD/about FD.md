## FD: file description



FD located in /proc/xxx/fd/.

解释：

- `/proc/<pid>/fd` 目录下列出了该进程打开的**所有文件描述符（fd）**
- 每个条目是一个符号链接，链接目标描述了 fd 背后的资源类型
- `socket:[123456]` 表示这个 fd 是一个 **socket**，方括号里的数字是该 socket 的 **inode 号**
- 类似的还有 `pipe:[...]`、`anon_inode:[eventpoll]`、`/var/log/syslog`（普通文件）等

这体现了 Linux 的设计哲学：**一切皆文件**。进程对网络连接、管道的读写，和对普通文件的读写一样，都统一用 `fd` 来表示。



## Socket

**什么是 Socket？**

Socket（套接字）是**进程间通信的一个端点**。可以把它理解为“通信的插口”：

```text
进程 A                          进程 B
+--------+                      +--------+
|  fd 3  | ----> [socket] <====> [socket] <---- | fd 4 |
+--------+      内核缓冲区        +--------+
```

两个进程各自持有 socket 的 fd，通过内核在它们之间传递数据。

**Socket 的两大类**

| 类型                       | 用途                   | 例子                                              |
| -------------------------- | ---------------------- | ------------------------------------------------- |
| **Unix Domain Socket**     | 同一台主机内的进程通信 | `/run/systemd/journal/stdout`、X11、Docker daemon |
| **网络 Socket（TCP/UDP）** | 跨主机的网络通信       | `192.168.1.10:22`、浏览器访问网页                 |

**为什么进程会打开很多 socket？**

一个进程每建立一个连接就多一个 fd。比如 nginx：

- 监听端口需要 1 个监听 socket（如 `0.0.0.0:80`）
- 每接受一个客户端连接，就新增 1 个已连接 socket
- 和上游服务（如 PHP-FPM）通信又是一批 socket

所以高并发服务器进程的 `/proc/<pid>/fd` 下会有大量 `socket:[inode]` 链接。

**如何查看这些 socket 的详细信息**

```bash
# 查看系统所有 socket（-p 显示进程）
sudo ss -tunap

# 用 lsof 反查某进程打开了哪些 socket
sudo lsof -p <pid> -i

# 通过 inode 找到对应连接（/proc/net/tcp 第三列是 inode）
cat /proc/net/tcp | grep -i <inode的16进制>
```

例如：

```bash
$ sudo ss -tunp | head -5
Netid State  Local Address:Port   Peer Address:Process
tcp   LISTEN 0.0.0.0:22           users:(("sshd",pid=800,fd=3))
tcp   ESTAB  10.0.0.5:22          10.0.0.100:51234 users:(("sshd",pid=1201,fd=4))
```

这里的 `fd=3`、`fd=4` 就和 `/proc/<pid>/fd/3 -> socket:[...]` 一一对应。

**一句话总结**

> `/proc/<pid>/fd` 中指向 `socket:[inode]` 的链接，表示该进程打开了一个套接字——它是进程进行网络通信或本机进程间通信的端点。Linux 把它抽象成文件，所以也以 fd 的形式出现在这里。



## Example:

ss -tunp | grep 22 

tcp   ESTAB      0      128    10.130.130.111:22       10.220.31.7:43414 users:(("sshd",**pid=3937262,fd=4)**,("sshd",pid=3935630,fd=4)) 



```shell
root >ls -ltrha
total 0
dr-xr-xr-x. 9 init susers  0 Aug 25 11:39 ..
l-wx------. 1 root root   64 Aug 25 11:39 9 -> 'pipe:[1015003624]'
lr-x------. 1 root root   64 Aug 25 11:39 8 -> 'pipe:[1015003624]'
l-wx------. 1 root root   64 Aug 25 11:39 7 -> /run/systemd/sessions/39118.ref
lrwx------. 1 root root   64 Aug 25 11:39 6 -> 'socket:[1011242261]'
lrwx------. 1 root root   64 Aug 25 11:39 5 -> 'socket:[1015037429]'
lrwx------. 1 root root   64 Aug 25 11:39 4 -> 'socket:[1011025278]'  <------
lrwx------. 1 root root   64 Aug 25 11:39 3 -> 'socket:[1015037426]'
lrwx------. 1 root root   64 Aug 25 11:39 2 -> /dev/null
lrwx------. 1 root root   64 Aug 25 11:39 15 -> /dev/ptmx
lrwx------. 1 root root   64 Aug 25 11:39 14 -> /dev/ptmx
lrwx------. 1 root root   64 Aug 25 11:39 10 -> /dev/ptmx
lrwx------. 1 root root   64 Aug 25 11:39 1 -> /dev/null
lrwx------. 1 root root   64 Aug 25 11:39 0 -> /dev/null
dr-x------. 2 root root    0 Aug 25 11:39 .
root >pwd
/proc/3937262/fd
```





## Socket 的创建：从系统调用开始

创建一个 socket 最核心的就是 `socket()` 系统调用：

```c
int fd = socket(domain, type, protocol);
```

三个参数：

| 参数       | 含义     | 常用值                                                 |
| ---------- | -------- | ------------------------------------------------------ |
| `domain`   | 地址族   | `AF_INET`（IPv4）、`AF_INET6`、`AF_UNIX`（本机进程间） |
| `type`     | 通信类型 | `SOCK_STREAM`（TCP）、`SOCK_DGRAM`（UDP）              |
| `protocol` | 具体协议 | 通常填 `0`（由前两项决定）                             |

返回值就是一个**文件描述符**——这就是你在 `/proc/<pid>/fd` 里看到 `socket:[...]` 的来源。

**TCP 服务端的完整流程**

```text
socket()  创建 socket，得到 fd
   |
bind()    绑定 IP + 端口（如 0.0.0.0:8080）
   |
listen()  标记为监听状态，建立等待队列
   |
accept()  阻塞等待客户端连接，成功后返回一个新的 fd
```

C 语言示例：

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

int main() {
    // 1. 创建 socket，得到 fd=3
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;   // 监听所有网卡
    addr.sin_port = htons(8080);

    // 2. 绑定端口
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));

    // 3. 开始监听
    listen(listen_fd, 128);

    // 4. 接受连接，每个连接返回一个新 fd
    while (1) {
        int conn_fd = accept(listen_fd, NULL, NULL);
        // conn_fd 就是一个新的 socket，对应 /proc/<pid>/fd 下的新条目
        write(conn_fd, "hello\n", 6);
        close(conn_fd);
    }
}
```

**TCP 客户端流程**

```c
// 1. 创建 socket
int fd = socket(AF_INET, SOCK_STREAM, 0);

struct sockaddr_in serv = {0};
serv.sin_family = AF_INET;
serv.sin_port = htons(8080);
inet_pton(AF_INET, "192.168.1.10", &serv.sin_addr);

// 2. 发起连接（完成 TCP 三次握手）
connect(fd, (struct sockaddr*)&serv, sizeof(serv));

// 之后 read()/write()/send()/recv() 都通过这个 fd 收发数据
```

**Unix Domain Socket 的创建**

只是把地址族换成 `AF_UNIX`，绑定的不再是 IP+端口，而是文件系统路径：

```c
int fd = socket(AF_UNIX, SOCK_STREAM, 0);

struct sockaddr_un addr = {0};
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/my.sock");

unlink("/tmp/my.sock");      // 先删除旧文件
bind(fd, (struct sockaddr*)&addr, sizeof(addr));
```

这就是为什么你在 `/run/` 下能看到 `.sock` 文件。

**内核里发生了什么**

调用 `socket()` 时，内核大致做这些事：

```text
1. 分配 struct socket / struct sock 结构（内核中表示套接字）
2. 关联到具体的协议栈操作函数（TCP/UDP/Unix）
3. 分配一个 inode 号（就是 socket:[123456] 里那个数字）
4. 在进程的文件描述符表中占用一个空槽位
5. 把 fd 返回给用户态
```

所以每个 socket 在内核里也是一个“文件”，有自己的 inode，可以通过：

```bash
cat /proc/<pid>/fd          # 查看进程打开的 socket fd
cat /proc/net/tcp           # 查看 TCP socket 表（含 inode）
sudo ss -tunap              # 更友好的查看方式
```

**关键点：监听 socket 和连接 socket 是两个东西**

- `bind/listen` 用的是**监听 socket**，只负责“接电话”
- 每次 `accept()` 成功，内核**新建一个已连接 socket** 并返回新 fd，负责和这个特定客户端通信
- 所以高并发服务器上 `/proc/<pid>/fd` 里会有大量 `socket:[...]`：1 个监听 + N 个连接

一句话总结：

> 调用 `socket()` 即创建一个套接字并得到 fd；服务端再经过 `bind → listen → accept`，客户端经过 `connect`，之后双方就通过各自的 fd 用 `read/write` 像读写文件一样通信。