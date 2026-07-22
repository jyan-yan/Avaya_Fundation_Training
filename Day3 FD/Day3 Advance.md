# Day3 Advance

Hi Team,

下个 Topic 开始，正式进入 Linux Kernel。


先回顾一下前两个 Topic 以及需要掌握的重点：
**1. Git**
\- Git：版本控制工具
\- GitHub/GitLab/Bitbucket：用Git管理的远程文件夹
\- VS Code：文本编辑器
\- Markdown：流行的纯文本排版语法规范

**2. GCC**
\- 源代码：给程序员看的 纯文本
\- 可执行文件：计算机可执行的 二进制文件
\- 进程：操作系统 加载 可执行文件 到 内存 后的实例
\- GCC：用于将程序变成可执行文件的工具集（包含预处理、编译、汇编、链接 4 步）

 

=====================



下一个 topic 是 **FD** (File Descriptor)，初步定在下 7/30 周四 2PM， Dalian Room



**内容：**

\- 进程在内核里是一个 C Struct

\- FD 是一个进程属性（描述打开的文件）

\- 用户程序通过 System Call 和内核通信（以获取操作系统资）

\- 用 `open`、`close`、`read`、`write` 手写一个 Linux 的 `cat`



预习链接：https://avaya.atlassian.net/wiki/spaces/DLBBEWIKI/pages/2429485226/1-3+FD



**建议补习的基础：**

1. C语言的基础数据类型，为什么要设置数据类型（从汇编的角度更好理解）
2. C语言的结构体 （如果有OOP基础，可以把class和C Struct进行对比理解）



这个topic掌握后，可以做些 **case study（**我放在 https://avaya.atlassian.net/wiki/spaces/DLBBEWIKI/pages/2429485226/1-3+FD 的comments ）：

1. 通过 C Struct 来学习 CM TCM 的输出
2. 通过基础 System Call 写 SBC logserver 进程的 copyTruncate 函数（用于 logrotate），可以自己去 Forge 上申请下 SBC (aurora) 的 code



=====================



**注意事项：**

1. GitHub 上千万不要上传任何公司源代码、客户资料等敏感信息。
2. 如果使用 AI 来读公司代码，可以使用公司的 Gemini/WebAI，绝对别用其他免费的 对话LLM，避免公司资料被拿去训练。
3. 为了降低学习复杂度，工具越简易越好，能用 CLI 就用 CLI。尽量避免使用高级工具（比如 Cursor 集成 AI 读代码、VS Code 的 Git 和 Markdown 插件），仅在必要时（或者熟悉后）才增加额外的工具/插件。
4. 这个 Training 节奏会很快，所以每个 Topic 一定要写笔记！不然过段时间忘了又要重新学，纯属折磨自己



有反馈或问题请随时ping我，谢谢。