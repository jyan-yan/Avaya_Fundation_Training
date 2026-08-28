Hi Team,

上个topic提到了一个关于AES tsrv OOM的case，我看了下tsrv的项目结构，感觉可以作为Linux 阶段的实战，把理论知识落到实际的项目代码里。

介绍下tsrv的背景：

\- tsrv (ApiServers\TSAPI\Tserver\server\tsrv) 是AES (https://scm.forge.avaya.com/svnroot/aes) 里的daemon process (/opt/mvap/bin/tsrv) 

\- 项目代码基于C++ (编译器是g++，构建工具Makefile/MPC)

\- 底层库使用开源框架ACE (ADAPTIVE Communication Environment)

\- tsrv动态加载 各种Driver（动态库，比如G3PD, /usr/lib64/libg3pd.so）

\- G3PD里有个做replication的模块 (ApiServers\TSAPI\G3pd\server\g3pd\core\rpl) ，是AES 10.2.1.0添加的，这次的OOM可能与它有关

大概的实战顺序：

1）三方框架ACE的编译和使用（完成，https://github.com/yijun-l/training/blob/main/ace/doc/1-overview.md，这个其实就是对于 "2-GCC" 章节的实战）

2）结合 设计模式 理解ACE提供的API

3）写tsrv进程，然后利用ACE API 进行重构

4）写基础的G3PD，并编译成so

5）tsrv动态加载G3PD

6）实现G3PD的replication功能

实战代码 一切从简，忽略一切优化、异常处理的部分，照着源码照猫画虎就行。通过和AI聊天，搞懂了自己写的内容，也就搞懂了源码。

大家看怎么样？要是觉得难的话，也可换成之前提到的连接池（https://github.com/yijun-l/conn_pool）