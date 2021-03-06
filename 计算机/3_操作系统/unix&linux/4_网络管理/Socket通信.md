
## 网络协议：Networking Protocol

    1、OSI七层模型：物理层、数据链路层、网络层、传输层、会话层、表示层、应用层
    
    2、业界标准TCP/IP模型（5层）：物理层、MAC层、IP层、传输层、应用层

    IP地址具有全局定位功能，而MAC地址只具有局部定位功能，类比外卖场景中的收货地址和手机尾号
    
### TCP与UDP
    TCP 是面向连接的，UDP 是面向无连接的。
    TCP 提供可靠交付，无差错、不丢失、不重复、并且按序到达；UDP 不提供可靠交付，不保证不丢失，不保证按顺序到达。
    TCP 是面向字节流的，发送时发的是一个流，没头没尾；UDP 是面向数据报的，一个一个的发送。
    TCP 是可以提供流量控制和拥塞控制的，既防止对端被压垮，也防止网络被压垮。
    
    从本质上来讲，所谓的`建立连接`，其实是为了在客户端和服务端维护连接，而建立一定的数据结构来维护双方交互的状态，并用这样的数据结构来保证面向连接的特性。
    TCP 无法左右中间的任何通路，也没有什么虚拟的连接，中间的通路根本意识不到两端使用了 TCP 还是 UDP。
    
### Socket定义
    
    操作系统内核态处理2、3、4层网络协议，用户态处理第5层应用层，内核态和用户态通过端口建立连接，通过Socket系统调用完成交互。
    所以Socket不属于网络协议特定的哪一层，而是操作系统内核态和用户态的通信机制。当然，Socket主要工作在传输层。
    
    int socket(int domain, int type, int protocol);
    返回文件描述符fd
    domain：表示使用什么IP协议 AF_INET 表示 IPv4，AF_INET6 表示 IPv6。
    type：表示 socket 类型。SOCK_STREAM 面向流的TCP；SOCK_DGRAM 面向数据报的UDP；
          SOCK_RAW 可以直接操作IP层，或者非TCP和UDP的协议，例如ICMP。
    protocol：表示协议，包括 IPPROTO_TCP、IPPTOTO_UDP。
    
### socket函数、bind函数、listen函数、accept函数、connect函数    