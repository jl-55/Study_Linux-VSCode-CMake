# Linux教程4——套接字通信

出处: https://subingwen.cn/linux/

# 一、套接字——Socket

## 1.1 概念

- 局域网和广域网

  - 局域网：局域网将一定区域内的各种计算机、外部设备和数据库连接起来形成计算机通信的私有网络。
  - 广域网：又称**广域网**、**外网**、**公网**。是连接不同地区局域网或城域网计算机通信的远程公共网络。

- IP（Internet Protocol）：本质是一个整形数，用于表示计算机在网络中的地址。IP协议版本有两个：IPv4和IPv6

  - IPv4（Internet Protocol version4）：

    - 使用一个32位的整形数描述一个IP地址，4个字节，int型
    - 也可以使用一个点分十进制字符串描述这个IP地址： `192.168.247.135`
    - 分成了4份，每份1字节，8bit（char），最大值为 255
      - `0.0.0.0` 是最小的IP地址
      - `255.255.255.255`是最大的IP地址
    - 按照IPv4协议计算，可以使用的IP地址共有 2^32^ 个

  - IPv6（Internet Protocol version6）：

    - 使用一个128位的整形数描述一个IP地址，16个字节
    - 也可以使用一个字符串描述这个IP地址：`2001:0db8:3c4d:0015:0000:0000:1a2f:1a2b`
    - 分成了8份，每份2字节，每一部分以16进制的方式表示
    - 按照IPv6协议计算，可以使用的IP地址共有 2^128^ 个

  - 查看IP地址

    ```shell
    # linux
    $ ifconfig
    
    # windows
    $ ipconfig
    
    # 测试网络是否畅通
    # 主机a: 192.168.1.11
    # 当前主机: 192.168.1.12
    $ ping 192.168.1.11     # 测试是否可用连接局域网
    $ ping www.baidu.com    # 测试是否可用连接外网
    
    # 特殊的IP地址: 127.0.0.1  ==> 和本地的IP地址是等价的
    # 假设当前电脑没有联网, 就没有IP地址, 又要做网络测试, 可用使用 127.0.0.1 进行本地测试
    ```

- 端口

  端口的作用是定位到主机上的某一个进程，通过这个端口进程就可以接受到对应的网络数据了。

  > 比如: 在电脑上运行了微信和QQ, 小明通过客户端给我的的微信发消息, 电脑上的微信就收到了消息, 为什么?
  >
  > - 运行在电脑上的微信和QQ都绑定了不同的端口
  > - 通过IP地址可以定位到某一台主机，通过端口就可以定位到主机上的某一个进程
  > - 通过指定的IP和端口，发送数据的时候对端就能接受到数据了

  端口也是一个整形数` unsigned short` ，一个16位整形数，有效端口的取值范围是：`0 ~ 65535`(0 ~ 2^16^-1)

  提问：计算机中所有的进程都需要关联一个端口吗，一个端口可以被重复使用吗?

  - 不需要，如果这个进程不需要网络通信，那么这个进程就不需要绑定端口的
  - 一个端口只能给某一个进程使用，多个进程不能同时使用同一个端口

- OSI/ISO 网络分层模型

  OSI（Open System Interconnect），即开放式系统互联。 一般都叫OSI参考模型，是ISO（国际标准化组织组织）在1985年研究的网络互联模型。

[![img](https://subingwen.cn/linux/socket/ip%E5%9B%9B%E5%B1%82%E5%8D%8F%E8%AE%AE%E6%A8%A1%E5%9E%8B.png)](https://subingwen.cn/linux/socket/ip四层协议模型.png)

> - 物理层：负责最后将信息编码成电流脉冲或其它信号用于网上传输
> - 数据链路层:
>   - 数据链路层通过物理网络链路供数据传输。
>   - 规定了0和1的分包形式，确定了网络数据包的形式；
> - 网络层
>   - 网络层负责在源和终点之间建立连接;
>   - 此处需要确定计算机的位置，通过IPv4，IPv6格式的IP地址来找到对应的主机
> - 传输层
>   - 传输层向高层提供可靠的端到端的网络数据流服务。
>   - 每一个应用程序都会在网卡注册一个端口号，该层就是端口与端口的通信
> - 会话层
>   - 会话层建立、管理和终止表示层与实体之间的通信会话；
>   - 建立一个连接（自动的手机信息、自动的网络寻址）;
> - 表示层:
>   - 对应用层数据编码和转化, 确保以一个系统应用层发送的信息 可以被另一个系统应用层识别;

## 1.2 网络协议

网络协议指的是计算机网络中互相通信的对等实体之间交换信息时所必须遵守的规则的集合。一般系统网络协议包括五个部分：通信环境，传输服务，词汇表，信息的编码格式，时序、规则和过程。先来通过下面几幅图了解一下常用的网络协议的格式：

- ==TCP协议 -> 传输层协议==

  [![img](https://subingwen.cn/linux/socket/tcp.png)](https://subingwen.cn/linux/socket/tcp.png)

- ==UDP协议 -> 传输层协议==

  ![img](https://subingwen.cn/linux/socket/udp.png)

- ==IP协议 -> 网络层协议==

  ![img](https://subingwen.cn/linux/socket/ip.png)

- ==以太网帧协议 -> 网络接口层协议==

  ![img](https://subingwen.cn/linux/socket/mac.png)

- ==数据的封装==

  ![img](https://subingwen.cn/linux/socket/1558001080021.png)

  在网络通信的时候, 程序猿需要负责的应用层数据的处理(最上层)

  - 应用层的数据可以使用某些协议进行封装, 也可以不封装
  - 程序猿需要调用发送数据的接口函数，将数据发送出去
  - 程序猿调用的API做底层数据处理
    - 传输层使用传输层协议打包数据
    - 网络层使用网络层协议打包数据
    - 网络接口层使用网络接口层协议打包数据
    - 数据被发送到internet
  - 接收端接收到发送端的数据
    - 程序猿调用接收数据的函数接收数据
    - 调用的API做相关的底层处理:
      - 网络接口层拆包 ==> 网络层的包
      - 网络层拆包 ==> 网络层的包
      - 传输层拆包 ==> 传输层数据
    - 如果应用层也使用了协议对数据进行了封装，数据的包的解析需要程序猿做

## 1.3 socket编程

Socket套接字由远景研究规划局（Advanced Research Projects Agency, ARPA）资助加里福尼亚大学伯克利分校的一个研究组研发。其目的是将TCP/IP协议相关软件移植到UNIX类系统中。设计者开发了一个接口，以便应用程序能简单地调用该接口通信。这个接口不断完善，最终形成了Socket套接字。Linux系统采用了Socket套接字，因此，Socket接口就被广泛使用，到现在已经成为事实上的标准。与套接字相关的函数被包含在头文件sys/socket.h中。

[![img](https://subingwen.cn/linux/socket/%E6%8F%92%E5%BA%A7.png)](https://subingwen.cn/linux/socket/插座.png)

通过上面的描述可以得知，套接字对应程序猿来说就是一套网络通信的接口，使用这套接口就可以完成网络通信。网络通信的主体主要分为两部分：`客户端`和`服务器端`。在客户端和服务器通信的时候需要频繁提到三个概念：`IP`、`端口`、`通信数据`，下面介绍一下需要注意的一些细节问题。

### 1.3.1 字节序

在各种计算机体系结构中，对于字节、字等的存储机制有所不同，因而引发了计算机通信领域中一个很重要的问题，即通信双方交流的信息单元（比特、字节、字、双字等等）应该以什么样的顺序进行传送。如果不达成一致的规则，通信双方将无法进行正确的编/译码从而导致通信失败。

**字节序，顾名思义字节的顺序，就是大于一个字节类型的数据在内存中的存放顺序，也就是说对于单字符来说是没有字节序问题的，字符串是单字符的集合，因此字符串也没有字节序问题。**

目前在各种体系的计算机中通常采用的字节存储机制主要有两种：Big-Endian 和 Little-Endian，下面先从字节序说起。

[![img](https://subingwen.cn/linux/socket/bits-order.jpg)](https://subingwen.cn/linux/socket/bits-order.jpg)

> 大小端的这个名词最早出现在《格列佛游记》中，里边记载了两个征战的强国，你不会想到的是，他们打仗竟然和剥鸡蛋的顺序有关。很多人认为，剥鸡蛋时应该打破鸡蛋较大的一端，这群人被称作“大端（Big endian）派”。可是那时皇帝儿子小时候吃鸡蛋的时候碰巧将一个手指弄破了。所以，当时的皇帝就下令剥鸡蛋必须打破鸡蛋较小的一端，违令者重罚，由此产生了“小端（Little endian）派”。
>
> 老百姓们对这项命令极其反感，由此引发了6次叛乱，其中一个皇帝送了命，另一个丢了王位。据估计，先后几次有11000人情愿受死也不肯去打破鸡蛋较小的一端！

- Little-Endian -> 主机字节序 (小端)——==低低高高==
  - 数据的`低位字节`存储到内存的`低地址位`, 数据的`高位字节`存储到内存的`高地址位`
  - 我们使用的PC机，数据的存储默认使用的是小端
- Big-Endian -> 网络字节序 (大端)——==低高高低==
  - 据的`低位字节`存储到内存的`高地址位`, 数据的`高位字节`存储到内存的`低地址位`
  - `套接字通信过程中操作的数据都是大端存储的，包括：接收/发送的数据、IP地址、端口。`
- 字节序举例

```c
// 有一个16进制的数, 有32位 (int): 0xab5c01ff
// 字节序, 最小的单位: char 字节, int 有4个字节, 需要将其拆分为4份
// 一个字节 unsigned char, 最大值是 255(十进制) ==> ff(16进制) 
                 内存低地址位                内存的高地址位
--------------------------------------------------------------------------->
小端:         0xff        0x01        0x5c        0xab
大端:         0xab        0x5c        0x01        0xff
    
0x12345678 是一个 16进制 数字，表示一个 32位 的整数。
    大端：12 34 56 78 
    小端：78 56 34 12
    
0x12345678 是一个 32位 的整数，因为每个十六进制数字表示 4 个二进制位（即 1 个字节是 8 位，2 个字节是 16 位，3 个字节是 24 位，4 个字节是 32 位）
    - 0x12345678 总共有 8 个十六进制数字，表示 32 位。
	- 32 位等于 4 个字节。
为什么是4个字节？
在计算机中，数字是通过二进制表示的，每 8 个二进制位（即 1 字节）构成一个存储单位。当你看到一个 16 进制的数 0x12345678 时，它代表着 32 位的整数。具体地，每个 16 进制的数字对应 4 个二进制位：
	0x1 = 0001
	0x2 = 0010
	0x3 = 0011
	0x4 = 0100
	0x5 = 0101
	0x6 = 0110
	0x7 = 0111
	0x8 = 1000
因此，总共是 8 个十六进制字符，相当于 32 位，也就是 4 个字节。
```

[![img](https://subingwen.cn/linux/socket/little.png)](https://subingwen.cn/linux/socket/little.png)

[![img](https://subingwen.cn/linux/socket/big.png)](https://subingwen.cn/linux/socket/big.png)

- 函数

> BSD Socket提供了封装好的转换接口，方便程序员使用。包括从主机字节序到网络字节序的转换函数：htons、htonl；从网络字节序到主机字节序的转换函数：ntohs、ntohl。

```c
#include <arpa/inet.h>
// u:unsigned
// 16: 16位, 32:32位
// h: host, 主机字节序
// n: net, 网络字节序
// s: short
// l: int

// 这套api主要用于 网络通信过程中 IP 和 端口 的 转换
// 将一个短整形从主机字节序（小端） -> 网络字节序（大端）
uint16_t htons(uint16_t hostshort);	
// 将一个整形从主机字节序（小端） -> 网络字节序（大端）
uint32_t htonl(uint32_t hostlong);	

// 将一个短整形从网络字节序（大端） -> 主机字节序（小端）
uint16_t ntohs(uint16_t netshort)
// 将一个整形从网络字节序（大端） -> 主机字节序（小端）
uint32_t ntohl(uint32_t netlong);
```

### 1.3.2 IP地址转换

虽然IP地址本质是一个整形数，但是在使用的过程中都是通过一个字符串来描述，下面的函数描述了如何将一个字符串类型的IP地址进行大小端转换：

```c
// 主机字节序的IP地址转换为网络字节序
// 主机字节序的IP地址是字符串, 网络字节序IP地址是整形
int inet_pton(int af, const char *src, void *dst); 
```

- 参数:
  - af: 地址族(IP地址的家族包括ipv4和ipv6)协议
    - AF_INET: ipv4格式的ip地址
    - AF_INET6: ipv6格式的ip地址
  - src: 传入参数, 对应要转换的点分十进制的ip地址: 192.168.1.100
  - dst: 传出参数, 函数调用完成, 转换得到的大端整形IP被写入到这块内存中
- 返回值：成功返回1，失败返回0或者-1

```c
#include <arpa/inet.h>// 将大端的整形数, 转换为小端的点分十进制的IP地址        const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```

- 参数:
  - af: 地址族协议
    - AF_INET: ipv4格式的ip地址
    - AF_INET6: ipv6格式的ip地址
  - src: 传入参数, 这个指针指向的内存中存储了大端的整形IP地址
  - dst: 传出参数, 存储转换得到的小端的点分十进制的IP地址
  - size: 修饰dst参数的, 标记dst指向的内存中最多可以存储多少个字节
- 返回值:
  - 成功: 指针指向第三个参数对应的内存地址, 通过返回值也可以直接取出转换得到的IP字符串
  - 失败: NULL

还有一组函数也能进程IP地址大小端的转换，但是只能处理ipv4的ip地址：

```c
// 点分十进制IP -> 大端整形
in_addr_t inet_addr (const char *cp);

// 大端整形 -> 点分十进制IP
char* inet_ntoa(struct in_addr in);
```

### 1.3.3 sockaddr 数据结构

[![img](https://subingwen.cn/linux/socket/sockaddr.png)](https://subingwen.cn/linux/socket/sockaddr.png)

```c
// 在写数据的时候不好用
struct sockaddr {
	sa_family_t sa_family;       // 地址族协议, ipv4
	char        sa_data[14];     // 端口(2字节) + IP地址(4字节) + 填充(8字节)
}

typedef unsigned short  uint16_t;
typedef unsigned int    uint32_t;
typedef uint16_t in_port_t;
typedef uint32_t in_addr_t;
typedef unsigned short int sa_family_t;
#define __SOCKADDR_COMMON_SIZE (sizeof (unsigned short int))

struct in_addr
{
    in_addr_t s_addr;
};  

// sizeof(struct sockaddr) == sizeof(struct sockaddr_in)
struct sockaddr_in
{
    sa_family_t sin_family;		/* 地址族协议: AF_INET */
    in_port_t sin_port;         /* 端口, 2字节-> 大端  */
    struct in_addr sin_addr;    /* IP地址, 4字节 -> 大端  */
    /* 填充 8字节 */
    unsigned char sin_zero[sizeof (struct sockaddr) - sizeof(sin_family) -
               sizeof (in_port_t) - sizeof (struct in_addr)];
};
```

### 1.3.4 套接字函数

使用套接字通信函数需要包含头文件`<arpa/inet.h>`，包含了这个头文件`<sys/socket.h>`就不用在包含了。

#### 1.3.4.1 socket

```c
// 创建一个套接字
int socket(int domain, int type, int protocol);
```

- 参数:
  - domain: 使用的地址族协议
    - AF_INET: 使用IPv4格式的ip地址
    - AF_INET6: 使用IPv6格式的ip地址
  - type:
    - SOCK_STREAM: 使用流式的传输协议
    - SOCK_DGRAM: 使用报式(报文)的传输协议
  - protocol: 一般写0即可, 使用默认的协议
    - SOCK_STREAM: 流式传输默认使用的是tcp
    - SOCK_DGRAM: 报式传输默认使用的udp
- 返回值:
  - 成功: 可用于套接字通信的文件描述符
  - 失败: -1

函数的返回值是一个文件描述符，通过这个文件描述符可以操作内核中的某一块内存，网络通信是基于这个文件描述符来完成的。

#### 1.3.4.2 bind

```c
// 将文件描述符和本地的IP与端口进行绑定   
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

- 参数:
  - sockfd: 监听的文件描述符, 通过socket()调用得到的返回值
  - addr: 传入参数, 要绑定的IP和端口信息需要初始化到这个结构体中，`IP和端口要转换为网络字节序（即大端）`
  - addrlen: 参数addr指向的内存大小, sizeof(struct sockaddr)
- 返回值：成功返回0，失败返回-1

#### 1.3.4.3 listen

```c
// 给监听的套接字设置监听
int listen(int sockfd, int backlog);
```

- 参数:
  - sockfd: 文件描述符, 可以通过调用socket()得到，**在监听之前必须要绑定 bind()**
  - backlog: 同时能处理的最大连接要求，**最大值为128**
- 返回值：函数调用成功返回0，调用失败返回 -1

#### 1.3.4.4 accept

```c
// 等待并接受客户端的连接请求, 建立新的连接, 会得到一个新的文件描述符(通信的)		
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

- 参数:
  - sockfd: 监听的文件描述符
  - addr: 传出参数, 里边存储了建立连接的客户端的地址信息
  - addrlen: 传入传出参数，用于存储addr指向的内存大小
- 返回值：函数调用成功，得到一个文件描述符, 用于和建立连接的这个客户端通信，调用失败返回 -1

**这个函数是一个阻塞函数，当没有新的客户端连接请求的时候，该函数阻塞；当检测到有新的客户端连接请求时，阻塞解除，新连接就建立了，得到的返回值也是一个文件描述符，基于这个文件描述符就可以和客户端通信了**。

#### 1.3.4.5 read/recv

```c
// 接收数据
ssize_t read(int sockfd, void *buf, size_t size);
ssize_t recv(int sockfd, void *buf, size_t size, int flags);
```

- 参数:
  - sockfd: 用于通信的文件描述符, accept() 函数的返回值
  - buf: 指向一块有效内存, 用于存储接收是数据
  - size: 参数buf指向的内存的容量
  - flags: 特殊的属性, 一般不使用, 指定为 0
- 返回值:
  - 大于0：实际接收的字节数
  - 等于0：对方断开了连接
  - -1：接收数据失败了

**如果连接没有断开，接收端接收不到数据，接收数据的函数会阻塞等待数据到达，数据到达后函数解除阻塞，开始接收数据，当发送端断开连接，接收端无法接收到任何数据，但是这时候就不会阻塞了，函数直接返回0**。

#### 1.3.4.6 write/send

```c
// 发送数据的函数
ssize_t write(int fd, const void *buf, size_t len);
ssize_t send(int fd, const void *buf, size_t len, int flags);
```

- 参数:
  - fd: 通信的文件描述符, accept() 函数的返回值
  - buf: 传入参数, 要发送的字符串
  - len: 要发送的字符串的长度
  - flags: 特殊的属性, 一般不使用, 指定为 0
- 返回值：
  - 大于0：实际发送的字节数，和参数len是相等的
  - -1：发送数据失败了

```c
// 成功连接服务器之后, 客户端会自动随机绑定一个端口// 服务器端调用accept()的函数, 第二个参数存储的就是客户端的IP和端口信息
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

- 参数:
  - sockfd: 通信的文件描述符, 通过调用socket()函数就得到了
  - addr: 存储了要连接的服务器端的地址信息: iP 和 端口，这个IP和端口也需要转换为大端然后再赋值
  - addrlen: addr指针指向的内存的大小 sizeof(struct sockaddr)
- 返回值：连接成功返回0，连接失败返回-1

## 1.4 TCP通信流程

TCP是一个面向连接的，安全的，流式传输协议，这个协议是一个传输层协议。

- 面向连接：是一个双向连接，通过三次握手完成，断开连接需要通过四次挥手完成。
- 安全：tcp通信过程中，会对发送的每一数据包都会进行校验, 如果发现数据丢失, 会自动重传
- 流式传输：发送端和接收端处理数据的速度，数据的量都可以不一致

[![img](https://subingwen.cn/linux/socket/tcp.jpg)](https://subingwen.cn/linux/socket/tcp.jpg)

### 1.4.1 服务器端通信流程

1. 创建用于监听的套接字, 这个套接字是一个文件描述符

   ```c
   int lfd = socket();
   ```

2. 将得到的监听的文件描述符和本地的IP 端口进行绑定

   ```c
   bind();
   ```

3. 设置监听(成功之后开始监听, 监听的是客户端的连接

   ```c
   listen();
   ```

4. 等待并接受客户端的连接请求, 建立新的连接, 会得到一个新的文件描述符(通信的)，`没有新连接请求就阻塞`

   ```c
   int cfd = accept();
   ```

5. 通信，读写操作默认都是阻塞的

   ```c
   // 接收数据
   read(); / recv();
   // 发送数据
   write(); / send();
   ```

6. 断开连接, 关闭套接字

   ```c
   close();
   ```

在tcp的服务器端, 有两类文件描述符

- 监听的文件描述符
  - 只需要有一个
  - 不负责和客户端通信, 负责检测客户端的连接请求, 检测到之后调用accept就可以建立新的连接
- 通信的文件描述符
  - 负责和建立连接的客户端通信
  - 如果有N个客户端和服务器建立了新的连接, 通信的文件描述符就有N个，每个客户端和服务器都对应一个通信的文件描述符

[![img](https://subingwen.cn/linux/socket/1558084711685.png)](https://subingwen.cn/linux/socket/1558084711685.png)

- 文件描述符对应的内存结构：
  - `一个文件文件描述符对应两块内存, 一块内存是读缓冲区, 一块内存是写缓冲区`
  - 读数据: `通过文件描述符将内存中的数据读出, 这块内存称之为读缓冲区`
  - 写数据: `通过文件描述符将数据写入到某块内存中, 这块内存称之为写缓冲区`
- 监听的文件描述符:
  - 客户端的连接请求会发送到服务器端监听的文件描述符的读缓冲区中
  - 读缓冲区中有数据, 说明有新的客户端连接
  - 调用accept()函数, 这个函数会检测监听文件描述符的读缓冲区
    - 检测不到数据, 该函数阻塞
    - 如果检测到数据, 解除阻塞, 新的连接建立
- 通信的文件描述符:
  - 客户端和服务器端都有通信的文件描述符
  - 发送数据：调用函数 write() / send()，数据进入到内核中
    - 数据并没有被发送出去, 而是将数据写入到了通信的文件描述符对应的写缓冲区中
    - 内核检测到通信的文件描述符写缓冲区中有数据, 内核会将数据发送到网络中
  - 接收数据: 调用的函数 read() / recv(), 从内核读数据
    - 数据如何进入到内核程序猿不需要处理, 数据进入到通信的文件描述符的读缓冲区中
    - 数据进入到内核, 必须使用通信的文件描述符, 将数据从读缓冲区中读出即可

> 基于tcp的服务器端通信代码:

```c
// server.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 将socket()返回值和本地的IP端口绑定到一起
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(10000);   // 大端端口
    // INADDR_ANY代表本机的所有IP, 假设有三个网卡就有三个IP地址
    // 这个宏可以代表任意一个IP地址
    // 这个宏一般用于本地的绑定操作
    addr.sin_addr.s_addr = INADDR_ANY;  // 这个宏的值为0 == 0.0.0.0
//    inet_pton(AF_INET, "192.168.237.131", &addr.sin_addr.s_addr);
    int ret = bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("bind");
        exit(0);
    }

    // 3. 设置监听
    ret = listen(lfd, 128);
    if(ret == -1)
    {
        perror("listen");
        exit(0);
    }

    // 4. 阻塞等待并接受客户端连接
    struct sockaddr_in cliaddr;
    int clilen = sizeof(cliaddr);
    int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &clilen);
    if(cfd == -1)
    {
        perror("accept");
        exit(0);
    }
    // 打印客户端的地址信息
    char ip[24] = {0};
    printf("客户端的IP地址: %s, 端口: %d\n",
           inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr, ip, sizeof(ip)),
           ntohs(cliaddr.sin_port));

    // 5. 和客户端通信
    while(1)
    {
        // 接收数据
        char buf[1024];
        memset(buf, 0, sizeof(buf));
        int len = read(cfd, buf, sizeof(buf));
        if(len > 0)
        {
            printf("客户端say: %s\n", buf);
            write(cfd, buf, len);
        }
        else if(len  == 0)
        {
            printf("客户端断开了连接...\n");
            break;
        }
        else
        {
            perror("read");
            break;
        }
    }

    close(cfd);
    close(lfd);

    return 0;
}
```

### 1.4.2 客户端的通信流程

> 在单线程的情况下客户端通信的文件描述符有一个, 没有监听的文件描述符

1. 创建一个通信的套接字

   ```c
   int cfd = socket();
   ```

2. 连接服务器, 需要知道服务器绑定的IP和端

   ```c
   connect();
   ```

3. 通信

   ```c
   // 接收数据
   read(); / recv();
   // 发送数据
   write(); / send();
   ```

4. 断开连接, 关闭文件描述符(套接字)

   ```c
   close();
   ```

> 基于tcp通信的客户端通信代码:

```c
// client.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建通信的套接字
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 连接服务器
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(10000);   // 大端端口
    inet_pton(AF_INET, "192.168.237.131", &addr.sin_addr.s_addr);

    int ret = connect(fd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("connect");
        exit(0);
    }

    // 3. 和服务器端通信
    int number = 0;
    while(1)
    {
        // 发送数据
        char buf[1024];
        sprintf(buf, "你好, 服务器...%d\n", number++);
        write(fd, buf, strlen(buf)+1);
        
        // 接收数据
        memset(buf, 0, sizeof(buf));
        int len = read(fd, buf, sizeof(buf));
        if(len > 0)
        {
            printf("服务器say: %s\n", buf);
        }
        else if(len  == 0)
        {
            printf("服务器断开了连接...\n");
            break;
        }
        else
        {
            perror("read");
            break;
        }
        sleep(1);   // 每隔1s发送一条数据
    }

    close(fd);

    return 0;
}
```

## 1.5 扩展阅读

在Window中也提供了套接字通信的API，这些API函数与Linux平台的API函数几乎相同，以至于很多人认为套接字通信的API函数库只有一套，下面来看一下这些Windows平台的套接字函数：

### 1.5.1 初始化套接字环境

使用Windows中的套接字函数需要额外包含对应的头文件以及加载响应的动态库：

```c
// 使用包含的头文件 
include <winsock2.h>
// 使用的套接字库 
ws2_32.dll       
```

在Windows中使用套接字需要先加载套接字库（套接字环境），最后需要释放套接字资源。

```c++
// 初始化Winsock库
// 返回值: 成功返回0，失败返回SOCKET_ERROR。
WSAStartup(WORD wVersionRequested, LPWSADATA lpWSAData);
```

- 参数:
  - wVersionRequested: 使用的Windows Socket的版本, 一般使用的版本是 2.2
    - 初始化这个 `MAKEWORD(2, 2);`参数
  - lpWSAData：一个WSADATA结构指针, 这是一个传入参数
    - 创建一个 WSADATA 类型的变量, 将地址传递给该函数的第二个参数

注销Winsock相关库，函数调用成功返回0，失败返回 SOCKET_ERROR。

```c++
int WSACleanup (void);
```

使用举例：

```c++
WSAData wsa;
// 初始化套接字库
WSAStartup(MAKEWORD(2, 2), &wsa);
// .......
//注销Winsock相关库
WSACleanup();
```

### 1.5.2 套接字通信函数

> 基于Linux的套接字通信流程是最全面的一套通信流程，如果是在某个框架中进行套接字通信，通信流程只会更简单，直接使用window的套接字api进行套接字通信，和Linux平台上的通信流程完全相同。

#### 1.5.2.1 结构体

```c
///////////////////////////////////////////////////////////////////////
/////////////////////////////// Windows ///////////////////////////////
///////////////////////////////////////////////////////////////////////
typedef struct in_addr {
　　union {
　　	struct{ unsigned char s_b1,s_b2, s_b3,s_b4;} S_un_b;
　　	struct{ unsigned short s_w1, s_w2;} S_un_w;
　　	unsigned long S_addr;	// 存储IP地址
　　} S_un;
}IN_ADDR;

struct sockaddr_in {
　　short int sin_family; /* Address family */
　　unsigned short int sin_port; /* Port number */
　　struct in_addr sin_addr; /* Internet address */
　　unsigned char sin_zero[8]; /* Same size as struct sockaddr */
};

///////////////////////////////////////////////////////////////////////
//////////////////////////////// Linux ////////////////////////////////
///////////////////////////////////////////////////////////////////////
typedef unsigned short  uint16_t;
typedef unsigned int    uint32_t;
typedef uint16_t in_port_t;
typedef uint32_t in_addr_t;
typedef unsigned short int sa_family_t;

struct in_addr
{
    in_addr_t s_addr;
};  

// sizeof(struct sockaddr) == sizeof(struct sockaddr_in)
struct sockaddr_in
{
    sa_family_t sin_family;     /* 地址族协议: AF_INET */
    in_port_t sin_port;         /* 端口, 2字节-> 大端  */
    struct in_addr sin_addr;    /* IP地址, 4字节 -> 大端  */
    /* 填充 8字节 */
    unsigned char sin_zero[sizeof (struct sockaddr) - sizeof(sin_family) -
                      sizeof (in_port_t) - sizeof (struct in_addr)];
};  
```

#### **1.5.2.2 大小端转换函数**

```c
// 主机字节序 -> 网络字节序
u_short htons (u_short hostshort );
u_long htonl ( u_long hostlong);

// 网络字节序 -> 主机字节序
u_short ntohs (u_short netshort );
u_long ntohl ( u_long netlong);

// linux函数, window上没有这两个函数
inet_ntop(); 
inet_pton();

// windows 和 linux 都使用, 只能处理ipv4的ip地址
// 点分十进制IP -> 大端整形
unsigned long inet_addr (const char FAR * cp);	// windows
in_addr_t     inet_addr (const char *cp);			// linux

// 大端整形 -> 点分十进制IP
// window, linux相同
char* inet_ntoa(struct in_addr in);
```

#### **1.5.2.3 套接字函数**

> **window的api中套接字对应的类型是 SOCKET 类型, linux中是 int 类型, 本质是一样的**

```c
// 创建一个套接字
// 返回值: 成功返回套接字, 失败返回INVALID_SOCKET
SOCKET socket(int af,int type,int protocal);
参数:
    - af: 地址族协议
        - ipv4: AF_INET (windows/linux)
        - PF_INET (windows)
        - AF_INET == PF_INET
   - type: 和linux一样
       	- SOCK_STREAM
        - SOCK_DGRAM
   - protocal: 一般写0 即可
       - 在windows上的另一种写法
           - IPPROTO_TCP, 使用指定的流式协议中的tcp协议
           - IPPROTO_UDP, 使用指定的报式协议中的udp协议

 // 关键字: FAR NEAR, 这两个关键字在32/64位机上是没有意义的, 指定的内存的寻址方式
// 套接字绑定本地IP和端口
// 返回值: 成功返回0，失败返回SOCKET_ERROR
int bind(SOCKET s,const struct sockaddr FAR* name, int namelen);

// 设置监听
// 返回值: 成功返回0，失败返回SOCKET_ERROR
int listen(SOCKET s,int backlog);

// 等待并接受客户端连接
// 返回值: 成功返回用于的套接字，失败返回INVALID_SOCKET。
SOCKET accept ( SOCKET s, struct sockaddr FAR* addr, int FAR* addrlen );

// 连接服务器
// 返回值: 成功返回0，失败返回SOCKET_ERROR
int connect (SOCKET s,const struct sockaddr FAR* name,int namelen );

// 在Qt中connect用户信号槽的连接, 如果要使用windows api 中的 connect 需要在函数名前加::
::connect(sock, (struct sockaddr*)&addr, sizeof(addr));

// 接收数据
// 返回值: 成功时返回接收的字节数，收到EOF时为0，失败时返回SOCKET_ERROR。
//		==0 代表对方已经断开了连接
int recv (SOCKET s,char FAR* buf,int len,int flags);

// 发送数据
// 返回值: 成功返回传输字节数，失败返回SOCKET_ERROR。
int send (SOCKET s,const char FAR * buf, int len,int flags);

// 关闭套接字
// 返回值: 成功返回0，失败返回SOCKET_ERROR
int closesocket (SOCKET s);		// 在linux中使用的函数是: int close(int fd);

//----------------------- udp 通信函数 -------------------------
// 接收数据
int recvfrom(SOCKET s,char FAR *buf,int len,int flags,
         struct sockaddr FAR *from,int FAR *fromlen);
// 发送数据
int sendto(SOCKET s,const char FAR *buf,int len,int flags,
       const struct sockaddr FAR *to,int tolen);
```

# 二、三次握手、四次挥手

TCP协议是一个安全的、面向连接的、流式传输协议，所谓的面向连接就是三次握手，对于程序猿来说只需要在客户端调用`connect()`函数，三次握手就自动进行了。先通过下图看一下TCP协议的格式，然后再介绍三次握手的具体流程。

## 2.1 Tcp协议介绍

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/tcp.png)](https://subingwen.cn/linux/three-four/tcp.png)

在Tcp协议中，比较重要的字段有：

- 源端口：表示发送端端口号，字段长 16 位，2个字节
- 目的端口：表示接收端端口号，字段长 16 位，2个字节
- 序号（sequence number）：字段长 32 位，占4个字节，序号的范围为 [0，4284967296]。
  - 由于TCP是面向字节流的，在一个TCP连接中传送的字节流中的每一个字节都按顺序编号
  - 首部中的序号字段则是指本报文段所发送的数据的第一个字节的序号，这是随机生成的。
  - 序号是循环使用的，当序号增加到最大值时，下一个序号就又回到了0
- 确认序号（acknowledgement number）：占32位（4字节），表示收到的下一个报文段的第一个数据字节的序号，如果确认序号为N，序号为S，则表明到序号N-S为止的所有数据字节都已经被正确地接收到了。
- 8个标志位（Flag）:
  - CWR：CWR 标志与后面的 ECE 标志都用于 IP 首部的 ECN 字段，ECE 标志为 1 时，则通知对方已将拥塞窗口缩小；
  - ECE：若其值为 1 则会通知对方，从对方到这边的网络有阻塞。在收到数据包的 IP 首部中 ECN 为 1 时将 TCP 首部中的 ECE 设为 1.；
  - URG：该位设为 1，表示包中有需要紧急处理的数据，对于需要紧急处理的数据，与后面的紧急指针有关；
  - ACK：该位设为 1，确认应答的字段有效，TCP规定除了最初建立连接时的 SYN 包之外该位必须设为 1；
  - PSH：该位设为 1，表示需要将收到的数据立刻传给上层应用协议，若设为 0，则先将数据进行缓存；
  - RST：该位设为 1，表示 TCP 连接出现异常必须强制断开连接；
  - SYN：用于建立连接，该位设为 1，表示希望建立连接，并在其序列号的字段进行序列号初值设定；
  - FIN：该位设为 1，表示今后不再有数据发送，希望断开连接。
- 窗口大小：该字段长 16 位，表示从确认序号所指位置开始能够接收的数据大小，TCP 不允许发送超过该窗口大小的数据。

## 2.2 三次握手

Tcp连接是双向连接，客户端和服务器需要分别向对方发送连接请求，并且建立连接，三次握手成功之后，二者之间的双向连接也就成功建立了。如果要保证三次握手顺利完成，必须要满足以下条件：

- 服务器端：已经启动，并且启动了监听（被动接受连接的一端）
- 客户端：基于服务器端监听的IP和端口，向服务器端发起连接请求（主动发起连接的一端）

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/o4YBAF-hUyCAV1vwAAbSqnJz2iM455.png)](https://subingwen.cn/linux/three-four/o4YBAF-hUyCAV1vwAAbSqnJz2iM455.png)

三次握手具体过程如下：

第一次握手：

- 客户端：客户端向服务器端发起连接请求将报文中的SYN字段置为1，生成随机序号x，seq=x
- 服务器端：接收客户端发送的请求数据，解析tcp协议，校验SYN标志位是否为1，并得到序号 x

第二次握手：

- 服务器端：给客户端回复数据

  1. 回复ACK, 将tcp协议ACK对应的标志位设置为1，表示同意了客户端建立连接的请求
  2. 回复了 ack=x+1, 这是确认序号
     - x: 客户端生成的随机序号
     - 1: 客户端给服务器发送的数据的量, SYN标志位存储到某一个字节中, 因此按照一个字节计算，表示客户端给服务器发送的1个字节服务器收到了。
  3. 将tcp协议中的SYN对应的标志位设置为 1, 服务器向客户端发起了连接请求
  4. 服务器端生成了一个随机序号 y, 发送给了客户端

- 客户端：接收回复的数据，并解析tcp协议

  1. 校验ACK标志位，为1表示服务器接收了客户端的连接请求

  2. 数据校验，确认发送给服务器的数据服务器收到了没有，计算公式如下：

     发送的数据的量 = 使用服务器回复的确认序号 - 客户端生成的随机序号 ===> 1=x+1-x

  3. 校验SYN标志位，为1表示服务器请求和客户端建立连接

  4. 得到服务器生成的随机序号: y

第三次握手：

- 客户端：发送数据给服务器

  1. 将tcp协议中ACK标志位设置为1，表示同意了服务器的连接请求
  2. 给服务器回复了一个确认序号 ack = y+1
     - y：服务器端生成的随机序号
     - 1：服务器给客户端发送的数据量，服务器给客户端发送了ACK和SYN, 都存储在这一个字节中
  3. 发送给服务器的序号就是上一次从服务器端收的确认序号因此 seq = x+1

- 服务器端：接收数据, 并解析tcp协议

  1. 查看ACK对应的标志位是否为1, 如果是1代表, 客户端同意了服务器的连接请求

  2. 数据校验，确认发送给客户端的数据客户端收到了没有，计算公式如下：

     给客户端发送的数据量 = 确认序号 - 服务器生成的随机序号 ===> 1=y+1-y

  3. 得到客户端发送的序号：x+1

     [![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/861fa058f2da35f8efa70b29bf7c45fd8689.gif)](https://subingwen.cn/linux/three-four/861fa058f2da35f8efa70b29bf7c45fd8689.gif)

## 2.3 四次挥手

四次挥手是断开连接的过程，需要双向断开，关于由哪一端先断开连接是没有要求的。通信的两端如果想要断开连接就需要调用`close()`函数，当两端都调用了该函数，四次挥手也就完成了。

- 客户端和服务器断开连接 -> 单向断开
- 服务器和客户端断开连接 -> 单向断开

进行了两次单向断开，双向断开就完成了，每进行一次单向断开，就会完成两次挥手的动作

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/o4YBAF-hUyuAOjxgAAfBsG_Zi5c370.png)](https://subingwen.cn/linux/three-four/o4YBAF-hUyuAOjxgAAfBsG_Zi5c370.png)

基于上图的例子对四次挥手的具体过程进行阐述（`实际上哪端先断开连接都是允许的`）：

第一次挥手:

- 主动断开连接的一方：发送断开连接的请求
  1. 将tcp协议中FIN标志位设置为1，表示请求断开连接
  2. 发送序号x给对端，seq=x，基于这个序号用于客户端数据校验的计算
- 被动断开连接的一方：接收请求数据, 并解析TCP协议
  1. 校验FIN标志位是否为1
  2. 收到了序号 x，基于这个数据计算回复的确认序号 ack 的值

第二次挥手:

- 被动断开连接的一方：回复数据
  1. 同意了对方断开连接的请求，将ACK标志位设置为1
  2. 回复 ack=x+1，表示成功接受了客户端发送的一个字节数据
  3. 向客户端发送序号 seq=y，基于这个序号用于服务器端数据校验的计算
- 主动断开连接的一方：接收回复数据, 并解析TCP协议
  1. 校验ACK标志位，如果为1表示断开连接的请求对方已经同意了
  2. 校验 ack确认发送的数据服务器是否收到了，发送的数据 = ack - x = x + 1 -x = 1

第三次挥手:

- 被动断开连接的一方：将tcp协议中FIN标志位设置为1，表示请求断开连接
- 主动断开连接的一方：接收请求数据, 并解析TCP协议，校验FIN标志位是否为1

第四次挥手:

- 主动断开连接的一方：回复数据
  - 将tcp协议中ACK对应的标志位设置为1，表示同意了断开连接的请求
  - ack=y+1，表示服务器发送给客户端的一个字节客户端接收到了
  - 序号 seq=h，此时的h应该等于 x+1，也就是第三次挥手时服务器回复的确认序号ack的值
- 被动断开连接的一方：收到回复的ACK, 此时双向连接双向断开, 通信的两端没有任何关系了

## 2.4 流量控制

流量控制可以让发送端根据接收端的实际接受能力控制发送的数据量。它的具体操作是，`接收端主机向发送端主机通知自己可以接收数据的大小，于是发送端会发送不会超过该大小的数据，该限制大小即为窗口大小，即窗口大小由接收端主机决定`。

TCP 首部中，专门有一个字段来通知窗口大小，接收主机将自己可以接收的缓冲区大小放在该字段中通知发送端。`当接收端的缓冲区面临数据溢出时，窗口大小的值也是随之改变，设置为一个更小的值通知发送端，从而控制数据的发送量，这样达到流量的控制。`这个控制流程的窗口也可以称作滑动窗口。

这个图是一个单向的数据发送:

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/sliding_window.png)](https://subingwen.cn/linux/three-four/sliding_window.png)

左侧是数据发送端：对应的是发送端的写缓冲区(内存)，通过一个环形队列进行数据管理

- 白色格子: 空闲的内存, 可以写数据
- 粉色的格子: 被写入到内存, 但是还没有被发送出去的数据
- 灰色的格子: 代表已经被发送出去的数据

右侧是数据接收端：对应的是接收端的读缓冲区，存储发送端发送过来的数据

- 白色格子：空闲的内存, 可以继续接收数据, 滑动窗口的值记录的就是白色的格子的大小
  - 随着接收的数据越来越多, 白色格子越来越少, 滑动窗口的值越来越小
  - 如果白色格子没有了, 滑动窗口变为0, 这时候, 发送端就被阻塞了
- 粉色格子：接收的数据，但是这个数据还没有从内核中读走，使用read() / recv()
  - 粉色格子变少了, 可用空间就变多了, 滑动窗口的值就变大了
  - 如果滑动窗口的值从0变为大于0, 接收端又重新有容量接收数据了, 发送端的阻塞自动解除，继续发送数据

基于TCP通信的流程图，记录了从三次握手 -> 数据通信 -> 四次挥手是全过程：

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3.jpg)](https://subingwen.cn/linux/three-four/滑动窗口.jpg)

```shell
# fast sender: 客户端
# slow recerver: 服务器
# win: 滑动窗口大小
# mss: maximum segment size, 单条数据的最大长度
```

第1步：第一次握手，发送连接请求SYN到服务器端

- 0(0)：0表示客户端生成的随机序号，(0)表示客户端没有额外给服务器发送数据, 因此数据的量为0
- win4096: 客户端告诉服务器, 能接收的数据(缓存)的最大量为4k
- mss1460: 客户端可以处理的单条最大字节数是1460字节

第2步：第二次握手

- ACK: 服务器同意了客户端的连接请求
  - SYN: 服务器请求和客户端建立连接
- 8000(0)：8000是服务器端生成的随机序号，(0)表示服务器没有额外给客户端发送数据, 因此数据的量为0
- 1: 发送给客户端的确认序号
  - 确认序号 = 客户端生成的随机序号 + 客户端给服务器发送的数据量(字节数) ===> 1=0+1
  - 表示客户端给服务器发送的1个字节服务器收到了
- win6144: 服务器告诉客户端我能最多缓存 6k数据
- mss1024: 服务器能处理的单条数据最大长度是 1k

第3步: 第三次握手

- ACK: 客户端同意了服务器的连接请求
- 8001: 发送给服务器的确认序号
  - 确认序号 = 服务器生成的随机序号 + 服务器给客户端发送的数据量 ===> 8001 = 8000 + 1
  - 客户端告诉服务器, 你给我发送的1个字节的数据我收到了
- win4096: 告诉服务器客户端能缓存的最大数据量是4k

第4~9步: 客户端给服务器发送数据

- 1(1024)：1 （1-0）表示之前一共给服务器发送了1个字节，(1024)表示这次要发送的数据量为 1k
- 1025(1024)：1025（1025-0）表示之前一共给服务器发送了1025个字节，(1024)表示这次要发送的数据量为 1k
- 2049(1024)：2049（2049-0）表示之前一共给服务器发送了2049个字节，(1024)表示这次要发送的数据量为 1k
- 第9步完成之后，服务器的滑动窗口变为0，接收数据的缓存被写满了，发送端阻塞

第10步:

- ack6145: 服务器给客户端回复数据，6145是确认序号, 代表实际接收的字节数

  服务器实际接收的字节数 = 确认序号 - 客户端生成的随机序号 ===> 6145 = 6145 - 0

- win2048：服务器告诉客户端我的缓存还有2k，也就是还有4k还在缓存中没有被读走

第11步：win4096表示滑动窗口变为4k，代表还可以接收4k数据，还有2k在缓存中

第12步：客户端又给服务器发送了1k数据

第13步: 第一次挥手，FIN表示客户端主动和服务器断开连接，并且发送了1k数据到服务器端

第14步: 第二次挥手，回复ACK, 同意断开连接

第15, 16步: 服务器端从读缓冲区中读数据, 第16步数据读完, 滑动窗口变成最大的6k

第17步:

- FIN: 服务器请求和客户端断开连接
- 8001(0): 服务器一共给客户端发送的字节数 8001 - 8000 = 1个字节，携带的数据量为0（FIN不计算在内）
- ack8194: 服务器收到了客户端的多少个字节: 8194 - 0 = 8194个字节

第18步: 第四次挥手

- ACK: 客户端同意了服务器断开连接的请求
- 8002: 确认序号, 可以计算出服务器给客户端发送了多少数据，8002 - 8000 = 2 个字节

# 三、Tcp状态转换

## 3.1 TCP状态转换

在TCP进行三次握手，或者四次挥手的过程中，通信的服务器和客户端内部会发送状态上的变化，发生的状态变化在程序中是看不到的，这个状态的变化也不需要程序猿去维护，但是在某些情况下进行程序的调试会去查看相关的状态信息，先来看三次握手过程中的状态转换。

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/20141015155713390.png)](https://subingwen.cn/linux/tcp-status/20141015155713390.png)

### 3.1.1 三次握手

```c
在第一次握手之前，服务器端必须先启动，并且已经开始了监听
  - 服务器端先调用了 listen() 函数, 开始监听
  - 服务器启动监听前后的状态变化: 没有状态 ---> LISTEN
```

当服务器监听启动之后，由客户端发起的三次握手过程中状态转换如下：

第一次握手:

- 客户端：调用了`connect()` 函数，状态变化：`没有状态 -> SYN_SENT`
- 服务器：收到连接请求SYN，状态变化：`LISTEN -> SYN_RCVD`

第二次握手:

- 服务器：给客户端回复ACK，并且请求和客户端建立连接，状态无变化，依然是 SYN_RCVD
- 客户端：接收数据，收到了ACK，状态变化：`SYN_SENT -> ESTABLISHED`

第三次握手:

- 客户端：给服务器回复ACK，同意建立连接，状态没有变化，还是 ESTABLISHED
- 服务器：收到了ACK，状态变化：`SYN_RCVD -> ESTABLISHED`

三次握手完成之后，客户端和服务器都变成了同一种状态，这种状态叫：ESTABLISHED，表示双向连接已经建立， 可以通信了。在通过过程中，正常的通信状态就是 ESTABLISHED。

### 3.1.2 四次挥手

关于四次挥手对于客户端和服务器哪段先断开连接没有要求，根据实际情况处理即可。下面根据上图中的实例描述一下四次挥手过程中TCP的状态转换（上图中主动断开连接的一方是客户端）：

第一次挥手:

- 客户端：调用`close()` 函数，将tcp协议中的FIN设置为1，请求和服务器断开连接，

  状态变化:`ESTABLISHED -> FIN_WAIT_1`

- 服务器：收到断开连接请求，状态变化: `ESTABLISHED -> CLOSE_WAIT`

第二次挥手:

- 服务器：回复ACK，同意断开连接的请求，状态没有变化，还是 CLOSE_WAIT
- 客户端：收到ACK，状态变化：`FIN_WAIT_1 -> FIN_WAIT_2`

第三次挥手:

- 服务器端：调用close() 函数，发送FIN给客户端，请求断开连接，状态变化：`CLOSE_WAIT -> LAST_ACK`
- 客户端：收到FIN，状态变化：`FIN_WAIT_2 -> TIME_WAIT`

第四次挥手:

- 客户端：回复ACK给服务器，状态是没有变化的，状态变化：`TIME_WAIT -> 没有状态`
- 服务器端：收到ACK，双向连接断开，状态变化：`LAST_ACK -> 无状态(没有了)`

### 3.1.3 状态转换

在下图中同样是描述TCP通信过程中的客户端和服务器端的状态转，看起来比较乱，其实只需要看两条主线：红色实线和绿色虚线。关于黑色的实线对应的是一些特殊情况下的状态切换，在此不做任何分析。

因为三次握手是由客户端发起的，据此分析红色的实线表示的客户端的状态，绿色虚线表示的是服务器端的状态。

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/wKiom1ZLIP2CRdNUAAlNCKgqwI0818.jpg)](https://subingwen.cn/linux/tcp-status/wKiom1ZLIP2CRdNUAAlNCKgqwI0818.jpg)

- 客户端：
  - 第一次握手：发送SYN，`没有状态 -> SYN_SENT`
  - 第二次握手：收到回复的ACK，`SYN_SENT -> ESTABLISHED`
  - 主动断开连接，第一次挥手发送FIN，状态`ESTABLISHED -> FIN_WAIT_1`
  - 第二次挥手，收到ACK，状态`FIN_WAIT_1 -> FIN_WAIT_2`
  - 第三次挥手，收到FIN，状态`FIN_WAIT_2 -> TIME_WAIT`
  - 第四次挥手，回复ACK，等待2倍报文时长之后，状态`TIME_WAIT -> 没有状态`
- 服务器端：
  - 启动监听，`没有状态 -> LISTEN`
  - 第一次握手，收到SYN，状态`LISTEN -> SYN_RCVD`
  - 第三次握手，收到ACK，状态`SYN_RCVD -> ESTABLISHED`
  - 收到断开连接请求，第一次挥手状态 `ESTABLISHED -> CLOSE_WAIT`
  - 第三次挥手，发送FIN请求和客户端断开连接，状态`CLOSE_WAIT -> LAST_ACK`
  - 第四次挥手，收到ACK，状态`LAST_ACK -> 无状态(没有了)`

在TCP通信的时候，当主动断开连接的一方接收到被动断开连接的一方发送的FIN和最终的ACK后（第三次挥手完成），连接的主动关闭方必须处于`TIME_WAIT`状态并持续`2MSL（Maximum Segment Lifetime）`时间，这样就能够让TCP连接的主动关闭方在它发送的ACK丢失的情况下重新发送最终的ACK。

一倍报文寿命(MSL)大概时长为30s，因此两倍报文寿命一般在1分钟作用。

**主动关闭方重新发送的最终ACK，是因为被动关闭方重传了它的FIN。事实上，被动关闭方总是重传FIN直到它收到一个最终的ACK**。

### 3.1.4 相关命令

```shell
$ netstat 参数
$ netstat -apn	| grep 关键字
```

- 参数:
  - `-a` (all)显示所有选项
  - `-p` 显示建立相关链接的程序名
  - `-n` 拒绝显示别名，能显示数字的全部转化成数字。
  - `-l` 仅列出有在 Listen (监听) 的服务状态
  - `-t` (tcp)仅显示tcp相关选项
  - `-u` (udp)仅显示udp相关选项

## 3.2 半关闭

TCP连接只有一方发送了FIN，另一方没有发出FIN包，仍然可以在一个方向上正常发送数据，这中状态可以称之为半关闭或者半连接。当四次挥手完成两次的时候，就相当于实现了半关闭，在程序中只需要在某一端直接调用 close() 函数即可。套接字通信默认是双工的，也就是双向通信，如果进行了半关闭就变成了单工，数据只能单向流动了。比如下面的这个例子：

- 服务器端:
  - 调用了close() 函数，因此不能发数据，只能接收数据
  - 关闭了服务器端的写操作，现在只能进行读操作 –> 变成了读端
- 客户端:
  - 没有调用close()，客户端和服务器的连接还保持着
  - 客户端可以给服务器发送数据，也可以接收服务器发送的数据 （但是，服务器已经丧失了发送数据的能力），因此客户端也只能发送数据，接收不到数据 –> 变成了写端

按照上述流程做了半关闭之后，从双工变成了单工，数据单向流动的方向: 客户端 —–> 服务器端。

```c
// 专门处理半关闭的函数
#include <sys/socket.h>
// 可以有选择的关闭读/写, close()函数只能关闭写操作
int shutdown(int sockfd, int how);
```

- 参数:
  - sockfd: 要操作的文件描述符
  - how:
    - SHUT_RD: 关闭文件描述符对应的读操作
    - SHUT_WR: 关闭文件描述符对应的写操作
    - SHUT_RDWR: 关闭文件描述符对应的读写操作
- 返回值：函数调用成功返回0，失败返回-1

## 3.3 端口复用

在网络通信中，一个端口只能被一个进程使用，不能多个进程共用同一个端口。我们在进行套接字通信的时候，如果按顺序执行如下操作：先启动服务器程序，再启动客户端程序，然后关闭服务器进程，再退出客户端进程，最后再启动服务器进程，就会出如下的错误提示信息：`bind error: Address already in use`

```shell
# 第二次启动服务器进程
$ ./server 
bind error: Address already in use

$ netstat -apn|grep 9999
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 127.0.0.1:9999          127.0.0.1:50178         TIME_WAIT   -   
```

通过`netstat`查看TCP状态，发现上一个服务器进程其实还没有真正退出。因为服务器进程是主动断开连接的进程, 最后状态变成了` TIME_WAIT`状态，这个进程会等待`2msl(大约1分钟)`才会退出，如果该进程不退出，其绑定的端口就不会释放，再次启动新的进程还是使用这个未释放的端口，端口被重复使用，就是提示`bind error: Address already in use`这个错误信息。

如果想要解决上述问题，就必须要设置端口复用，使用的函数原型如下：

```c
// 这个函数是一个多功能函数, 可以设置套接字选项
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```

- 参数:
  - sockfd：用于监听的文件描述符
  - level：设置端口复用需要使用 SOL_SOCKET 宏
  - optname：要设置什么属性（下边的两个宏都可以设置端口复用）
    - SO_REUSEADDR
    - SO_REUSEPORT
  - optval：设置是去除端口复用属性还是设置端口复用属性，实际应该使用 int 型变量
    - 0：不设置
    - 1：设置
  - optlen：optval指针指向的内存大小 sizeof(int)

这个函数应该添加到服务器端代码中，具体应该放到什么位置呢？答：在绑定之前设置端口复用

1. 创建监听的套接字
2. 设置端口复用
3. 绑定
4. 设置监听
5. 等待并接受客户端连接
6. 通信
7. 断开连接

参考代码如下:

```c
#include <stdio.h>
#include <ctype.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/select.h>

// server
int main(int argc, const char* argv[])
{
    // 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(9999);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 本地多有的ＩＰ
    // 127.0.0.1
    // inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr.s_addr);
    
    // 设置端口复用
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 绑定端口
    int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }

    // 监听
    ret = listen(lfd, 64);
    if(ret == -1)
    {
        perror("listen error");
        exit(1);
    }

    fd_set reads, tmp;
    FD_ZERO(&reads);
    FD_SET(lfd, &reads);

    int maxfd = lfd;

    while(1)
    {
        tmp = reads;
        int ret = select(maxfd+1, &tmp, NULL, NULL, NULL);
        if(ret == -1)
        {
            perror("select");
            exit(0);
        }

        if(FD_ISSET(lfd, &tmp))
        {
            int cfd = accept(lfd, NULL, NULL);
            FD_SET(cfd, &reads);
            maxfd = cfd > maxfd ? cfd : maxfd;
        }
        for(int i=lfd+1; i<=maxfd; ++i)
        {
            if(FD_ISSET(i, &tmp))
            {
                char buf[1024];
                int len = read(i, buf, sizeof(buf));
                if(len > 0)
                {
                    printf("client say: %s\n", buf);
                    write(i, buf, len);
                }
                else if(len == 0)
                {
                    printf("客户端断开了连接\n");
                    FD_CLR(i, &reads);
                    close(i);
                }
                else
                {
                    perror("read");
                    exit(0);
                }
            }
        }
    }

    return 0;
}
```

# 四、服务器并发（多进程/多线程）

## 4.1 单线程/进程

在TCP通信过程中，服务器端启动之后可以同时和多个客户端建立连接，并进行网络通信，但是在介绍[TCP通信流程](https://subingwen.cn/linux/socket/#4-TCP通信流程)的时候，提供的服务器代码却不能完成这样的需求，先简单的看一下之前的服务器代码的处理思路，再来分析代码中的弊端：

```c
// server.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    // 2. 将socket()返回值和本地的IP端口绑定到一起
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(10000);   // 大端端口
    // INADDR_ANY代表本机的所有IP, 假设有三个网卡就有三个IP地址
    // 这个宏可以代表任意一个IP地址
    addr.sin_addr.s_addr = INADDR_ANY;  // 这个宏的值为0 == 0.0.0.0
    int ret = bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
    // 3. 设置监听
    ret = listen(lfd, 128);
    // 4. 阻塞等待并接受客户端连接
    struct sockaddr_in cliaddr;
    int clilen = sizeof(cliaddr);
    int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &clilen);
    // 5. 和客户端通信
    while(1)
    {
        // 接收数据
        char buf[1024];
        memset(buf, 0, sizeof(buf));
        int len = read(cfd, buf, sizeof(buf));
        if(len > 0)
        {
            printf("客户端say: %s\n", buf);
            write(cfd, buf, len);
        }
        else if(len  == 0)
        {
            printf("客户端断开了连接...\n");
            break;
        }
        else
        {
            perror("read");
            break;
        }
    }
    close(cfd);
    close(lfd);
    return 0;
}
```

在上面的代码中用到了三个会引起程序阻塞的函数，分别是：

- `accept()`：如果服务器端没有新客户端连接，阻塞当前进程/线程，如果检测到新连接解除阻塞，建立连接
- `read()`：如果通信的套接字对应的读缓冲区没有数据，阻塞当前进程/线程，检测到数据解除阻塞，接收数据
- `write()`：如果通信的套接字写缓冲区被写满了，阻塞当前进程/线程（这种情况比较少见）

如果需要和发起新的连接请求的客户端建立连接，那么就必须在服务器端通过一个循环调用`accept()`函数，另外已经和服务器建立连接的客户端需要和服务器通信，发送数据时的阻塞可以忽略，当接收不到数据时程序也会被阻塞，这时候就会非常矛盾，被`accept()`阻塞就无法通信，被`read()`阻塞就无法和客户端建立新连接。因此得出一个结论，基于上述处理方式，在单线程/单进程场景下，服务器是无法处理多连接的，解决方案也有很多，常用的有三种：

1. 使用多线程实现
2. 使用多进程实现
3. 使用IO多路转接（复用）实现
4. 使用IO多路转接 + 多线程实现

## 4.2 多进程并发

如果要编写多进程版的并发服务器程序，首先要考虑，创建出的多个进程都是什么角色，这样就可以在程序中对号入座了。在Tcp服务器端一共有两个角色，分别是：监听和通信，监听是一个持续的动作，如果有新连接就建立连接，如果没有新连接就阻塞。关于通信是需要和多个客户端同时进行的，因此需要多个进程，这样才能达到互不影响的效果。进程也有两大类：父进程和子进程，通过分析我们可以这样分配进程：

- 父进程：
  - 负责监听，处理客户端的连接请求，也就是在父进程中循环调用`accept()`函数
  - 创建子进程：建立一个新的连接，就创建一个新的子进程，让这个子进程和对应的客户端通信
  - 回收子进程资源：子进程退出回收其内核PCB资源，防止出现僵尸进程
- 子进程：负责通信，基于父进程建立新连接之后得到的文件描述符，和对应的客户端完成数据的接收和发送。
  - 发送数据：`send() / write()`
  - 接收数据：`recv() / read()`

在多进程版的服务器端程序中，多个进程是有血缘关系，对应有血缘关系的进程来说，还需要想明白他们有哪些资源是可以被继承的，哪些资源是独占的，以及一些其他细节：

- 子进程是父进程的拷贝，在子进程的内核区PCB中，文件描述符也是可以被拷贝的，因此在父进程可以使用的文件描述符在子进程中也有一份，并且可以使用它们做和父进程一样的事情。
- 父子进程有用各自的独立的虚拟地址空间，因此所有的资源都是独占的
- 为了节省系统资源，对于只有在父进程才能用到的资源，可以在子进程中将其释放掉，父进程亦如此。
- 由于需要在父进程中做`accept()`操作，并且要释放子进程资源，如果想要更高效一下可以使用信号的方式处理

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/1558162823677.png)](https://subingwen.cn/linux/concurrence/1558162823677.png)

多进程版并发TCP服务器示例代码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>
#include <signal.h>
#include <sys/wait.h>
#include <errno.h>

// 信号处理函数
void callback(int num)
{
    while(1)
    {
        pid_t pid = waitpid(-1, NULL, WNOHANG);
        if(pid <= 0)
        {
            printf("子进程正在运行, 或者子进程被回收完毕了\n");
            break;
        }
        printf("child die, pid = %d\n", pid);
    }
}

int childWork(int cfd);
int main()
{
    // 1. 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 将socket()返回值和本地的IP端口绑定到一起
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(10000);   // 大端端口
    // INADDR_ANY代表本机的所有IP, 假设有三个网卡就有三个IP地址
    // 这个宏可以代表任意一个IP地址
    // 这个宏一般用于本地的绑定操作
    addr.sin_addr.s_addr = INADDR_ANY;  // 这个宏的值为0 == 0.0.0.0
    //    inet_pton(AF_INET, "192.168.237.131", &addr.sin_addr.s_addr);
    int ret = bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("bind");
        exit(0);
    }

    // 3. 设置监听
    ret = listen(lfd, 128);
    if(ret == -1)
    {
        perror("listen");
        exit(0);
    }

    // 注册信号的捕捉
    struct sigaction act;
    act.sa_flags = 0;
    act.sa_handler = callback;
    sigemptyset(&act.sa_mask);
    sigaction(SIGCHLD, &act, NULL);

    // 接受多个客户端连接, 对需要循环调用 accept
    while(1)
    {
        // 4. 阻塞等待并接受客户端连接
        struct sockaddr_in cliaddr;
        int clilen = sizeof(cliaddr);
        int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &clilen);
        if(cfd == -1)
        {
            if(errno == EINTR)
            {
                // accept调用被信号中断了, 解除阻塞, 返回了-1
                // 重新调用一次accept
                continue;
            }
            perror("accept");
            exit(0);
 
        }
        // 打印客户端的地址信息
        char ip[24] = {0};
        printf("客户端的IP地址: %s, 端口: %d\n",
               inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr, ip, sizeof(ip)),
               ntohs(cliaddr.sin_port));
        // 新的连接已经建立了, 创建子进程, 让子进程和这个客户端通信
        pid_t pid = fork();
        if(pid == 0)
        {
            // 子进程 -> 和客户端通信
            // 通信的文件描述符cfd被拷贝到子进程中
            // 子进程不负责监听
            close(lfd);
            while(1)
            {
                int ret = childWork(cfd);
                if(ret <=0)
                {
                    break;
                }
            }
            // 退出子进程
            close(cfd);
            exit(0);
        }
        else if(pid > 0)
        {
            // 父进程不和客户端通信
            close(cfd);
        }
    }
    return 0;
}


// 5. 和客户端通信
int childWork(int cfd)
{

    // 接收数据
    char buf[1024];
    memset(buf, 0, sizeof(buf));
    int len = read(cfd, buf, sizeof(buf));
    if(len > 0)
    {
        printf("客户端say: %s\n", buf);
        write(cfd, buf, len);
    }
    else if(len  == 0)
    {
        printf("客户端断开了连接...\n");
    }
    else
    {
        perror("read");
    }

    return len;
}
```

在上面的示例代码中，父子进程中分别关掉了用不到的文件描述符（父进程不需要通信，子进程也不需要监听）。如果客户端主动断开连接，那么服务器端负责和客户端通信的子进程也就退出了，子进程退出之后会给父进程发送一个叫做`SIGCHLD`的信号，在父进程中通过`sigaction()`函数捕捉了该信号，通过回调函数`callback()`中的`waitpid()`对退出的子进程进行了资源回收。

另外还有一个细节要说明一下，这是父进程的处理代码：

```c
int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &clilen);
while(1)
{
        int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &clilen);
        if(cfd == -1)
        {
            if(errno == EINTR)
            {
                // accept调用被信号中断了, 解除阻塞, 返回了-1
                // 重新调用一次accept
                continue;
            }
            perror("accept");
            exit(0);
 
        }
 }
```

如果父进程调用`accept()` 函数没有检测到新的客户端连接，父进程就阻塞在这儿了，这时候有子进程退出了，发送信号给父进程，父进程就捕捉到了这个信号`SIGCHLD`， 由于信号的优先级很高，会打断代码正常的执行流程，因此父进程的阻塞被中断，转而去处理这个信号对应的函数`callback()`，处理完毕，再次回到`accept()`位置，但是这是已经无法阻塞了，函数直接返回-1，此时函数调用失败，错误描述为`accept: Interrupted system call`，对应的错误号为`EINTR`，由于代码是被信号中断导致的错误，所以可以在程序中对这个错误号进行判断，让父进程重新调用`accept()`，继续阻塞或者接受客户端的新连接。

## 4.3 多线程并发

编写多线程版的并发服务器程序和多进程思路差不多，考虑明白了对号入座即可。多线程中的线程有两大类：主线程（父线程）和子线程，他们分别要在服务器端处理监听和通信流程。根据多进程的处理思路，就可以这样设计了：

- 主线程：
  - 负责监听，处理客户端的连接请求，也就是在父进程中循环调用`accept()`函数
  - 创建子线程：建立一个新的连接，就创建一个新的子进程，让这个子进程和对应的客户端通信
  - 回收子线程资源：由于回收需要调用阻塞函数，这样就会影响`accept()`，直接做线程分离即可。
- 子线程：负责通信，基于主线程建立新连接之后得到的文件描述符，和对应的客户端完成数据的接收和发送。
  - 发送数据：`send() / write()`
  - 接收数据：`recv() / read()`

在多线程版的服务器端程序中，多个线程共用同一个地址空间，有些数据是共享的，有些数据的独占的，下面来分析一些其中的一些细节：

- 同一地址空间中的多个线程的栈空间是独占的
- 多个线程共享全局数据区，堆区，以及内核区的文件描述符等资源，因此`需要注意数据覆盖问题`，并且在多个线程访问共享资源的时候，还需要进行线程同步。

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210306181008515.png)](https://subingwen.cn/linux/concurrence/image-20210306181008515.png)

多线程版Tcp服务器示例代码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>
#include <pthread.h>

struct SockInfo
{
    int fd;                      // 通信
    pthread_t tid;               // 线程ID
    struct sockaddr_in addr;     // 地址信息
};

struct SockInfo infos[128];

void* working(void* arg)
{
    while(1)
    {
        struct SockInfo* info = (struct SockInfo*)arg;
        // 接收数据
        char buf[1024];
        int ret = read(info->fd, buf, sizeof(buf));
        if(ret == 0)
        {
            printf("客户端已经关闭连接...\n");
            info->fd = -1;
            break;
        }
        else if(ret == -1)
        {
            printf("接收数据失败...\n");
            info->fd = -1;
            break;
        }
        else
        {
            write(info->fd, buf, strlen(buf)+1);
        }
    }
    return NULL;
}

int main()
{
    // 1. 创建用于监听的套接字
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 绑定
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;          // ipv4
    addr.sin_port = htons(8989);        // 字节序应该是网络字节序
    addr.sin_addr.s_addr =  INADDR_ANY; // == 0, 获取IP的操作交给了内核
    int ret = bind(fd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("bind");
        exit(0);
    }

    // 3.设置监听
    ret = listen(fd, 100);
    if(ret == -1)
    {
        perror("listen");
        exit(0);
    }

    // 4. 等待, 接受连接请求
    int len = sizeof(struct sockaddr);

    // 数据初始化
    int max = sizeof(infos) / sizeof(infos[0]);
    for(int i=0; i<max; ++i)
    {
        bzero(&infos[i], sizeof(infos[i]));
        infos[i].fd = -1;
        infos[i].tid = -1;
    }

    // 父进程监听, 子进程通信
    while(1)
    {
        // 创建子线程
        struct SockInfo* pinfo;
        for(int i=0; i<max; ++i)
        {
            if(infos[i].fd == -1)
            {
                pinfo = &infos[i];
                break;
            }
            if(i == max-1)
            {
                sleep(1);
                i--;
            }
        }

        int connfd = accept(fd, (struct sockaddr*)&pinfo->addr, &len);
        printf("parent thread, connfd: %d\n", connfd);
        if(connfd == -1)
        {
            perror("accept");
            exit(0);
        }
        pinfo->fd = connfd;
        pthread_create(&pinfo->tid, NULL, working, pinfo);
        pthread_detach(pinfo->tid);
    }

    // 释放资源
    close(fd);  // 监听

    return 0;
}
```

在编写多线程版并发服务器代码的时候，需要注意父子线程共用同一个地址空间中的文件描述符，因此每当在主线程中建立一个新的连接，都需要将得到文件描述符值保存起来，不能在同一变量上进行覆盖，这样做丢失了之前的文件描述符值也就不知道怎么和客户端通信了。

在上面示例代码中是将成功建立连接之后得到的用于通信的文件描述符值保存到了一个全局数组中，每个子线程需要和不同的客户端通信，需要的文件描述符值也就不一样，只要保证存储每个有效文件描述符值的变量对应不同的内存地址，在使用的时候就不会发生数据覆盖的现象，造成通信数据的混乱了。

# 五、TCP数据粘包的处理

## 5.1 背锅侠TCP

在前面介绍套接字通信的时候说到了`TCP`是传输层协议，它是一个面向连接的、安全的、流式传输协议。因为数据的传输是基于流的所以发送端和接收端每次处理的数据的量，处理数据的频率可以不是对等的，可以按照自身需求来进行决策。

TCP协议是优势非常明显，但是有时也会给我们造成困扰，正所谓：成也萧何败萧何。假设我们有如下需求：

> 客户端和服务器之间要进行基于TCP的套接字通信
>
> - 通信过程中客户端会每次会不定期给服务器发送一个不定长度的有特定含义的字符串。
> - 通信的服务器端每次都需要接收到客户端这个不定长度的字符串，并对其进行解析

根据上面的描述，服务器在接收数据的时候有如下几种情况：

1. 一次接收到了客户端发送过来的一个完整的数据包
2. 一次接收到了客户端发送过来的N个数据包，由于每个包的长度不定，无法将各个数据包拆开
3. 一次接收到了一个或者N个数据包 + 下一个数据包的一部分，还是很悲剧，无法将数据包拆开
4. 一次收到了半个数据包，下一次接收数据的时候收到了剩下的一部分+下个数据包的一部分，更悲剧，头大了
5. 另外，还有一些不可抗拒的因素：比如客户端和服务器端的网速不一样，发送和接收的数据量也会不一致

对于以上描述的现象很多时候我们将其称之为`TCP的粘包问题`，但是这种叫法不太对的，本身TCP就是面向连接的流式传输协议，特性如此，我们却说是TCP这个协议出了问题，这只能说是使用者的无知。多个数据包粘连到一起无法拆分是我们的需求过于复杂造成的，是程序猿的问题而不是协议的问题，TCP协议表示这锅它不想背。

现在问题来了，服务器端如果想保证每次都能接收到客户端发送过来的这个不定长度的数据包，程序猿应该如何解决这个问题呢？下面给大家提供几种解决方案：

1. 使用标准的应用层协议（比如：http、https）来封装要传输的不定长的数据包
2. 在每条数据的尾部添加特殊字符, 如果遇到特殊字符, 代表当条数据接收完毕了
   - 有缺陷: 效率低, 需要一个字节一个字节接收, 接收一个字节判断一次, 判断是不是那个特殊字符串
3. 在发送数据块之前, 在数据块最前边添加一个固定大小的数据头, 这时候数据由两部分组成：数据头+数据块
   - `数据头：存储当前数据包的总字节数，接收端先接收数据头，然后在根据数据头接收对应大小的字节`
   - `数据块：当前数据包的内容`

## 5.2 解决方案

如果使用TCP进行套接字通信，如果发送的数据包粘连到一起导致接收端无法解析，我们通常使用添加包头的方式轻松地解决掉这个问题。`关于数据包的包头大小可以根据自己的实际需求进行设定，这里没有啥特殊需求，因此规定包头的固定大小为4个字节，用于存储当前数据块的总字节数。`

[![image-20210511191145968](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210511191145968.png)](https://subingwen.cn/linux/tcp-data-package/image-20210511191145968.png)

### 5.2.1 发送端

对于发送端来说，数据的发送分为4步：

1. 根据待发送的数据长度N动态申请一块固定大小的内存：N+4（4是包头占用的字节数）
2. 将待发送数据的总长度写入申请的内存的前四个字节中，`此处需要将其转换为网络字节序（大端）`
3. 将待发送的数据拷贝到包头后边的地址空间中，将完整的数据包发送出去（`字符串没有字节序问题`）
4. 释放申请的堆内存。

由于发送端每次都需要将这个数据包完整的发送出去，因此可以设计一个发送函数，如果当前数据包中的数据没有发送完就让它一直发送，处理代码如下：

```c
/*
函数描述: 发送指定的字节数
函数参数:
    - fd: 通信的文件描述符(套接字)
    - msg: 待发送的原始数据
    - size: 待发送的原始数据的总字节数
函数返回值: 函数调用成功返回发送的字节数, 发送失败返回-1
*/
int writen(int fd, const char* msg, int size)
{
    const char* buf = msg;
    int count = size;
    while (count > 0)
    {
        int len = send(fd, buf, count, 0);
        if (len == -1)
        {
            close(fd);
            return -1;
        }
        else if (len == 0)
        {
            continue;
        }
        buf += len;
        count -= len;
    }
    return size;
}
```

有了这个功能函数之后就可以发送带有包头的数据块了，具体处理动作如下：

```c
/*
函数描述: 发送带有数据头的数据包
函数参数:
    - cfd: 通信的文件描述符(套接字)
    - msg: 待发送的原始数据
    - len: 待发送的原始数据的总字节数
函数返回值: 函数调用成功返回发送的字节数, 发送失败返回-1
*/
int sendMsg(int cfd, char* msg, int len)
{
   if(msg == NULL || len <= 0 || cfd <=0)
   {
       return -1;
   }
   // 申请内存空间: 数据长度 + 包头4字节(存储数据长度)
   char* data = (char*)malloc(len+4);
   int bigLen = htonl(len);
   memcpy(data, &bigLen, 4);
   memcpy(data+4, msg, len);
   // 发送数据
   int ret = writen(cfd, data, len+4);
   // 释放内存
   free(data);
   return ret;
}
```

> 关于数据的发送最后再次强调：==字符串没有字节序问题，但是数据头不是字符串是整形，因此需要从主机字节序转换为网络字节序再发送==。

## 2.2 接收端

了解了套接字的发送端如何发送数据，接收端的处理步骤也就清晰了，具体过程如下：

1. 首先接收4字节数据，`并将其从网络字节序转换为主机字节序`，这样就得到了即将要接收的数据的总长度
2. 根据得到的长度申请固定大小的堆内存，用于存储待接收的数据
3. 根据得到的数据块长度接收固定数目的数据保存到申请的堆内存中
4. 处理接收的数据
5. 释放存储数据的堆内存

从数据包头解析出要接收的数据长度之后，还需要将这个数据块完整的接收到本地才能进行后续的数据处理，因此需要编写一个接收数据的功能函数，保证能够得到一个完整的数据包数据，处理函数实现如下：

```c
/*
函数描述: 接收指定的字节数
函数参数:
    - fd: 通信的文件描述符(套接字)
    - buf: 存储待接收数据的内存的起始地址
    - size: 指定要接收的字节数
函数返回值: 函数调用成功返回发送的字节数, 发送失败返回-1
*/
int readn(int fd, char* buf, int size)
{
    char* pt = buf;
    int count = size;
    while (count > 0)
    {
        int len = recv(fd, pt, count, 0);
        if (len == -1)
        {
            return -1;
        }
        else if (len == 0)
        {
            return size - count;
        }
        pt += len;
        count -= len;
    }
    return size;
}
```

这个函数搞定之后，就可以轻松地接收带包头的数据块了，接收函数实现如下：

```c
/*
函数描述: 接收带数据头的数据包
函数参数:
    - cfd: 通信的文件描述符(套接字)
    - msg: 一级指针的地址，函数内部会给这个指针分配内存，用于存储待接收的数据，这块内存需要使用者释放
函数返回值: 函数调用成功返回接收的字节数, 发送失败返回-1
*/
int recvMsg(int cfd, char** msg)
{
    // 接收数据
    // 1. 读数据头
    int len = 0;
    readn(cfd, (char*)&len, 4);
    len = ntohl(len);
    printf("数据块大小: %d\n", len);

    // 根据读出的长度分配内存，+1 -> 这个字节存储\0
    char *buf = (char*)malloc(len+1);
    int ret = readn(cfd, buf, len);
    if(ret != len)
    {
        close(cfd);
        free(buf);
        return -1;
    }
    buf[len] = '\0';
    *msg = buf;

    return ret;
}
```

这样，在进行套接字通信的时候通过调用封装的`sendMsg()`和`recvMsg()`就可以发送和接收带数据头的数据包了，而且完美地解决了粘包的问题。

# 六、套接字通信类的封装

在掌握了基于TCP的套接字通信流程之后，为了方便使用，提高编码效率，可以对通信操作进行封装，本着有浅入深的原则，先基于C语言进行面向过程的函数封装，然后再基于C++进行面向对象的类封装。

## 6.1 基于C语言的封装

基于TCP的套接字通信分为两部分：服务器端通信和客户端通信。我们只要掌握了通信流程，封装出对应的功能函数也就不在话下了，先来回顾一下通信流程：

- 服务器端
  1. 创建用于监听的套接字
  2. 将用于监听的套接字和本地的IP以及端口进行绑定
  3. 启动监听
  4. 等待并接受新的客户端连接，连接建立得到用于通信的套接字和客户端的IP、端口信息
  5. 使用得到的通信的套接字和客户端通信（接收和发送数据）
  6. 通信结束，关闭套接字（监听 + 通信）
- 客户端
  1. 创建用于通信的套接字
  2. 使用服务器端绑定的IP和端口连接服务器
  3. 使用通信的套接字和服务器通信（发送和接收数据）
  4. 通信结束，关闭套接字（通信）

### 6.1.1 函数声明

通过通信流程可以看出服务器和客户端有些操作步骤是相同的，因此封装的功能函数是可以共用的，相关的通信函数声明如下：

```c
/////////////////////////////////////////////////// 
//////////////////// 服务器 ///////////////////////
///////////////////////////////////////////////////
int bindSocket(int lfd, unsigned short port);
int setListen(int lfd);
int acceptConn(int lfd, struct sockaddr_in *addr);

/////////////////////////////////////////////////// 
//////////////////// 客户端 ///////////////////////
///////////////////////////////////////////////////
int connectToHost(int fd, const char* ip, unsigned short port);

/////////////////////////////////////////////////// 
///////////////////// 共用 ////////////////////////
///////////////////////////////////////////////////
int createSocket();
int sendMsg(int fd, const char* msg);
int recvMsg(int fd, char* msg, int size);
int closeSocket(int fd);
int readn(int fd, char* buf, int size);
int writen(int fd, const char* msg, int size);
```

关于函数`readn()`和`writen()`的作用请参考 ==五、TCP数据粘包处理==

### 6.1.2 函数定义

```c
// 创建监套接字
int createSocket()
{
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd == -1)
    {
        perror("socket");
        return -1;
    }
    printf("套接字创建成功, fd=%d\n", fd);
    return fd;
}

// 绑定本地的IP和端口
int bindSocket(int lfd, unsigned short port)
{
    struct sockaddr_in saddr;
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(port);
    saddr.sin_addr.s_addr = INADDR_ANY;  // 0 = 0.0.0.0
    int ret = bind(lfd, (struct sockaddr*)&saddr, sizeof(saddr));
    if(ret == -1)
    {
        perror("bind");
        return -1;
    }
    printf("套接字绑定成功, ip: %s, port: %d\n",
           inet_ntoa(saddr.sin_addr), port);
    return ret;
}

// 设置监听
int setListen(int lfd)
{
    int ret = listen(lfd, 128);
    if(ret == -1)
    {
        perror("listen");
        return -1;
    }
    printf("设置监听成功...\n");
    return ret;
}

// 阻塞并等待客户端的连接
int acceptConn(int lfd, struct sockaddr_in *addr)
{
    int cfd = -1;
    if(addr == NULL)
    {
        cfd = accept(lfd, NULL, NULL);
    }
    else
    {
        int addrlen = sizeof(struct sockaddr_in);
        cfd = accept(lfd, (struct sockaddr*)addr, &addrlen);
    }
    if(cfd == -1)
    {
        perror("accept");
        return -1;
    }       
    printf("成功和客户端建立连接...\n");
    return cfd; 
}

// 接收数据
int recvMsg(int cfd, char** msg)
{
    if(msg == NULL || cfd <= 0)
    {
        return -1;
    }
    // 接收数据
    // 1. 读数据头
    int len = 0;
    readn(cfd, (char*)&len, 4);
    len = ntohl(len);
    printf("数据块大小: %d\n", len);

    // 根据读出的长度分配内存
    char *buf = (char*)malloc(len+1);
    int ret = readn(cfd, buf, len);
    if(ret != len)
    {
        return -1;
    }
    buf[len] = '\0';
    *msg = buf;

    return ret;
}

// 发送数据
int sendMsg(int cfd, char* msg, int len)
{
   if(msg == NULL || len <= 0)
   {
       return -1;
   }
   // 申请内存空间: 数据长度 + 包头4字节(存储数据长度)
   char* data = (char*)malloc(len+4);
   int bigLen = htonl(len);
   memcpy(data, &bigLen, 4);
   memcpy(data+4, msg, len);
   // 发送数据
   int ret = writen(cfd, data, len+4);
   return ret;
}

// 连接服务器
int connectToHost(int fd, const char* ip, unsigned short port)
{
    // 2. 连接服务器IP port
    struct sockaddr_in saddr;
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(port);
    inet_pton(AF_INET, ip, &saddr.sin_addr.s_addr);
    int ret = connect(fd, (struct sockaddr*)&saddr, sizeof(saddr));
    if(ret == -1)
    {
        perror("connect");
        return -1;
    }
    printf("成功和服务器建立连接...\n");
    return ret;
}

// 关闭套接字
int closeSocket(int fd)
{
    int ret = close(fd);
    if(ret == -1)
    {
        perror("close");
    }
    return ret;
}

// 接收指定的字节数
// 函数调用成功返回 size
int readn(int fd, char* buf, int size)
{
    int nread = 0;
    int left = size;
    char* p = buf;

    while(left > 0)
    {
        if((nread = read(fd, p, left)) > 0)
        {
            p += nread;
            left -= nread;
        }
        else if(nread == -1)
        {
            return -1;
        }
    }
    return size;
}

// 发送指定的字节数
// 函数调用成功返回 size
int writen(int fd, const char* msg, int size)
{
    int left = size;
    int nwrite = 0;
    const char* p = msg;

    while(left > 0)
    {
        if((nwrite = write(fd, msg, left)) > 0)
        {
            p += nwrite;
            left -= nwrite;
        }
        else if(nwrite == -1)
        {
            return -1;
        }
    }
    return size;
}
```

## 6.2 基于C++的封装

编写C++程序应当遵循面向对象三要素：封装、继承、多态。简单地说就是封装之后的类可以隐藏掉某些属性使操作更简单并且类的功能要单一，如果要代码重用可以进行类之间的继承，如果要让函数的使用更加灵活可以使用多态。因此，我们需要封装两个类：客户端类和服务器端的类。

### 6.2.1 版本1

根据面向对象的思想，整个通信过程不管是监听还是通信的套接字都是可以封装到类的内部并且将其隐藏掉，这样相关操作函数的参数也就随之减少了，使用者用起来也更简便。

#### 6.2.1.1 客户端

```c++
class TcpClient
{
public:
    TcpClient();
    ~TcpClient();
    // int connectToHost(int fd, const char* ip, unsigned short port);
    int connectToHost(string ip, unsigned short port);

    // int sendMsg(int fd, const char* msg);
    int sendMsg(string msg);
    // int recvMsg(int fd, char* msg, int size);
    string recvMsg();
    
    // int createSocket();
    // int closeSocket(int fd);

private:
    // int readn(int fd, char* buf, int size);
    int readn(char* buf, int size);
    // int writen(int fd, const char* msg, int size);
    int writen(const char* msg, int size);
    
private:
    int cfd;	// 通信的套接字
};
```

通过对客户端的操作进行封装，我们可以看到有如下的变化：

1. 文件描述被隐藏了，封装到了类的内部已经无法进行外部访问
2. 功能函数的参数变少了，因为类成员函数可以直接使用类内部的成员变量。
3. 创建和销毁套接字的函数去掉了，这两个操作可以分别放到构造和析构函数内部进行处理。
4. 在C++中可以适当的将`char*` 替换为 `string` 类，这样操作字符串就更简便一些。

#### 6.2.1.2 服务器端

```c++
class TcpServer
{
public:
    TcpServer();
    ~TcpServer();

    // int bindSocket(int lfd, unsigned short port) + int setListen(int lfd)
    int setListen(unsigned short port);
    // int acceptConn(int lfd, struct sockaddr_in *addr);
    int acceptConn(struct sockaddr_in *addr);

    // int sendMsg(int fd, const char* msg);
    int sendMsg(string msg);
    // int recvMsg(int fd, char* msg, int size);
    string recvMsg();
    
    // int createSocket();
    // int closeSocket(int fd);

private:
    // int readn(int fd, char* buf, int size);
    int readn(char* buf, int size);
    // int writen(int fd, const char* msg, int size);
    int writen(const char* msg, int size);
    
private:
    int lfd;	// 监听的套接字
    int cfd;	// 通信的套接字
};
```

通过对服务器端的操作进行封装，我们可以看到这个类和客户端的类结构以及封装思路是差不多的，并且两个类的内部有些操作的重叠的：接收和发送通信数据的函数`recvMsg()`、`sendMsg()`，以及内部函数`readn()`、`writen()`。不仅如此服务器端的类设计成这样样子是有缺陷的：服务器端一般需要和多个客户端建立连接，因此通信的套接字就需要有N个，但是在上面封装的类里边只有一个。

既然如此，我们如何解决服务器和客户端的代码冗余和服务器不能跟多客户端通信的问题呢？

答：瘦身、减负。可以将服务器的通信功能去掉，只留下监听并建立新连接一个功能。将客户端类变成一个专门用于套接字通信的类即可。服务器端整个流程使用服务器类+通信类来处理；客户端整个流程通过通信的类来处理。

### 6.2.2 版本2

根据对第一个版本的分析，可以对以上代码做如下修改：

#### 6.2.2.1 通信类

套接字通信类既可以在客户端使用，也可以在服务器端使用，职责是接收和发送数据包。

##### **类声明**

```c++
class TcpSocket
{
public:
    TcpSocket();
    TcpSocket(int socket);
    ~TcpSocket();
    int connectToHost(string ip, unsigned short port);
    int sendMsg(string msg);
    string recvMsg();

private:
    int readn(char* buf, int size);
    int writen(const char* msg, int size);

private:
    int m_fd;	// 通信的套接字
};
```

##### 类定义

```c++
TcpSocket::TcpSocket()
{
    m_fd = socket(AF_INET, SOCK_STREAM, 0);
}

TcpSocket::TcpSocket(int socket)
{
    m_fd = socket;
}

TcpSocket::~TcpSocket()
{
    if (m_fd > 0)
    {
        close(m_fd);
    }
}

int TcpSocket::connectToHost(string ip, unsigned short port)
{
    // 连接服务器IP port
    struct sockaddr_in saddr;
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(port);
    inet_pton(AF_INET, ip.data(), &saddr.sin_addr.s_addr);
    int ret = connect(m_fd, (struct sockaddr*)&saddr, sizeof(saddr));
    if (ret == -1)
    {
        perror("connect");
        return -1;
    }
    cout << "成功和服务器建立连接..." << endl;
    return ret;
}

int TcpSocket::sendMsg(string msg)
{
    // 申请内存空间: 数据长度 + 包头4字节(存储数据长度)
    char* data = new char[msg.size() + 4];
    int bigLen = htonl(msg.size());
    memcpy(data, &bigLen, 4);
    memcpy(data + 4, msg.data(), msg.size());
    // 发送数据
    int ret = writen(data, msg.size() + 4);
    delete[]data;
    return ret;
}

string TcpSocket::recvMsg()
{
    // 接收数据
    // 1. 读数据头
    int len = 0;
    readn((char*)&len, 4);
    len = ntohl(len);
    cout << "数据块大小: " << len << endl;

    // 根据读出的长度分配内存
    char* buf = new char[len + 1];
    int ret = readn(buf, len);
    if (ret != len)
    {
        return string();
    }
    buf[len] = '\0';
    string retStr(buf);
    delete[]buf;

    return retStr;
}

int TcpSocket::readn(char* buf, int size)
{
    int nread = 0;
    int left = size;
    char* p = buf;

    while (left > 0)
    {
        if ((nread = read(m_fd, p, left)) > 0)
        {
            p += nread;
            left -= nread;
        }
        else if (nread == -1)
        {
            return -1;
        }
    }
    return size;
}

int TcpSocket::writen(const char* msg, int size)
{
    int left = size;
    int nwrite = 0;
    const char* p = msg;

    while (left > 0)
    {
        if ((nwrite = write(m_fd, msg, left)) > 0)
        {
            p += nwrite;
            left -= nwrite;
        }
        else if (nwrite == -1)
        {
            return -1;
        }
    }
    return size;
}
```

在第二个版本的套接字通信类中一共有两个构造函数：

```c++
TcpSocket::TcpSocket()
{
    m_fd = socket(AF_INET, SOCK_STREAM, 0);
}

TcpSocket::TcpSocket(int socket)
{
    m_fd = socket;
}
```

- `其中无参构造一般在客户端使用，通过这个套接字对象再和服务器进行连接，之后就可以通信了`
- `有参构造主要在服务器端使用，当服务器端得到了一个用于通信的套接字对象之后，就可以基于这个套接字直接通信，因此不需要再次进行连接操作。`

#### 6.2.2.2 服务器类

服务器类主要用于套接字通信的服务器端，并且没有通信能力，当服务器和客户端的新连接建立之后，需要通过`TcpSocket`类的带参构造将通信的描述符包装成一个通信对象，这样就可以使用这个对象和客户端通信了。

##### **类声明**

```c++
class TcpServer
{
public:
    TcpServer();
    ~TcpServer();
    int setListen(unsigned short port);
    TcpSocket* acceptConn(struct sockaddr_in* addr = nullptr);

private:
    int m_fd;	// 监听的套接字
};
```

##### **类定义**

```c++
TcpServer::TcpServer()
{
    m_fd = socket(AF_INET, SOCK_STREAM, 0);
}

TcpServer::~TcpServer()
{
    close(m_fd);
}

int TcpServer::setListen(unsigned short port)
{
    struct sockaddr_in saddr;
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(port);
    saddr.sin_addr.s_addr = INADDR_ANY;  // 0 = 0.0.0.0
    int ret = bind(m_fd, (struct sockaddr*)&saddr, sizeof(saddr));
    if (ret == -1)
    {
        perror("bind");
        return -1;
    }
    cout << "套接字绑定成功, ip: "
        << inet_ntoa(saddr.sin_addr)
        << ", port: " << port << endl;

    ret = listen(m_fd, 128);
    if (ret == -1)
    {
        perror("listen");
        return -1;
    }
    cout << "设置监听成功..." << endl;

    return ret;
}

TcpSocket* TcpServer::acceptConn(sockaddr_in* addr)
{
    if (addr == NULL)
    {
        return nullptr;
    }

    socklen_t addrlen = sizeof(struct sockaddr_in);
    int cfd = accept(m_fd, (struct sockaddr*)addr, &addrlen);
    if (cfd == -1)
    {
        perror("accept");
        return nullptr;
    }
    printf("成功和客户端建立连接...\n");
    return new TcpSocket(cfd);
}
```

通过调整可以发现，套接字服务器类功能更加单一了，这样设计即解决了代码冗余问题，还能使这两个类更容易维护。

## 6.3 测试代码

### 6.3.1 客户端

```c++
int main()
{
    // 1. 创建通信的套接字
    TcpSocket tcp;

    // 2. 连接服务器IP port
    int ret = tcp.connectToHost("192.168.237.131", 10000);
    if (ret == -1)
    {
        return -1;
    }

    // 3. 通信
    int fd1 = open("english.txt", O_RDONLY);
    int length = 0;
    char tmp[100];
    memset(tmp, 0, sizeof(tmp));
    while ((length = read(fd1, tmp, sizeof(tmp))) > 0)
    {
        // 发送数据
        tcp.sendMsg(string(tmp, length));

        cout << "send Msg: " << endl;
        cout << tmp << endl << endl << endl;
        memset(tmp, 0, sizeof(tmp));

        // 接收数据
        usleep(300);
    }

    sleep(10);

    return 0;
}
```

### 6.3.2 服务器端

```c++
struct SockInfo
{
    TcpServer* s;
    TcpSocket* tcp;
    struct sockaddr_in addr;
};

void* working(void* arg)
{
    struct SockInfo* pinfo = static_cast<struct SockInfo*>(arg);
    // 连接建立成功, 打印客户端的IP和端口信息
    char ip[32];
    printf("客户端的IP: %s, 端口: %d\n",
        inet_ntop(AF_INET, &pinfo->addr.sin_addr.s_addr, ip, sizeof(ip)),
        ntohs(pinfo->addr.sin_port));

    // 5. 通信
    while (1)
    {
        printf("接收数据: .....\n");
        string msg = pinfo->tcp->recvMsg();
        if (!msg.empty())
        {
            cout << msg << endl << endl << endl;
        }
        else
        {
            break;
        }
    }
    delete pinfo->tcp;
    delete pinfo;
    return nullptr;
}

int main()
{
    // 1. 创建监听的套接字
    TcpServer s;
    // 2. 绑定本地的IP port并设置监听
    s.setListen(10000);
    // 3. 阻塞并等待客户端的连接
    while (1)
    {
        SockInfo* info = new SockInfo;
        TcpSocket* tcp = s.acceptConn(&info->addr);
        if (tcp == nullptr)
        {
            cout << "重试...." << endl;
            continue;
        }
        // 创建子线程
        pthread_t tid;
        info->s = &s;
        info->tcp = tcp;

        pthread_create(&tid, NULL, working, info);
        pthread_detach(tid);
    }

    return 0;
}
```

# 附1：基于TCP的Qt网络通信

在标准C++没有提供专门用于套接字通信的类，所以只能使用操作系统提供的基于C的API函数，基于这些C的API函数我们也可以封装自己的C++类。但是Qt就不一样了，它是C++的一个框架并且里边提供了用于套接字通信的类（TCP、UDP）这样就使得我们的操作变得更加简单了（`当然，在Qt中使用标准C的API进行套接字通信也是完全没有问题的`）。下面，给大家讲一下如果使用相关类的进行TCP通信。

使用Qt提供的类进行基于TCP的套接字通信需要用到两个类：

- QTcpServer：服务器类，用于监听客户端连接以及和客户端建立连接。
- QTcpSocket：通信的套接字类，客户端、服务器端都需要使用。

这两个套接字通信类都属于网络模块`network`。

## 1.1 QTcpServer

`QTcpServer`类用于监听客户端连接以及和客户端建立连接，在使用之前先介绍一下这个类提供的一些常用API函数：

### 1.1.1 公共成员函数

> 构造函数

```c++
QTcpServer::QTcpServer(QObject *parent = Q_NULLPTR);
```

> 给监听的套接字设置监听
>

```c++
bool QTcpServer::listen(const QHostAddress &address = QHostAddress::Any, quint16 port = 0);
// 判断当前对象是否在监听, 是返回true，没有监听返回false
bool QTcpServer::isListening() const;
// 如果当前对象正在监听返回监听的服务器地址信息, 否则返回 QHostAddress::Null
QHostAddress QTcpServer::serverAddress() const;
// 如果服务器正在侦听连接，则返回服务器的端口; 否则返回0
quint16 QTcpServer::serverPort() const
```

- 参数：
  - address：通过类`QHostAddress`可以封装`IPv4`、`IPv6`格式的IP地址，`QHostAddress::Any`表示自动绑定
  - port：如果指定为0表示随机绑定一个可用端口。
- 返回值：绑定成功返回true，失败返回false

> 得到和客户端建立连接之后用于通信的`QTcpSocket`套接字对象，它是`QTcpServer`的一个子对象，当`QTcpServer`对象析构的时候会自动析构这个子对象，当然也可自己手动析构，建议用完之后自己手动析构这个通信的`QTcpSocket`对象。

```c++
QTcpSocket *QTcpServer::nextPendingConnection();
```

阻塞等待客户端发起的连接请求，`不推荐在单线程程序中使用，建议使用非阻塞方式处理新连接，即使用信号 newConnection() `。

```c++
bool QTcpServer::waitForNewConnection(int msec = 0, bool *timedOut = Q_NULLPTR);
```

- 参数：
  - msec：指定阻塞的最大时长，单位为毫秒（ms）
  - timeout：传出参数，如果操作超时timeout为true，没有超时timeout为false

### 1.1.2 信号

> 当接受新连接导致错误时，将发射如下信号。socketError参数描述了发生的错误相关的信息。

```c++
[signal] void QTcpServer::acceptError(QAbstractSocket::SocketError socketError);
```

> 每次有新连接可用时都会发出 newConnection() 信号。
>

```c++
[signal] void QTcpServer::newConnection();
```

## 1.2 QTcpSocket

`QTcpSocket`是一个套接字通信类，不管是客户端还是服务器端都需要使用。在Qt中发送和接收数据也属于IO操作（网络IO），先来看一下这个类的继承关系：

[![image-20210512174459252](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210512174459252.png)](https://subingwen.cn/qt/socket-tcp/image-20210512174459252.png)

### 1.2.1 公共成员函数

> 构造函数

```c++
QTcpSocket::QTcpSocket(QObject *parent = Q_NULLPTR);
```

> 连接服务器，`需要指定服务器端绑定的IP和端口信息。`
>

```c++
[virtual] void QAbstractSocket::connectToHost(const QString &hostName, quint16 port, OpenMode openMode = ReadWrite, NetworkLayerProtocol protocol = AnyIPProtocol);

[virtual] void QAbstractSocket::connectToHost(const QHostAddress &address, quint16 port, OpenMode openMode = ReadWrite);
```

**在Qt中不管调用读操作函数接收数据，还是调用写函数发送数据，操作的对象都是本地的由Qt框架维护的一块内存。因此，调用了发送函数数据不一定会马上被发送到网络中，调用了接收函数也不是直接从网络中接收数据，关于底层的相关操作是不需要使用者来维护的**。

> 接收数据
>

```c++
// 指定可接收的最大字节数 maxSize 的数据到指针 data 指向的内存中
qint64 QIODevice::read(char *data, qint64 maxSize);
// 指定可接收的最大字节数 maxSize，返回接收的字符串
QByteArray QIODevice::read(qint64 maxSize);
// 将当前可用操作数据全部读出，通过返回值返回读出的字符串
QByteArray QIODevice::readAll();
```

> 发送数据
>

```c++
// 发送指针 data 指向的内存中的 maxSize 个字节的数据
qint64 QIODevice::write(const char *data, qint64 maxSize);
// 发送指针 data 指向的内存中的数据，字符串以 \0 作为结束标记
qint64 QIODevice::write(const char *data);
// 发送参数指定的字符串
qint64 QIODevice::write(const QByteArray &byteArray);
```

### 1.2.2 信号

> 在使用`QTcpSocket`进行套接字通信的过程中，如果该类对象发射出`readyRead()`信号，说明对端发送的数据达到了，之后就可以调用 `read 函数`接收数据了。

```c++
[signal] void QIODevice::readyRead();
```

> 调用`connectToHost()`函数并成功建立连接之后发出`connected()`信号。
>

```c++
[signal] void QAbstractSocket::connected();
```

> 在套接字断开连接时发出`disconnected()`信号。
>

```c++
[signal] void QAbstractSocket::disconnected();
```

## 1.3 通信流程

使用Qt提供的类进行套接字通信比使用标准C API进行网络通信要简单（因为在内部进行了封装）。 Qt中的套接字通信流程如下：

### 1.3.1 服务器端

#### 1.3.1.1 通信流程

1. 创建套接字服务器`QTcpServer`对象
2. 通过`QTcpServer`对象设置监听，即：`QTcpServer::listen()`
3. 基于`QTcpServer::newConnection()`信号检测是否有新的客户端连接
4. 如果有新的客户端连接调用`QTcpSocket *QTcpServer::nextPendingConnection()`得到通信的套接字对象
5. 使用通信的套接字对象`QTcpSocket`和客户端进行通信

#### 1.3.1.2 代码片段

服务器端的窗口界面如下图所示：

[![image-20210512184859718](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210512184859718.png)](https://subingwen.cn/qt/socket-tcp/image-20210512184859718.png)

##### **头文件**

```c++
class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    explicit MainWindow(QWidget *parent = 0);
    ~MainWindow();

private slots:
    void on_startServer_clicked();

    void on_sendMsg_clicked();

private:
    Ui::MainWindow *ui;
    QTcpServer* m_server;
    QTcpSocket* m_tcp;
};
```

##### **源文件**

```c++
MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);
    setWindowTitle("TCP - 服务器");
    // 创建 QTcpServer 对象
    m_server = new QTcpServer(this);
    // 检测是否有新的客户端连接
    connect(m_server, &QTcpServer::newConnection, this, [=]()
    {
        m_tcp = m_server->nextPendingConnection();
        ui->record->append("成功和客户端建立了新的连接...");
        m_status->setPixmap(QPixmap(":/connect.png").scaled(20, 20));
        // 检测是否有客户端数据
        connect(m_tcp, &QTcpSocket::readyRead, this, [=]()
        {
            // 接收数据
            QString recvMsg = m_tcp->readAll();
            ui->record->append("客户端Say: " + recvMsg);
        });
        // 客户端断开了连接
        connect(m_tcp, &QTcpSocket::disconnected, this, [=]()
        {
            ui->record->append("客户端已经断开了连接...");
            m_tcp->deleteLater();
            m_status->setPixmap(QPixmap(":/disconnect.png").scaled(20, 20));
        });
    });
}

MainWindow::~MainWindow()
{
    delete ui;
}

// 启动服务器端的服务按钮
void MainWindow::on_startServer_clicked()
{
    unsigned short port = ui->port->text().toInt();
    // 设置服务器监听
    m_server->listen(QHostAddress::Any, port);
    ui->startServer->setEnabled(false);
}

// 点击发送数据按钮
void MainWindow::on_sendMsg_clicked()
{
    QString sendMsg = ui->msg->toPlainText();
    m_tcp->write(sendMsg.toUtf8());
    ui->record->append("服务器Say: " + sendMsg);
    ui->msg->clear();
}
```

### 1.3.2 客户端

#### 1.3.2.1 通信流程

1. 创建通信的套接字类`QTcpSocket`对象
2. 使用服务器端绑定的IP和端口连接服务器`QAbstractSocket::connectToHost()`
3. 使用`QTcpSocket`对象和服务器进行通信

#### 1.3.2.2 代码片段

客户端的窗口界面如下图所示：

[![image-20210512184937702](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210512184937702.png)](https://subingwen.cn/qt/socket-tcp/image-20210512184937702.png)

##### **头文件**

```c++
class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    explicit MainWindow(QWidget *parent = 0);
    ~MainWindow();

private slots:
    void on_connectServer_clicked();

    void on_sendMsg_clicked();

    void on_disconnect_clicked();

private:
    Ui::MainWindow *ui;
    QTcpSocket* m_tcp;
};
```

##### 源文件

```c++
MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);
    setWindowTitle("TCP - 客户端");

    // 创建通信的套接字对象
    m_tcp = new QTcpSocket(this);
    // 检测服务器是否回复了数据
    connect(m_tcp, &QTcpSocket::readyRead, [=]()
    {
        // 接收服务器发送的数据
        QByteArray recvMsg = m_tcp->readAll();
        ui->record->append("服务器Say: " + recvMsg);
    });
        
    // 检测是否和服务器是否连接成功了
    connect(m_tcp, &QTcpSocket::connected, this, [=]()
    {
        ui->record->append("恭喜, 连接服务器成功!!!");
        m_status->setPixmap(QPixmap(":/connect.png").scaled(20, 20));
    });
        
    // 检测服务器是否和客户端断开了连接
    connect(m_tcp, &QTcpSocket::disconnected, this, [=]()
    {
        ui->record->append("服务器已经断开了连接, ...");
        ui->connectServer->setEnabled(true);
        ui->disconnect->setEnabled(false);
    });
}

MainWindow::~MainWindow()
{
    delete ui;
}

// 连接服务器按钮按下之后的处理动作
void MainWindow::on_connectServer_clicked()
{
    QString ip = ui->ip->text();
    unsigned short port = ui->port->text().toInt();
    // 连接服务器
    m_tcp->connectToHost(QHostAddress(ip), port);
    ui->connectServer->setEnabled(false);
    ui->disconnect->setEnabled(true);
}

// 发送数据按钮按下之后的处理动作
void MainWindow::on_sendMsg_clicked()
{
    QString sendMsg = ui->msg->toPlainText();
    m_tcp->write(sendMsg.toUtf8());
    ui->record->append("客户端Say: " + sendMsg);
    ui->msg->clear();
}

// 断开连接按钮被按下之后的处理动作
void MainWindow::on_disconnect_clicked()
{
    m_tcp->close();
    ui->connectServer->setEnabled(true);
    ui->disconnect->setEnabled(false);
}
```

# 七、IO多路转接（复用）之select

## 7.1 IO多路转接(复用)

IO多路转接也称为IO多路复用，它是一种网络通信的手段（机制），通过`这种方式可以同时监测多个文件描述符并且这个过程是阻塞的，一旦检测到有文件描述符就绪（ 可以读数据或者可以写数据）程序的阻塞就会被解除，之后就可以基于这些（一个或多个）就绪的文件描述符进行通信了`。通过这种方式在单线程/进程的场景下也可以在服务器端实现并发。常见的IO多路转接方式有：`select`、`poll`、`epoll`。

下面先对多线程/多进程并发和IO多路转接的并发处理流程进行对比（服务器端）：

- 多线程/多进程并发
  - 主线程/父进程：调用 **accept()** 监测客户端连接请求
    - 如果没有新的客户端的连接请求，当前线程/进程会阻塞
    - 如果有新的客户端连接请求解除阻塞，建立连接
  - 子线程/子进程：和建立连接的客户端通信
    - 调用 `read() / recv() `接收客户端发送的通信数据，如果没有通信数据，当前线程/进程会阻塞，数据到达之后阻塞自动解除
    - 调用 `write() / send() `给客户端发送数据，如果写缓冲区已满，当前线程/进程会阻塞，否则将待发送数据写入写缓冲区中
- IO多路转接并发
  - 使用IO多路转接函数委托内核检测服务器端所有的文件描述符（通信和监听两类），这个检测过程会导致进程/线程的阻塞，如果检测到已就绪的文件描述符阻塞解除，并将这些已就绪的文件描述符传出
  - 根据类型对传出的所有已就绪文件描述符进行判断，并做出不同的处理
    - 监听的文件描述符：和客户端建立连接
      - 此时调用`accept()`是不会导致程序阻塞的，因为监听的文件描述符是已就绪的（有新请求）
    - 通信的文件描述符：调用通信函数和已建立连接的客户端通信
      - 调用 `read() / recv() `不会阻塞程序，因为通信的文件描述符是就绪的，读缓冲区内已有数据
      - 调用 `write() / send() `不会阻塞程序，因为通信的文件描述符是就绪的，写缓冲区不满，可以往里面写数据
  - 对这些文件描述符继续进行下一轮的检测（循环往复。。。）

**与多进程和多线程技术相比，I/O多路复用技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销**。

## 7.2 select

### 7.2.1 函数原型

使用select这种IO多路转接方式需要调用一个同名函数`select`，这个函数是跨平台的，`Linux`、`Mac`、`Windows`都是支持的。程序猿通过调用这个函数可以委托内核帮助我们检测若干个文件描述符的状态，`其实就是检测这些文件描述符对应的读写缓冲区的状态`：

- 读缓冲区：检测里边有没有数据，如果有数据该缓冲区对应的文件描述符就绪
- 写缓冲区：检测写缓冲区是否可以写(有没有容量)，如果有容量可以写，缓冲区对应的文件描述符就绪
- 读写异常：检测读写缓冲区是否有异常，如果有该缓冲区对应的文件描述符就绪

委托检测的文件描述符被遍历检测完毕之后，已就绪的这些满足条件的文件描述符会通过`select()`的参数分3个集合传出，程序猿得到这几个集合之后就可以分情况依次处理了。

下面来看一下这个函数的函数原型：

```c
#include <sys/select.h>
struct timeval {
    time_t      tv_sec;         /* seconds */
    suseconds_t tv_usec;        /* microseconds */
};

int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval * timeout);
```

- 函数参数：
  - nfds：委托内核检测的这三个集合中最大的文件描述符 + 1
    - 内核需要线性遍历这些集合中的文件描述符，这个值是循环结束的条件
    - 在Window中这个参数是无效的，指定为-1即可
  - readfds：文件描述符的集合, 内核只检测这个集合中文件描述符对应的读缓冲区
    - `传入传出参数`，读集合一般情况下都是需要检测的，这样才知道通过哪个文件描述符接收数据
  - writefds：文件描述符的集合, 内核只检测这个集合中文件描述符对应的写缓冲区
    - `传入传出参数`，如果不需要使用这个参数可以指定为NULL
  - exceptfds：文件描述符的集合, 内核检测集合中文件描述符是否有异常状态
    - `传入传出参数`，如果不需要使用这个参数可以指定为NULL
  - timeout：超时时长，用来强制解除select()函数的阻塞的
    - NULL：函数检测不到就绪的文件描述符会一直阻塞。
    - 等待固定时长（秒）：函数检测不到就绪的文件描述符，在指定时长之后强制解除阻塞，函数返回0
    - 不等待：函数不会阻塞，直接将该参数对应的结构体初始化为0即可。
- 函数返回值：
  - 大于0：成功，返回集合中已就绪的文件描述符的总个数
  - 等于-1：函数调用失败
  - 等于0：超时，没有检测到就绪的文件描述符

另外初始化`fd_set`类型的参数还需要使用相关的一些列操作函数，具体如下：

```c
// 将文件描述符fd从set集合中删除 == 将fd对应的标志位设置为0        
void FD_CLR(int fd, fd_set *set);
// 判断文件描述符fd是否在set集合中 == 读一下fd对应的标志位到底是0还是1
int  FD_ISSET(int fd, fd_set *set);
// 将文件描述符fd添加到set集合中 == 将fd对应的标志位设置为1
void FD_SET(int fd, fd_set *set);
// 将set集合中, 所有文件文件描述符对应的标志位设置为0, 集合中没有添加任何文件描述符
void FD_ZERO(fd_set *set);
```

### 7.2.2 细节描述

在`select()`函数中第2、3、4个参数都是`fd_set`类型，它表示一个文件描述符的集合，类似于信号集 `sigset_t`，这个类型的数据有128个字节，也就是1024个标志位，和内核中文件描述符表中的文件描述符个数是一样的。

```c
sizeof(fd_set) = 128 字节 * 8 = 1024 bit      // int [32]
```

这并不是巧合，而是故意为之。这块内存中的每一个bit 和 文件描述符表中的每一个文件描述符是一一对应的关系，这样就可以使用最小的存储空间将要表达的意思描述出来了。

下图中的fd_set中存储了要委托内核检测读缓冲区的文件描述符集合。

- 如果集合中的标志位为`0`代表`不检测`这个文件描述符状态
- 如果集合中的标志位为`1`代表`检测`这个文件描述符状态

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210324233252448.png)](https://subingwen.cn/linux/select/image-20210324233252448.png)

内核在遍历这个读集合的过程中，如果被检测的文件描述符对应的读缓冲区中没有数据，内核将修改这个文件描述符在读集合`fd_set`中对应的标志位，改为`0`，如果有数据那么这个标志位的值不变，还是`1`。

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210324234324388.png)](https://subingwen.cn/linux/select/image-20210324234324388.png)

当`select()`函数解除阻塞之后，被内核修改过的读集合通过参数传出，此时集合中只要标志位的值为`1`，那么它对应的文件描述符肯定是就绪的，我们就可以基于这个文件描述符和客户端建立新连接或者通信了。

## 7.3 并发处理

### 7.3.1 处理流程

如果在服务器基于select实现并发，其处理流程如下：

1. 创建监听的套接字 lfd = socket();
2. 将监听的套接字和本地的IP和端口绑定 bind()
3. 给监听的套接字设置监听 listen()
4. 创建一个文件描述符集合 fd_set，用于存储需要检测读事件的所有的文件描述符
   - 通过 FD_ZERO() 初始化
   - 通过 FD_SET() 将监听的文件描述符放入检测的读集合中
5. 循环调用select()，周期性的对所有的文件描述符进行检测
6. select() 解除阻塞返回，得到内核传出的满足条件的就绪的文件描述符集合
   - 通过FD_ISSET() 判断集合中的标志位是否为 1
     - 如果这个文件描述符是监听的文件描述符，调用 accept() 和客户端建立连接
       - 将得到的新的通信的文件描述符，通过FD_SET() 放入到检测集合中
     - 如果这个文件描述符是通信的文件描述符，调用通信函数和客户端通信
       - 如果客户端和服务器断开了连接，使用FD_CLR()将这个文件描述符从检测集合中删除
       - 如果没有断开连接，正常通信即可
7. 重复第6步

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210324235635304.png)](https://subingwen.cn/linux/select/image-20210324235635304.png)

### 7.3.2 通信代码

#### **服务器端代码如下：**

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建监听的fd
    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    // 2. 绑定
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(lfd, (struct sockaddr*)&addr, sizeof(addr));

    // 3. 设置监听
    listen(lfd, 128);

    // 将监听的fd的状态检测委托给内核检测
    int maxfd = lfd;
    // 初始化检测的读集合
    fd_set rdset;
    fd_set rdtemp;
    // 清零
    FD_ZERO(&rdset);
    // 将监听的lfd设置到检测的读集合中
    FD_SET(lfd, &rdset);
    // 通过select委托内核检测读集合中的文件描述符状态, 检测read缓冲区有没有数据
    // 如果有数据, select解除阻塞返回
    // 应该让内核持续检测
    while(1)
    {
        // 默认阻塞
        // rdset 中是委托内核检测的所有的文件描述符
        rdtemp = rdset;
        int num = select(maxfd+1, &rdtemp, NULL, NULL, NULL);
        // rdset中的数据被内核改写了, 只保留了发生变化的文件描述的标志位上的1, 没变化的改为0
        // 只要rdset中的fd对应的标志位为1 -> 缓冲区有数据了
        // 判断
        // 有没有新连接
        if(FD_ISSET(lfd, &rdtemp))
        {
            // 接受连接请求, 这个调用不阻塞
            struct sockaddr_in cliaddr;
            int cliLen = sizeof(cliaddr);
            int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &cliLen);

            // 得到了有效的文件描述符
            // 通信的文件描述符添加到读集合
            // 在下一轮select检测的时候, 就能得到缓冲区的状态
            FD_SET(cfd, &rdset);
            // 重置最大的文件描述符
            maxfd = cfd > maxfd ? cfd : maxfd;
        }

        // 没有新连接, 通信
        for(int i=0; i<maxfd+1; ++i)
        {
			// 判断从监听的文件描述符之后到maxfd这个范围内的文件描述符是否读缓冲区有数据
            if(i != lfd && FD_ISSET(i, &rdtemp))
            {
                // 接收数据
                char buf[10] = {0};
                // 一次只能接收10个字节, 客户端一次发送100个字节
                // 一次是接收不完的, 文件描述符对应的读缓冲区中还有数据
                // 下一轮select检测的时候, 内核还会标记这个文件描述符缓冲区有数据 -> 再读一次
                // 	循环会一直持续, 知道缓冲区数据被读完位置
                int len = read(i, buf, sizeof(buf));
                if(len == 0)
                {
                    printf("客户端关闭了连接...\n");
                    // 将检测的文件描述符从读集合中删除
                    FD_CLR(i, &rdset);
                    close(i);
                }
                else if(len > 0)
                {
                    // 收到了数据
                    // 发送数据
                    write(i, buf, strlen(buf)+1);
                }
                else
                {
                    // 异常
                    perror("read");
                }
            }
        }
    }

    return 0;
}
```

在上面的代码中，创建了两个`fd_set`变量，用于保存要检测的读集合：

```c
// 初始化检测的读集合
fd_set rdset;
fd_set rdtemp;
```

> `rdset`用于保存要检测的原始数据，这个变量不能作为参数传递给select函数，因为在函数内部这个变量中的值会被内核修改，函数调用完毕返回之后，里边就不是原始数据了，大部分情况下是值为1的标志位变少了，不可能每一轮检测，所有的文件描述符都是就行的状态。因此需要通过`rdtemp`变量将原始数据传递给内核，select() 调用完毕之后再将内核数据传出，这两个变量的功能是不一样的。

#### **客户端代码:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建用于通信的套接字
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 连接服务器
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;     // ipv4
    addr.sin_port = htons(9999);   // 服务器监听的端口, 字节序应该是网络字节序
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr.s_addr);
    int ret = connect(fd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("connect");
        exit(0);
    }

    // 通信
    while(1)
    {
        // 读数据
        char recvBuf[1024];
        // 写数据
        // sprintf(recvBuf, "data: %d\n", i++);
        fgets(recvBuf, sizeof(recvBuf), stdin);
        write(fd, recvBuf, strlen(recvBuf)+1);
        // 如果客户端没有发送数据, 默认阻塞
        read(fd, recvBuf, sizeof(recvBuf));
        printf("recv buf: %s\n", recvBuf);
        sleep(1);
    }

    // 释放资源
    close(fd); 

    return 0;
}
```

客户端不需要使用IO多路转接进行处理，因为客户端和服务器的对应关系是 1：N，也就是说客户端是比较专一的，只能和一个连接成功的服务器通信。

虽然使用select这种IO多路转接技术可以降低系统开销，提高程序效率，但是它也有局限性：

1. 待检测集合（第2、3、4个参数）需要频繁的在用户区和内核区之间进行数据的拷贝，效率低
2. 内核对于select传递进来的待检测集合的检测方式是线性的
   - 如果集合内待检测的文件描述符很多，检测效率会比较低
   - 如果集合内待检测的文件描述符相对较少，检测效率会比较高
3. `使用select能够检测的最大文件描述符个数有上限，默认是1024，这是在内核中被写死了的。`

# 八、IO多路转接（复用）之poll

## 8.1 poll函数

poll的机制与select类似，与select在本质上没有多大差别，使用方法也类似，下面的是对于二者的对比：

- 内核对应文件描述符的检测也是以线性的方式进行轮询，根据描述符的状态进行处理
- poll和select检测的文件描述符集合会在检测过程中频繁的进行用户区和内核区的拷贝，它的开销随着文件描述符数量的增加而线性增大，从而效率也会越来越低。
- `select检测的文件描述符个数上限是1024，poll没有最大文件描述符数量的限制`
- `select可以跨平台使用，poll只能在Linux平台使用`

poll函数的函数原型如下：

```c
#include <poll.h>
// 每个委托poll检测的fd都对应这样一个结构体
struct pollfd {
    int   fd;         /* 委托内核检测的文件描述符 */
    short events;     /* 委托内核检测文件描述符的什么事件 */
    short revents;    /* 文件描述符实际发生的事件 -> 传出 */
};

struct pollfd myfd[100];
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

- 函数参数：

  - fds: 这是一个`struct pollfd`类型的数组, 里边存储了待检测的文件描述符的信息，这个数组中有三个成员：

    - fd：委托内核检测的文件描述符
    - events：委托内核检测的fd事件（输入、输出、错误），每一个事件有多个取值
    - revents：这是一个传出参数，数据由内核写入，存储内核检测之后的结果

    ![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/1558308141721.png)

  - nfds: 这是第一个参数数组中最后一个有效元素的下标 + 1（也可以指定参数1数组的元素总个数）

  - timeout: 指定poll函数的阻塞时长

    - -1：一直阻塞，直到检测的集合中有就绪的文件描述符（有事件产生）解除阻塞
    - 0：不阻塞，不管检测集合中有没有已就绪的文件描述符，函数马上返回
    - 大于0：阻塞指定的毫秒（ms）数之后，解除阻塞

- 函数返回值：

  - 失败： 返回-1
  - 成功：返回一个大于0的整数，表示检测的集合中已就绪的文件描述符的总个数

## 8.2 测试代码

### **服务器端**

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/select.h>
#include <poll.h>

int main()
{
    // 1.创建套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket");
        exit(0);
    }
    // 2. 绑定 ip, port
    struct sockaddr_in addr;
    addr.sin_port = htons(9999);
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    int ret = bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("bind");
        exit(0);
    }
    // 3. 监听
    ret = listen(lfd, 100);
    if(ret == -1)
    {
        perror("listen");
        exit(0);
    }
    
    // 4. 等待连接 -> 循环
    // 检测 -> 读缓冲区, 委托内核去处理
    // 数据初始化, 创建自定义的文件描述符集
    struct pollfd fds[1024];
    // 初始化
    for(int i=0; i<1024; ++i)
    {
        fds[i].fd = -1;
        fds[i].events = POLLIN;
    }
    fds[0].fd = lfd;

    int maxfd = 0;
    while(1)
    {
        // 委托内核检测
        ret = poll(fds, maxfd+1, -1);
        if(ret == -1)
        {
            perror("select");
            exit(0);
        }

        // 检测的度缓冲区有变化
        // 有新连接
        if(fds[0].revents & POLLIN)
        {
            // 接收连接请求
            struct sockaddr_in sockcli;
            int len = sizeof(sockcli);
            // 这个accept是不会阻塞的
            int connfd = accept(lfd, (struct sockaddr*)&sockcli, &len);
            // 委托内核检测connfd的读缓冲区
            int i;
            for(i=0; i<1024; ++i)
            {
                if(fds[i].fd == -1)
                {
                    fds[i].fd = connfd;
                    break;
                }
            }
            maxfd = i > maxfd ? i : maxfd;
        }
        // 通信, 有客户端发送数据过来
        for(int i=1; i<=maxfd; ++i)
        {
            // 如果在集合中, 说明读缓冲区有数据
            if(fds[i].revents & POLLIN)
            {
                char buf[128];
                int ret = read(fds[i].fd, buf, sizeof(buf));
                if(ret == -1)
                {
                    perror("read");
                    exit(0);
                }
                else if(ret == 0)
                {
                    printf("对方已经关闭了连接...\n");
                    close(fds[i].fd);
                    fds[i].fd = -1;
                }
                else
                {
                    printf("客户端say: %s\n", buf);
                    write(fds[i].fd, buf, strlen(buf)+1);
                }
            }
        }
    }
    close(lfd);
    return 0;
}
```

从上面的测试代码可以得知，使用poll和select进行IO多路转接的处理思路是完全相同的，但是使用poll编写的代码看起来会更直观一些，select使用的位图的方式来标记要委托内核检测的文件描述符（每个比特位对应一个唯一的文件描述符），并且对这个`fd_set`类型的位图变量进行读写还需要借助一系列的宏函数，操作比较麻烦。而poll直接将要检测的文件描述符的相关信息封装到了一个结构体`struct pollfd`中，我们可以直接读写这个结构体变量。

另外poll的第二个参数有两种赋值方式，但是都和第一个参数的数组有关系：

- 使用参数1数组的元素个数
- 使用参数1数组中存储的最后一个有效元素对应的下标值 + 1

内核会根据第二个参数传递的值对参数1数组中的文件描述符进行线性遍历，这一点和select也是类似的。

### **客户端**

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建用于通信的套接字
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 连接服务器
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;  // ipv4
    addr.sin_port = htons(9999);   // 服务器监听的端口, 字节序应该是网络字节序
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr.s_addr);
    int ret = connect(fd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("connect");
        exit(0);
    }

    // 通信
    while(1)
    {
        // 读数据
        char recvBuf[1024];
        // 写数据
        // sprintf(recvBuf, "data: %d\n", i++);
        fgets(recvBuf, sizeof(recvBuf), stdin);
        write(fd, recvBuf, strlen(recvBuf)+1);
        // 如果客户端没有发送数据, 默认阻塞
        read(fd, recvBuf, sizeof(recvBuf));
        printf("recv buf: %s\n", recvBuf);
        sleep(1);
    }
    // 释放资源
    close(fd); 
    return 0;
}
```

客户端不需要使用IO多路转接进行处理，因为客户端和服务器的对应关系是 1：N，也就是说客户端是比较专一的，只能和一个连接成功的服务器通信。

# 九、IO多路转接（复用）之epoll

## 9.1 概述

epoll 全称 eventpoll，是 linux 内核实现IO多路转接/复用（IO multiplexing）的一个实现。IO多路转接的意思是在一个操作里同时监听多个输入输出源，在其中一个或多个输入输出源可用的时候返回，然后对其的进行读写操作。epoll是select和poll的升级版，相较于这两个前辈，epoll改进了工作方式，因此它更加高效。

- `对于待检测集合select和poll是基于线性方式处理的，epoll是基于红黑树来管理待检测集合的。`
- `select和poll每次都会线性扫描整个待检测集合，集合越大速度越慢，epoll使用的是回调机制，效率高，处理效率也不会随着检测集合的变大而下降`
- `select和poll工作过程中存在内核/用户空间数据的频繁拷贝问题，在epoll中内核和用户区使用的是共享内存（基于mmap内存映射区实现），省去了不必要的内存拷贝。`
- `程序猿需要对select和poll返回的集合进行判断才能知道哪些文件描述符是就绪的，通过epoll可以直接得到已就绪的文件描述符集合，无需再次检测`
- 使用epoll没有最大文件描述符的限制，仅受系统中进程能打开的最大文件数目限制

当多路复用的文件数量庞大、IO流量频繁的时候，一般不太适合使用select()和poll()，这种情况下select()和poll()表现较差，推荐使用epoll()。

## 9.2 操作函数

在epoll中一共提供是三个API函数，分别处理不同的操作，函数原型如下：

```c
#include <sys/epoll.h>
// 创建epoll实例，通过一棵红黑树管理待检测集合
int epoll_create(int size);
// 管理红黑树上的文件描述符(添加、修改、删除)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 检测epoll树中是否有就绪的文件描述符
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

select/poll低效的原因之一是将“添加/维护待检测任务”和“阻塞进程/线程”两个步骤合二为一。每次调用select都需要这两步操作，然而大多数应用场景中，需要监视的socket个数相对固定，并不需要每次都修改。epoll将这两个操作分开，先用`epoll_ctl()`维护等待队列，再调用`epoll_wait()`阻塞进程（解耦）。通过下图的对比显而易见，epoll的效率得到了提升。

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210403181746358.png)](https://subingwen.cn/linux/epoll/image-20210403181746358.png)

`epoll_create()`函数的作用是创建一个红黑树模型的实例，用于管理待检测的文件描述符的集合。

```c
int epoll_create(int size);
```

- 函数参数 size：在Linux内核2.6.8版本以后，这个参数是被忽略的，只需要指定一个大于0的数值就可以了。
- 函数返回值：
  - 失败：返回-1
  - 成功：返回一个有效的文件描述符，通过这个文件描述符就可以访问创建的epoll实例了

`epoll_ctl()`函数的作用是管理红黑树实例上的节点，可以进行添加、删除、修改操作。

```c
// 联合体, 多个变量共用同一块内存        
typedef union epoll_data {
 	void        *ptr;
	int          fd;	// 通常情况下使用这个成员, 和epoll_ctl的第三个参数相同即可
	uint32_t     u32;
	uint64_t     u64;
} epoll_data_t;

struct epoll_event {
	uint32_t     events;      /* Epoll events */
	epoll_data_t data;        /* User data variable */
};
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

- 函数参数：
  - epfd：epoll_create() 函数的返回值，通过这个参数找到epoll实例
  - op：这是一个枚举值，控制通过该函数执行什么操作
    - `EPOLL_CTL_ADD`：往epoll模型中添加新的节点
    - `EPOLL_CTL_MOD`：修改epoll模型中已经存在的节点
    - `EPOLL_CTL_DEL`：删除epoll模型中的指定的节点
  - fd：文件描述符，即要添加/修改/删除的文件描述符
  - event：epoll事件，用来修饰第三个参数对应的文件描述符的，指定检测这个文件描述符的什么事件
    - events：委托epoll检测的事件
      - `EPOLLIN`：读事件, 接收数据, 检测读缓冲区，如果有数据该文件描述符就绪
      - `EPOLLOUT`：写事件, 发送数据, 检测写缓冲区，如果可写该文件描述符就绪
      - `EPOLLERR`：异常事件
    - data：用户数据变量，这是一个联合体类型，通常情况下使用里边的`fd`成员，用于存储待检测的文件描述符的值，在调用`epoll_wait()`函数的时候这个值会被传出。
- 函数返回值：
  - 失败：返回-1
  - 成功：返回0

`epoll_wait()`函数的作用是检测创建的epoll实例中有没有就绪的文件描述符。

```c
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

- 函数参数：
  - epfd：epoll_create() 函数的返回值, 通过这个参数找到epoll实例
  - events：传出参数, 这是一个结构体数组的地址, 里边存储了已就绪的文件描述符的信息
  - maxevents：修饰第二个参数, 结构体数组的容量（元素个数）
  - timeout：如果检测的epoll实例中没有已就绪的文件描述符，该函数阻塞的时长, 单位ms 毫秒
    - 0：函数不阻塞，不管epoll实例中有没有就绪的文件描述符，函数被调用后都直接返回
    - 大于0：如果epoll实例中没有已就绪的文件描述符，函数阻塞对应的毫秒数再返回
    - -1：函数一直阻塞，直到epoll实例中有已就绪的文件描述符之后才解除阻塞
- 函数返回值：
  - 成功：
    - 等于0：函数是阻塞被强制解除了, 没有检测到满足条件的文件描述符
    - 大于0：检测到的已就绪的文件描述符的总个数
  - 失败：返回-1

## 9.3 epoll的使用

### 9.3.1 操作步骤

在服务器端使用epoll进行IO多路转接的操作步骤如下：

1. `创建监听的套接字`

   ```c
   int lfd = socket(AF_INET, SOCK_STREAM, 0);
   ```

2. `设置端口复用（可选）`

   ```c
   int opt = 1;
   setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
   ```

3. `使用本地的IP与端口和监听的套接字进行绑定`

   ```c
   int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
   ```

4. `给监听的套接字设置监听`

   ```c
   listen(lfd, 128);
   ```

5. `创建epoll实例对象`

   ```c
   int epfd = epoll_create(100);
   ```

6. `将用于监听的套接字添加到epoll实例中`

   ```c
   struct epoll_event ev;
   ev.events = EPOLLIN;    // 检测lfd读读缓冲区是否有数据
   ev.data.fd = lfd;
   int ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
   ```

7. `检测添加到epoll实例中的文件描述符是否已就绪，并将这些已就绪的文件描述符进行处理`

   ```c
   int num = epoll_wait(epfd, evs, size, -1);
   ```

   - `如果是监听的文件描述符，和新客户端建立连接，将得到的文件描述符添加到epoll实例中`

     ```c
     int cfd = accept(curfd, NULL, NULL);
     ev.events = EPOLLIN;
     ev.data.fd = cfd;
     // 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
     epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
     ```

   - `如果是通信的文件描述符，和对应的客户端通信，如果连接已断开，将该文件描述符从epoll实例中删除`

     ```c
     int len = recv(curfd, buf, sizeof(buf), 0);
     if(len == 0)
     {
         // 将这个文件描述符从epoll模型中删除
         epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
         close(curfd);
     }
     else if(len > 0)
     {
         send(curfd, buf, len, 0);
     }
     ```

8. 重复第7步的操作

### 9.3.2 示例代码

```c
#include <stdio.h>
#include <ctype.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>

// server
int main(int argc, const char* argv[])
{
    // 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(9999);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 本地多有的ＩＰ
    
    // 设置端口复用
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 绑定端口
    int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }

    // 监听
    ret = listen(lfd, 64);
    if(ret == -1)
    {
        perror("listen error");
        exit(1);
    }

    // 现在只有监听的文件描述符
    // 所有的文件描述符对应读写缓冲区状态都是委托内核进行检测的epoll
    // 创建一个epoll模型
    int epfd = epoll_create(100);
    if(epfd == -1)
    {
        perror("epoll_create");
        exit(0);
    }

    // 往epoll实例中添加需要检测的节点, 现在只有监听的文件描述符
    struct epoll_event ev;
    ev.events = EPOLLIN;    // 检测lfd读读缓冲区是否有数据
    ev.data.fd = lfd;
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
    if(ret == -1)
    {
        perror("epoll_ctl");
        exit(0);
    }

    struct epoll_event evs[1024];
    int size = sizeof(evs) / sizeof(struct epoll_event);
    // 持续检测
    while(1)
    {
        // 调用一次, 检测一次
        int num = epoll_wait(epfd, evs, size, -1);
        for(int i=0; i<num; ++i)
        {
            // 取出当前的文件描述符
            int curfd = evs[i].data.fd;
            // 判断这个文件描述符是不是用于监听的
            if(curfd == lfd)
            {
                // 建立新的连接
                int cfd = accept(curfd, NULL, NULL);
                // 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
                ev.events = EPOLLIN;    // 读缓冲区是否有数据
                ev.data.fd = cfd;
                ret = epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
                if(ret == -1)
                {
                    perror("epoll_ctl-accept");
                    exit(0);
                }
            }
            else
            {
                // 处理通信的文件描述符
                // 接收数据
                char buf[1024];
                memset(buf, 0, sizeof(buf));
                int len = recv(curfd, buf, sizeof(buf), 0);
                if(len == 0)
                {
                    printf("客户端已经断开了连接\n");
                    // 将这个文件描述符从epoll模型中删除
                    epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                    close(curfd);
                }
                else if(len > 0)
                {
                    printf("客户端say: %s\n", buf);
                    send(curfd, buf, len, 0);
                }
                else
                {
                    perror("recv");
                    exit(0);
                } 
            }
        }
    }

    return 0;
}
```

当在服务器端循环调用`epoll_wait()`的时候，就会得到一个就绪列表，并通过该函数的第二个参数传出：

```c
struct epoll_event evs[1024];
int num = epoll_wait(epfd, evs, size, -1);
```

每当`epoll_wait()`函数返回一次，在`evs`中最多可以存储`size`个已就绪的文件描述符信息，但是在这个数组中实际存储的有效元素个数为`num`个，如果在这个epoll实例的红黑树中已就绪的文件描述符很多，并且`evs`数组无法将这些信息全部传出，那么这些信息会在下一次`epoll_wait()`函数返回的时候被传出。

通过`evs`数组被传递出的每一个有效元素里边都包含了已就绪的文件描述符的相关信息，这些信息并不是凭空得来的，这取决于我们在往epoll实例中添加节点的时候，往节点中初始化了哪些数据：

```c
struct epoll_event ev;
// 节点初始化
ev.events = EPOLLIN;    
ev.data.fd = lfd;	// 使用了联合体中 fd 成员
// 添加待检测节点到epoll实例中
int ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
```

在添加节点的时候，需要对这个`struct epoll_event`类型的节点进行初始化，当这个节点对应的文件描述符变为已就绪状态，这些被传入的初始化信息就会被原样传出，这个对应关系必须要搞清楚。

## 9.4 epoll的工作模式

### 9.4.1 水平模式

水平模式可以简称为LT模式，`LT（level triggered）是缺省的工作方式，并且同时支持block和no-block socket`。在这种做法中，内核通知使用者哪些文件描述符已经就绪，之后就可以对这些已就绪的文件描述符进行IO操作了。`如果我们不作任何操作，内核还是会继续通知使用者`。

**水平模式的特点：**

- 读事件：

  如果文件描述符对应的读缓冲区还有数据，读事件就会被触发，epoll_wait()解除阻塞

  - 当读事件被触发，epoll_wait()解除阻塞，之后就可以接收数据了
  - 如果接收数据的buf很小，不能全部将缓冲区数据读出，那么读事件会继续被触发，直到数据被全部读出，如果接收数据的内存相对较大，读数据的效率也会相对较高（减少了读数据的次数）
  - `因为读数据是被动的，必须要通过读事件才能知道有数据到达了，因此对于读事件的检测是必须的`

- 写事件：

  如果文件描述符对应的写缓冲区可写，写事件就会被触发，epoll_wait()解除阻塞

  - 当写事件被触发，epoll_wait()解除阻塞，之后就可以将数据写入到写缓冲区了
  - `写事件的触发发生在写数据之前而不是之后`，被写入到写缓冲区中的数据是由内核自动发送出去的
  - 如果写缓冲区没有被写满，写事件会一直被触发
  - `因为写数据是主动的，并且写缓冲区一般情况下都是可写的（缓冲区不满），因此对于写事件的检测不是必须的`

### 9.4.2 边沿模式

边沿模式可以简称为ET模式，`ET（edge-triggered）是高速工作方式，只支持no-block socket`。在这种模式下，`当文件描述符从未就绪变为就绪时，内核会通过epoll通知使用者。然后它会假设使用者知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知（only once）`。如果我们对这个文件描述符做IO操作，从而导致它再次变成未就绪，当这个未就绪的文件描述符再次变成就绪状态，内核会再次进行通知，并且还是只通知一次。`ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高`。

**边沿模式的特点:**

- 读事件：

  当读缓冲区有新的数据进入，读事件被触发一次，没有新数据不会触发该事件

  - 如果有新数据进入到读缓冲区，读事件被触发，epoll_wait()解除阻塞
  - 读事件被触发，可以通过调用read()/recv()函数将缓冲区数据读出
    - `如果数据没有被全部读走，并且没有新数据进入，读事件不会再次触发，只通知一次`
    - `如果数据被全部读走或者只读走一部分，此时有新数据进入，读事件被触发，并且只通知一次`

- 写事件：

  当写缓冲区状态可写，写事件只会触发一次

  - 如果写缓冲区被检测到可写，写事件被触发，epoll_wait()解除阻塞
  - 写事件被触发，就可以通过调用write()/send()函数，将数据写入到写缓冲区中
    - 写缓冲区从不满到被写满，期间写事件只会被触发一次
    - 写缓冲区从满到不满，状态变为可写，写事件只会被触发一次

综上所述：epoll的边沿模式下 epoll_wait()检测到文件描述符有新事件才会通知，如果不是新的事件就不通知，通知的次数比水平模式少，效率比水平模式要高。

#### 9.4.2.1 ET模式的设置

边沿模式不是默认的epoll模式，需要额外进行设置。epoll设置边沿模式是非常简单的，epoll管理的红黑树示例中每个节点都是`struct epoll_event`类型，只需要将`EPOLLET`添加到结构体的`events`成员中即可：

```c
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;	// 设置边沿模式
```

示例代码如下：

```c
int num = epoll_wait(epfd, evs, size, -1);
for(int i=0; i<num; ++i)
{
    // 取出当前的文件描述符
    int curfd = evs[i].data.fd;
    // 判断这个文件描述符是不是用于监听的
    if(curfd == lfd)
    {
        // 建立新的连接
        int cfd = accept(curfd, NULL, NULL);
        // 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
        // 读缓冲区是否有数据, 并且将文件描述符设置为边沿模式
        struct epoll_event ev;
        ev.events = EPOLLIN | EPOLLET;   
        ev.data.fd = cfd;
        ret = epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
        if(ret == -1)
        {
            perror("epoll_ctl-accept");
            exit(0);
        }
    }
}
```

#### 9.4.2.2 设置非阻塞

对于写事件的触发一般情况下是不需要进行检测的，因为写缓冲区大部分情况下都是有足够的空间可以进行数据的写入。对于读事件的触发就必须要检测了，因为服务器也不知道客户端什么时候发送数据，如果使用epoll的边沿模式进行读事件的检测，有新数据达到只会通知一次，那么必须要保证得到通知后将数据全部从读缓冲区中读出。那么，应该如何读这些数据呢？

- 方式1：准备一块特别大的内存，用于存储从读缓冲区中读出的数据，但是这种方式有很大的弊端：

  - 内存的大小没有办法界定，太大浪费内存，太小又不够用
  - 系统能够分配的最大堆内存也是有上限的，栈内存就更不必多言了

- 方式2：循环接收数据

  ```c
  int len = 0;
  while((len = recv(curfd, buf, sizeof(buf), 0)) > 0)
  {
      // 数据处理...
  }
  ```

  这样做也是有弊端的，因为套接字操作默认是阻塞的，当读缓冲区数据被读完之后，读操作就阻塞了也就是调用的`read()/recv()`函数被阻塞了，当前进程/线程被阻塞之后就无法处理其他操作了。

  要解决阻塞问题，就需要将套接字默认的阻塞行为修改为非阻塞，需要使用`fcntl()`函数进行处理：

  ```c
  // 设置完成之后, 读写都变成了非阻塞模式
  int flag = fcntl(cfd, F_GETFL);
  flag |= O_NONBLOCK;                                                        
  fcntl(cfd, F_SETFL, flag);
  ```

通过上述分析就可以得出一个结论：epoll在边沿模式下，必须要将套接字设置为非阻塞模式，但是，这样就会引发另外的一个bug，在非阻塞模式下，循环地将读缓冲区数据读到本地内存中，当缓冲区数据被读完了，调用的`read()/recv()`函数还会继续从缓冲区中读数据，此时函数调用就失败了，返回-1，对应的全局变量 errno 值为 `EAGAIN` 或者 `EWOULDBLOCK`如果打印错误信息会得到如下的信息：`Resource temporarily unavailable`

```c
// 非阻塞模式下recv() / read()函数返回值 len == -1
int len = recv(curfd, buf, sizeof(buf), 0);
if(len == -1)
{
    if(errno == EAGAIN)
    {
        printf("数据读完了...\n");
    }
    else
    {
        perror("recv");
        exit(0);
    }
}
```

#### 9.4.2.3 示例代码

```c
#include <stdio.h>
#include <ctype.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <fcntl.h>
#include <errno.h>

// server
int main(int argc, const char* argv[])
{
    // 创建监听的套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(9999);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 本地多有的ＩＰ
    // 127.0.0.1
    // inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr.s_addr);
    
    // 设置端口复用
    int opt = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 绑定端口
    int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if(ret == -1)
    {
        perror("bind error");
        exit(1);
    }

    // 监听
    ret = listen(lfd, 64);
    if(ret == -1)
    {
        perror("listen error");
        exit(1);
    }

    // 现在只有监听的文件描述符
    // 所有的文件描述符对应读写缓冲区状态都是委托内核进行检测的epoll
    // 创建一个epoll模型
    int epfd = epoll_create(100);
    if(epfd == -1)
    {
        perror("epoll_create");
        exit(0);
    }

    // 往epoll实例中添加需要检测的节点, 现在只有监听的文件描述符
    struct epoll_event ev;
    ev.events = EPOLLIN;    // 检测lfd读读缓冲区是否有数据
    ev.data.fd = lfd;
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);
    if(ret == -1)
    {
        perror("epoll_ctl");
        exit(0);
    }


    struct epoll_event evs[1024];
    int size = sizeof(evs) / sizeof(struct epoll_event);
    // 持续检测
    while(1)
    {
        // 调用一次, 检测一次
        int num = epoll_wait(epfd, evs, size, -1);
        printf("==== num: %d\n", num);

        for(int i=0; i<num; ++i)
        {
            // 取出当前的文件描述符
            int curfd = evs[i].data.fd;
            // 判断这个文件描述符是不是用于监听的
            if(curfd == lfd)
            {
                // 建立新的连接
                int cfd = accept(curfd, NULL, NULL);
                // 将文件描述符设置为非阻塞
                // 得到文件描述符的属性
                int flag = fcntl(cfd, F_GETFL);
                flag |= O_NONBLOCK;
                fcntl(cfd, F_SETFL, flag);
                // 新得到的文件描述符添加到epoll模型中, 下一轮循环的时候就可以被检测了
                // 通信的文件描述符检测读缓冲区数据的时候设置为边沿模式
                ev.events = EPOLLIN | EPOLLET;    // 读缓冲区是否有数据
                ev.data.fd = cfd;
                ret = epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &ev);
                if(ret == -1)
                {
                    perror("epoll_ctl-accept");
                    exit(0);
                }
            }
            else
            {
                // 处理通信的文件描述符
                // 接收数据
                char buf[5];
                memset(buf, 0, sizeof(buf));
                // 循环读数据
                while(1)
                {
                    int len = recv(curfd, buf, sizeof(buf), 0);
                    if(len == 0)
                    {
                        // 非阻塞模式下和阻塞模式是一样的 => 判断对方是否断开连接
                        printf("客户端断开了连接...\n");
                        // 将这个文件描述符从epoll模型中删除
                        epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                        close(curfd);
                        break;
                    }
                    else if(len > 0)
                    {
                        // 通信
                        // 接收的数据打印到终端
                        write(STDOUT_FILENO, buf, len);
                        // 发送数据
                        send(curfd, buf, len, 0);
                    }
                    else
                    {
                        // len == -1
                        if(errno == EAGAIN)
                        {
                            printf("数据读完了...\n");
                            break;
                        }
                        else
                        {
                            perror("recv");
                            exit(0);
                        }
                    }
                }
            }
        }
    }

    return 0;
}
```

# 附2：Qt中多线程的使用

在进行桌面应用程序开发的时候， 假设应用程序在某些情况下需要处理比较复杂的逻辑， 如果只有一个线程去处理，就会导致窗口卡顿，无法处理用户的相关操作。这种情况下就需要使用多线程，其中一个线程处理窗口事件，其他线程进行逻辑运算，多个线程各司其职，不仅可以提高用户体验还可以提升程序的执行效率。

在qt中使用了多线程，有些事项是需要额外注意的：

- `默认的线程在Qt中称之为窗口线程，也叫主线程，负责窗口事件处理或者窗口控件数据的更新`
- `子线程负责后台的业务逻辑处理，子线程中不能对窗口对象做任何操作，这些事情需要交给窗口线程处理`
- `主线程和子线程之间如果要进行数据的传递，需要使用Qt中的信号槽机制`

## 2.1 线程类 QThread

Qt中提供了一个线程类，通过这个类就可以创建子线程了，Qt中一共提供了两种创建子线程的方式，后边会依次介绍其使用方式。先来看一下这个类中提供的一些常用API函数：

### 2.1.1 常用共用成员函数

```c++
// QThread 类常用 API
// 构造函数
QThread::QThread(QObject *parent = Q_NULLPTR);
// 判断线程中的任务是不是处理完毕了
bool QThread::isFinished() const;
// 判断子线程是不是在执行任务
bool QThread::isRunning() const;

// Qt中的线程可以设置优先级
// 得到当前线程的优先级
Priority QThread::priority() const;
void QThread::setPriority(Priority priority);
优先级:
    QThread::IdlePriority         --> 最低的优先级
    QThread::LowestPriority
    QThread::LowPriority
    QThread::NormalPriority
    QThread::HighPriority
    QThread::HighestPriority
    QThread::TimeCriticalPriority --> 最高的优先级
    QThread::InheritPriority      --> 子线程和其父线程的优先级相同, 默认是这个
// 退出线程, 停止底层的事件循环
// 退出线程的工作函数
void QThread::exit(int returnCode = 0);
// 调用线程退出函数之后, 线程不会马上退出因为当前任务有可能还没有完成, 调回用这个函数是
// 等待任务完成, 然后退出线程, 一般情况下会在 exit() 后边调用这个函数
bool QThread::wait(unsigned long time = ULONG_MAX);
```

### 2.1.2 信号槽

```c++
// 和调用 exit() 效果是一样的
// 调用这个函数之后, 再调用 wait() 函数
[slot] void QThread::quit();
// 启动子线程
[slot] void QThread::start(Priority priority = InheritPriority);
// 线程退出, 可能是会马上终止线程, 一般情况下不使用这个函数
[slot] void QThread::terminate();

// 线程中执行的任务完成了, 发出该信号
// 任务函数中的处理逻辑执行完毕了
[signal] void QThread::finished();
// 开始工作之前发出这个信号, 一般不使用
[signal] void QThread::started();
```

### 2.1.3 静态函数

```c++
// 返回一个指向管理当前执行线程的QThread的指针
[static] QThread *QThread::currentThread();
// 返回可以在系统上运行的理想线程数 == 和当前电脑的 CPU 核心数相同
[static] int QThread::idealThreadCount();
// 线程休眠函数
[static] void QThread::msleep(unsigned long msecs);	// 单位: 毫秒
[static] void QThread::sleep(unsigned long secs);	// 单位: 秒
[static] void QThread::usleep(unsigned long usecs);	// 单位: 微秒
```

### 2.1.4 任务处理函数

```c++
// 子线程要处理什么任务, 需要写到 run() 中
[virtual protected] void QThread::run()
```

这个`run()`是一个虚函数，如果想让创建的子线程执行某个任务，需要写一个子类让其继承`QThread`，并且在子类中重写父类的`run()`方法，函数体就是对应的任务处理流程。另外，这个函数是一个受保护的成员函数，不能够在类的外部调用，如果想要让线程执行这个函数中的业务流程，需要通过当前线程对象调用槽函数`start()`启动子线程，当子线程被启动，这个`run()`函数也就在线程内部被调用了

## 2.2 使用方式1

### 2.2.1 操作步骤

Qt中提供的多线程的第一种使用方式的特点是： 简单。操作步骤如下：

1. 需要创建一个线程类的子类，让其继承QT中的线程类 QThread，比如:

   ```c++
   class MyThread:public QThread
   {
       ......
   }
   ```

2. 重写父类的 run() 方法，在该函数内部编写子线程要处理的具体的业务流程

   ```c++
   class MyThread:public QThread
   {
       ......
    protected:
       void run()
       {
           ........
       }
   }
   ```

3. 在主线程中创建子线程对象，new 一个就可以了

   ```c++
   c++MyThread * subThread = new MyThread;
   ```

4. 启动子线程, 调用 start() 方法

   ```c++
   subThread->start();
   ```

不能在类的外部调用run() 方法启动子线程，在外部调用start()相当于让run()开始运行

当子线程别创建出来之后，父子线程之间的通信可以通过信号槽的方式，注意事项:

- 在Qt中在子线程中不要操作程序中的窗口类型对象, 不允许, 如果操作了程序就挂了
- 只有主线程才能操作程序中的窗口对象, 默认的线程就是主线程, 自己创建的就是子线程

### 2.2.2 示例代码

举一个简单的数数的例子，假如只有一个线程，让其一直数数，会发现数字并不会在窗口中时时更新，并且这时候如果用户使用鼠标拖动窗口，就会出现无响应的情况，使用多线程就不会出现这样的现象了。

在下面的窗口中，点击按钮开始在子线程中数数，让后通过信号槽机制将数据传递给UI线程，通过UI线程将数据更新到窗口中。

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/1.gif)](https://subingwen.cn/qt/thread/1.gif)

#### **mythread.h**

```c++
#ifndef MYTHREAD_H
#define MYTHREAD_H

#include <QThread>

class MyThread : public QThread
{
    Q_OBJECT
public:
    explicit MyThread(QObject *parent = nullptr);

protected:
    void run();

signals:
    // 自定义信号, 传递数据
    void curNumber(int num);

public slots:
};

#endif // MYTHREAD_H
```

#### **mythread.cpp**

```c++
#include "mythread.h"
#include <QDebug>

MyThread::MyThread(QObject *parent) : QThread(parent)
{

}

void MyThread::run()
{
    qDebug() << "当前线程对象的地址: " << QThread::currentThread();

    int num = 0;
    while(1)
    {
        emit curNumber(num++);
        if(num == 10000000)
        {
            break;
        }
        QThread::usleep(1);
    }
    qDebug() << "run() 执行完毕, 子线程退出...";
}
```

#### **mainwindow.cpp**

```c++
#include "mainwindow.h"
#include "ui_mainwindow.h"
#include "mythread.h"
#include <QDebug>

MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    qDebug() << "主线程对象地址:  " << QThread::currentThread();
    // 创建子线程
    MyThread* subThread = new MyThread;

    connect(subThread, &MyThread::curNumber, this, [=](int num)
    {
        ui->label->setNum(num);
    });

    connect(ui->startBtn, &QPushButton::clicked, this, [=]()
    {
        // 启动子线程
        subThread->start();
    });
}

MainWindow::~MainWindow()
{
    delete ui;
}
```

这种在程序中添加子线程的方式是非常简单的，但是也有弊端，`假设要在一个子线程中处理多个任务，所有的处理逻辑都需要写到run()函数中，这样该函数中的处理逻辑就会变得非常混乱，不太容易维护`。

## 2.3 使用方式2

### 2.3.1 操作步骤

Qt提供的第二种线程的创建方式弥补了第一种方式的缺点，用起来更加灵活，但是这种方式写起来会相对复杂一些，其具体操作步骤如下：

1. 创建一个新的类，让这个类从QObject派生

   ```c
   class MyWork:public QObject
   {
       .......
   }
   ```

2. 在这个类中添加一个公共的成员函数，函数体就是我们要子线程中执行的业务逻辑

   ```c++
   class MyWork:public QObject
   {
   public:
       .......
       // 函数名自己指定, 叫什么都可以, 参数可以根据实际需求添加
       void working();
   }
   ```

3. 在主线程中创建一个QThread对象, 这就是子线程的对象

   ```c++
   QThread* sub = new QThread;
   ```

4. 在主线程中创建工作的类对象（`千万不要指定给创建的对象指定父对象`）

   ```c++
   MyWork* work = new MyWork(this);    // error
   MyWork* work = new MyWork;          // ok
   ```

5. 将MyWork对象移动到创建的子线程对象中, 需要调用QObject类提供的`moveToThread()`方法

   ```c++
   // void QObject::moveToThread(QThread *targetThread);
   // 如果给work指定了父对象, 这个函数调用就失败了
   // 提示： QObject::moveToThread: Cannot move objects with a parent
   work->moveToThread(sub);	// 移动到子线程中工作
   ```

6. 启动子线程，调用 `start()`, 这时候线程启动了, 但是移动到线程中的对象并没有工作

7. 调用MyWork类对象的工作函数，让这个函数开始执行，这时候是在移动到的那个子线程中运行的

### 2.3.2 示例代码

假设函数处理上面在程序中数数的这个需求，具体的处理代码如下：

#### **mywork.h**

```c++
#ifndef MYWORK_H
#define MYWORK_H

#include <QObject>

class MyWork : public QObject
{
    Q_OBJECT
public:
    explicit MyWork(QObject *parent = nullptr);

    // 工作函数
    void working();

signals:
    void curNumber(int num);

public slots:
};

#endif // MYWORK_H
```

#### **mywork.cpp**

```c++
#include "mywork.h"
#include <QDebug>
#include <QThread>

MyWork::MyWork(QObject *parent) : QObject(parent)
{

}

void MyWork::working()
{
    qDebug() << "当前线程对象的地址: " << QThread::currentThread();

    int num = 0;
    while(1)
    {
        emit curNumber(num++);
        if(num == 10000000)
        {
            break;
        }
        QThread::usleep(1);
    }
    qDebug() << "run() 执行完毕, 子线程退出...";
}
```

#### **mainwindow.cpp**

```c++
#include "mainwindow.h"
#include "ui_mainwindow.h"
#include <QThread>
#include "mywork.h"
#include <QDebug>

MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    qDebug() << "主线程对象的地址: " << QThread::currentThread();

    // 创建线程对象
    QThread* sub = new QThread;
    // 创建工作的类对象
    // 千万不要指定给创建的对象指定父对象
    // 如果指定了: QObject::moveToThread: Cannot move objects with a parent
    MyWork* work = new MyWork;
    // 将工作的类对象移动到创建的子线程对象中
    work->moveToThread(sub);
    // 启动线程
    sub->start();
    // 让工作的对象开始工作, 点击开始按钮, 开始工作
    connect(ui->startBtn, &QPushButton::clicked, work, &MyWork::working);
    // 显示数据
    connect(work, &MyWork::curNumber, this, [=](int num)
    {
        ui->label->setNum(num);
    });
}

MainWindow::~MainWindow()
{
    delete ui;
}
```

使用这种多线程方式，假设有多个不相关的业务流程需要被处理，那么就可以创建多个类似于`MyWork`的类，将业务流程放多类的公共成员函数中，然后将这个业务类的实例对象移动到对应的子线程中`moveToThread()`就可以了，这样可以让编写的程序更加灵活，可读性更强，更易于维护。

# 十、基于UDP的套接字通信

udp是一个面向无连接的，不安全的，报式传输层协议，udp的通信过程默认也是阻塞的。

- `UDP通信不需要建立连接` ，因此不需要进行connect()操作
- `UDP通信过程中，每次都需要指定数据接收端的IP和端口`，和发快递差不多
- `UDP不对收到的数据进行排序，在UDP报文的首部中并没有关于数据顺序的信息`
- `UDP对接收到的数据报不回复确认信息，发送端不知道数据是否被正确接收，也不会重发数据。`
- `如果发生了数据丢失，不存在丢一半的情况，如果丢当前这个数据包就全部丢失了`

## 10.1 通信流程

使用UDP进行通信，服务器和客户端的处理步骤比TCP要简单很多，并且两端是对等的 （通信的处理流程几乎是一样的），也就是说并没有严格意义上的客户端和服务器端。UDP的通信流程如下：

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/udp.jpg)](https://subingwen.cn/linux/udp/udp.jpg)

### 10.1.1 服务器端

**假设服务器端是接收数据的角色：**

1. 创建通信的套接字

   ```c
   // 第二个参数是 SOCK_DGRAM, 第三个参数0表示使用报式协议中的udp
   int fd = socket(AF_INET, SOCK_DGRAM, 0);
   ```

2. 使用通信的套接字和本地的IP和端口绑定，IP和端口需要转换为大端(可选)

   ```c
   bind();
   ```

3. 通信

   ```c
   // 接收数据
   recvfrom();
   // 发送数据
   sendto();
   ```

4. 关闭套接字（文件描述符）

   ```c
   close(fd);
   ```

### 10.1.2 客户端

**假设客户端是发送数据的角色：**

1. 创建通信的套接字

   ```c
   // 第二个参数是 SOCK_DGRAM, 第三个参数0表示使用报式协议中的udp
   int fd = socket(AF_INET, SOCK_DGRAM, 0);
   ```

2. 通信

   ```c
   // 接收数据
   recvfrom();
   // 发送数据
   sendto();
   ```

3. 关闭套接字（文件描述符）

   ```c
   close(fd);
   ```

在UDP通信过程中，`哪一端是接收数据的角色，那么这个接收端就必须绑定一个固定的端口`，如果某一端不需要接收数据，这个绑定操作就可以省略不写了，通信的套接字会自动绑定一个随机端口。

## 10.2 通信函数

基于UDP进行套接字通信，创建套接字的函数还是`socket()`但是第二个参数的值需要指定为`SOCK_DGRAM`，通过该参数指定要创建一个基于报式传输协议的套接字，最后一个参数指定为0表示使用报式协议中的UDP协议。

```c
int socket(int domain, int type, int protocol);
```

- 参数:
  - domain：地址族协议，AF_INET -> IPv4，AF_INET6-> IPv6
  - type：使用的传输协议类型，报式传输协议需要指定为 SOCK_DGRAM
  - protocol：指定为0，表示使用的默认报式传输协议为 UDP
- 返回值：函数调用成功返回一个可用的文件描述符（大于0），调用失败返回-1

另外进行UDP通信，通信过程虽然默认还是阻塞的，但是通信函数和TCP不同，操作函数原型如下：

```c
// 接收数据, 如果没有数据,该函数阻塞
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);
```

- 参数:
  - sockfd: 基于udp的通信的文件描述符
  - buf: 指针指向的地址用来存储接收的数据
  - len: buf指针指向的内存的容量, 最多能存储多少字节
  - flags: 设置套接字属性，一般使用默认属性，指定为0即可
  - src_addr: 发送数据的一端的地址信息，IP和端口都存储在这里边, 是大端存储的
    - 如果这个参数中的信息对当前业务处理没有用处, 可以指定为NULL, 不保存这些信息
  - addrlen: 类似于accept() 函数的最后一个参数, 是一个传入传出参数
    - 传入的是src_addr参数指向的内存的大小, 传出的也是这块内存的大小
    - 如果src_addr参数指定为NULL, 这个参数也指定为NULL即可
- 返回值：成功返回接收的字节数，失败返回-1

```c
// 发送数据函数
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);
```

- 参数:
  - sockfd: 基于udp的通信的文件描述符
  - buf: 这个指针指向的内存中存储了要发送的数据
  - len: 要发送的数据的实际长度
  - flags: 设置套接字属性，一般使用默认属性，指定为0即可
  - dest_addr: 接收数据的一端对应的地址信息, 大端的IP和端口
  - addrlen: 参数 dest_addr 指向的内存大小
- 返回值：函数调用成功返回实际发送的字节数，调用失败返回-1

## 10.3 通信代码

在UDP通信过程中，服务器和客户端都可以作为数据的发送端和数据接收端，假设服务器端是被动接收数据，客户端是主动发送数据，那么在服务器端就必须绑定固定的端口了。

### 10.3.1 服务器端

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建通信的套接字
    int fd = socket(AF_INET, SOCK_DGRAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 通信的套接字和本地的IP与端口绑定
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);    // 大端
    addr.sin_addr.s_addr = INADDR_ANY;  // 0.0.0.0
    int ret = bind(fd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("bind");
        exit(0);
    }

    char buf[1024];
    char ipbuf[64];
    struct sockaddr_in cliaddr;
    int len = sizeof(cliaddr);
    // 3. 通信
    while(1)
    {
        // 接收数据
        memset(buf, 0, sizeof(buf));
        int rlen = recvfrom(fd, buf, sizeof(buf), 0, (struct sockaddr*)&cliaddr, &len);
        printf("客户端的IP地址: %s, 端口: %d\n",
               inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr, ipbuf, sizeof(ipbuf)),
               ntohs(cliaddr.sin_port));
        printf("客户端say: %s\n", buf);

        // 回复数据
        // 数据回复给了发送数据的客户端
        sendto(fd, buf, rlen, 0, (struct sockaddr*)&cliaddr, sizeof(cliaddr));
    }

    close(fd);

    return 0;
}
```

作为数据接收端，服务器端通过`bind()`函数绑定了固定的端口，然后基于这个固定的端口通过`recvfrom()`函数接收客户端发送的数据，同时通过这个函数也得到了数据发送端的地址信息（recvfrom的第三个参数），这样就可以通过得到的地址信息通过`sendto()`函数给客户端回复数据了。

### 10.3.2 客户端

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建通信的套接字
    int fd = socket(AF_INET, SOCK_DGRAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }
    
    // 初始化服务器地址信息
    struct sockaddr_in seraddr;
    seraddr.sin_family = AF_INET;
    seraddr.sin_port = htons(9999);    // 大端
    inet_pton(AF_INET, "192.168.1.100", &seraddr.sin_addr.s_addr);

    char buf[1024];
    char ipbuf[64];
    struct sockaddr_in cliaddr;
    int len = sizeof(cliaddr);
    int num = 0;
    // 2. 通信
    while(1)
    {
        sprintf(buf, "hello, udp %d....\n", num++);
        // 发送数据, 数据发送给了服务器
        sendto(fd, buf, strlen(buf)+1, 0, (struct sockaddr*)&seraddr, sizeof(seraddr));

        // 接收数据
        memset(buf, 0, sizeof(buf));
        recvfrom(fd, buf, sizeof(buf), 0, NULL, NULL);
        printf("服务器say: %s\n", buf);
        sleep(1);
    }

    close(fd);

    return 0;
}
```

作为数据发送端，客户端不需要绑定固定端口，客户端使用的端口是随机绑定的（也可以调用bind()函数手动进行绑定）。客户端在接收服务器端回复的数据的时候需要调用`recvfrom()`函数，因为客户端在发送数据之前就已经知道服务器绑定的固定的IP和端口信息了，所以接收服务器数据的时候就可以不保存服务器端的地址信息，直接将函数的最后两个参数指定为NULL即可。

# 十一、UDP特性之广播

## 11.1 广播的特点

广播的UDP的特性之一，`通过广播可以向子网中多台计算机发送消息，并且子网中所有的计算机都可以接收到发送方发送的消息`，每个广播消息都包含一个特殊的IP地址，这个IP中子网内主机标志部分的二进制全部为1 （即点分十进制IP的最后一部分是255）。点分十进制的IP地址每一部分是1字节，最大值为255，比如：`192.168.1.100`

- 前两部分`192.168`表示当前网络是局域网
- 第三部分`1`表示局域网中的某一个网段，最大值为 255
- 第四部分`100`用于标记当前网段中的某一台主机，最大值为255
- 每个网段都有一个特殊的广播地址，即：`192.168.xxx.255`

广播分为两端，即数据发送端和数据接收端，通过广播的方式发送数据，发送端和接收端的关系是 1:N

- `发送广播消息的一端，通过广播地址，可以将消息同时发送到局域网的多台主机上（数据接收端）`
- `在发送广播消息的时候，必须要把数据发送到广播地址上`
- `广播只能在局域网内使用，广域网是无法使用UDP进行广播的`
- 只要发送端在发送广播消息，数据接收端就能收到广播消息，消息的接收是无法拒绝的，除非将接收端的进程关闭，就接收不到了。

UDP的广播和日常生活中的广播是一样的，都是一种快速传播消息的方式，因此`广播的开销很小`，发送端使用一个广播地址，就可以将数据发送到多个接收数据的终端上，如果不使用广播，就需要进行多次发送才能将数据分别发送到不同的主机上。

## 11.2 设置广播属性

基于UDP虽然可以进行数据的广播，但是这个属性默认是关闭的，如果需要对数据进行广播，那么需要在广播端代码中开启广播属性，需要通过套接字选项函数进行设置，该函数原型为：

```c
int setsockopt(int sockfd, int level, int optname, 	const void *optval, socklen_t optlen);
```

- 参数:
  - sockfd：进行UDP通信的文件描述符
  - level: 套接字级别，需要设置为 `SOL_SOCKET`
  - optname：选项名，此处要设置udp的广播属性，该参数需要指定为：`SO_BROADCAST`
  - optval：如果是设置广播属性，该指针实际指向一块 **int** 类型的内存
    - 该整型值为0：关闭广播属性
    - 该整形值为1：打开广播属性
  - optlen：optval指针指向的内存大小，即：`sizeof(int)`
- 返回值：函数调用成功返回0，失败返回-1

## 11.3 广播通信流程

如果使用UDP在局域网范围内进行消息的广播，一般情况下广播端只发送数据，接收端只接受广播消息。因此在数据接收端需要绑定固定的端口，广播端则不需要手动绑定固定端口，自动随机绑定即可。

[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210407000319507.png)](https://subingwen.cn/linux/broadcast/image-20210407000319507.png)

- **数据发送端**

  1. `创建通信的套接字`

     ```c
     // 第二个参数是 SOCK_DGRAM, 第三个参数0表示使用报式协议中的udp
     int fd = socket(AF_INET, SOCK_DGRAM, 0);
     ```

  2. `主动发送数据不需要手动绑定固定端口（自动随机分配就可以了），因此直接设置广播属性`

     ```c
     int opt  = 1;
     setsockopt(fd, SOL_SOCKET, SO_BROADCAST, &opt, sizeof(opt));
     ```

  3. `使用广播地址发送广播数据到接收端绑定的固定端口上`

     ```c
     sendto();
     ```

  4. `关闭套接字（文件描述符）`

     ```c
     close(fd);
     ```

- **数据接收端**

  1. `创建通信的套接字`

     ```c
     // 第二个参数是 SOCK_DGRAM, 第三个参数0表示使用报式协议中的udp
     int fd = socket(AF_INET, SOCK_DGRAM, 0);
     ```

  2. `因为是被动接收数据的一端，所以必须要绑定固定的端口和本地IP地址`

     ```c
     bind();
     ```

  3. `接收广播消息`

     ```c
     recvfrom();
     ```

  4. `关闭套接字（文件描述符）`

     ```c
     close(fd);
     ```

## 11.4 通信代码

### **广播端**

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建通信的套接字
    int fd = socket(AF_INET, SOCK_DGRAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 设置广播属性
    int opt  = 1;
    setsockopt(fd, SOL_SOCKET, SO_BROADCAST, &opt, sizeof(opt));

    char buf[1024];
    struct sockaddr_in cliaddr;
    int len = sizeof(cliaddr);
    cliaddr.sin_family = AF_INET;
    cliaddr.sin_port = htons(9999); // 接收端需要绑定9999端口
    // 只要主机在237网段, 并且绑定了9999端口, 这个接收端就能收到广播消息
    inet_pton(AF_INET, "192.168.237.255", &cliaddr.sin_addr.s_addr);
    // 3. 通信
    int num = 0;
    while(1)
    {
        sprintf(buf, "hello, client...%d\n", num++);
        // 数据广播
        sendto(fd, buf, strlen(buf)+1, 0, (struct sockaddr*)&cliaddr, len);
        printf("发送的广播的数据: %s\n", buf);
        sleep(1);
    }

    close(fd);

    return 0;
}
```

注意事项：发送广播消息一端必须要开启UDP的广播属性，并且发送消息的地址必须是当前发送端所在网段的广播地址，这样才能通过调用一个消息发送函数将消息同时发送N台接收端主机上。

### **接收端**

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建通信的套接字
    int fd = socket(AF_INET, SOCK_DGRAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 通信的套接字和本地的IP与端口绑定
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);    // 大端
    addr.sin_addr.s_addr = INADDR_ANY;  // 0.0.0.0
    int ret = bind(fd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("bind");
        exit(0);
    }

    char buf[1024];
    // 3. 通信
    while(1)
    {
        // 接收广播消息
        memset(buf, 0, sizeof(buf));
        // 阻塞等待数据达到
        recvfrom(fd, buf, sizeof(buf), 0, NULL, NULL);
        printf("接收到的广播消息: %s\n", buf);
    }

    close(fd);

    return 0;
}
```

**对于接收广播消息的一端，必须要绑定固定的端口，并由广播端将广播消息发送到这个端口上，因此所有接收端都应绑定相同的端口，这样才能同时收到广播数据**。

# 十二、UDP特性之组播（多播）

## 12.1 组播的特点

组播也可以称之为多播这也是UDP的特性之一。`组播是主机间一对多的通讯模式，是一种允许一个或多个组播源发送同一报文到多个接收者的技术。`组播源将一份报文发送到特定的组播地址，组播地址不同于单播地址，它并不属于特定某个主机，而是属于一组主机。一个组播地址表示一个群组，需要接收组播报文的接收者都加入这个群组。

- 广播只能在局域网访问内使用，组播既可以在局域网中使用，也可以用于广域网
- 在发送广播消息的时候，连接到局域网的客户端不管想不想都会接收到广播数据，组播可以控制发送端的消息能够被哪些接收端接收，更灵活和人性化。
- 广播使用的是广播地址，组播需要使用组播地址。
- 广播和组播属性默认都是关闭的，如果使用需要通过setsockopt()函数进行设置。

组播需要使用组播地址，在 IPv4 中它的范围从 `224.0.0.0` 到 `239.255.255.255`，并被划分为局部链接多播地址、预留多播地址和管理权限多播地址三类:

| IP地址                      | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `224.0.0.0~224.0.0.255`     | 局部链接多播地址：是为路由协议和其它用途保留的地址， 只能用于局域网中，路由器是不会转发的地址 224.0.0.0不能用，是保留地址 |
| `224.0.1.0~224.0.1.255`     | 为用户可用的组播地址（临时组地址），可以用于 Internet 上的。 |
| `224.0.2.0~238.255.255.255` | 用户可用的组播地址（临时组地址），全网范围内有效             |
| `239.0.0.0~239.255.255.255` | 为本地管理组播地址，仅在特定的本地范围内有效                 |

组播地址不属于任何服务器或个人，它有点类似一个微信群号，任何成员（**组播源**）往微信群（**组播IP**）发送消息（**组播数据**），这个群里的成员（**组播接收者**）都会接收到此消息。

## 12.2 设置组播属性

如果使用组播进行数据的传输，不管是消息发送端还是接收端，都需要进行相关的属性设置，设置函数使用的是同一个，即：`setsockopt()`。

### 12.2.1 发送端

发送组播消息的一端需要设置组播属性，具体的设置方式如下：

```c
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```

- 参数:

  - sockfd：用于UDP通信的套接字

  - level：套接字级别，设置组播属性需要将该参数指定为：`IPPTOTO_IP`

  - optname: 套接字选项名，设置组播属性需要将该参数指定为：`IP_MULTICAST_IF`

  - optval：设置组播属性，这个指针需要指向一个`struct in_addr{}` 类型的结构体地址，这个结构体地址用于存储组播地址，并且组播IP地址的存储方式是大端的。

    ```c
    struct in_addr
    {
        in_addr_t s_addr;	// unsigned int
    }; 
    ```

  - optlen：optval指针指向的内存大小，即：`sizeof(struct in_addr)`

- 返回值：函数调用成功返回0，调用失败返回-1

### 12.2.2 接收端

因为一个组播地址表示一个群组，所以需要接收组播报文的接收者都加入这个群组，和想要接收群消息就必须要先入群是一个道理。加入到这个组播群组的方式如下：

```c
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```

- 参数:

  - sockfd：基于udp的通信的套接字

  - level：套接字级别，加入到多播组该参数需要指定为：`IPPTOTO_IP`

  - optname：套接字选项名，加入到多播组该参数需要指定为：`IP_ADD_MEMBERSHIP`

  - optval：加入到多播组，这个指针应该指向一个`struct ip_mreqn{}`类型的结构体地址

    ```c
    typedef unsigned int  uint32_t;
    typedef uint32_t in_addr_t;
    struct sockaddr_in addr;
    
    struct in_addr
    {
        in_addr_t s_addr;	// unsigned int
    };
    
    struct ip_mreqn
    {
        struct in_addr imr_multiaddr;   // 组播地址/多播地址
        struct in_addr imr_address;     // 本地地址
        int   imr_ifindex;              // 网卡的编号, 每个网卡都有一个编号
    };
    // 必须通过网卡名字才能得到网卡的编号: 可以通过 ifconfig 命令查看网卡名字
    #include <net/if.h>
    // 将网卡名转换为网卡的编号, 参数是网卡的名字, 比如: "ens33"
    // 返回值就是网卡的编号
    unsigned int if_nametoindex(const char *ifname);
    ```

  - optlen：optval指向的内存大小，即：`sizeof(struct ip_mreqn)`

## 12.3 组播通信流程

发送组播消息的一端需要将数据发送到组播地址和固定的端口上，想要接收组播消息的终端需要绑定对应的固定端口然后加入到组播的群组，最终就可以实现数据的共享。

[[![img](Linux%E6%95%99%E7%A8%8B4%E2%80%94%E2%80%94%E5%A5%97%E6%8E%A5%E5%AD%97%E9%80%9A%E4%BF%A1.assets/image-20210407110753006.png)](https://subingwen.cn/linux/multicast/image-20210407110753006.png)
](https://subingwen.cn/linux/multicast/image-20210407110753006.png)

### 12.3.1 发送端

1. `创建通信的套接字`

   ```c
   // 第二个参数是 SOCK_DGRAM, 第三个参数0表示使用报式协议中的udp
   int fd = socket(AF_INET, SOCK_DGRAM, 0);
   ```

2. `主动发送数据的一端不需要手动绑定端口（自动随机分配就可以了），设置UDP组播属性`

   ```c
   // 设置组播属性
   struct in_addr opt;
   // 将组播地址初始化到这个结构体成员中
   inet_pton(AF_INET, "239.0.1.10", &opt.s_addr);
   setsockopt(fd, IPPROTO_IP, IP_MULTICAST_IF, &opt, sizeof(opt));
   ```

3. `使用组播地址发送组播消息到固定的端口（接收端需要绑定这个端口）`

   ```c
   sendto();
   ```

4. `关闭套接字（文件描述符）`

   ```c
   close(fd);
   ```

### 12.3.2 接收端

1. `创建通信的套接字`

   ```c
   // 第二个参数是 SOCK_DGRAM, 第三个参数0表示使用报式协议中的udp
   int fd = socket(AF_INET, SOCK_DGRAM, 0);
   ```

2. `绑定固定的端口，发送端应该将数据发送到接收端绑定的端口上`

   ```c
   bind();
   ```

3. `加入到组播的群组中，入群之后就可以接受组播消息了。`

   ```c
   // 加入到多播组
   struct ip_mreqn opt;
   // 要加入到哪个多播组, 通过组播地址来区分
   inet_pton(AF_INET, "239.0.1.10", &opt.imr_multiaddr.s_addr);
   opt.imr_address.s_addr = INADDR_ANY;
   opt.imr_ifindex = if_nametoindex("ens33");
   setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &opt, sizeof(opt));
   ```

4. `接收组播数据`

   ```c
   recvfrom();
   ```

5. `关闭套接字（文件描述符）`

   ```c
   close(fd);
   ```

## 12.4 通信代码

### 12.4.1 发送端

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

int main()
{
    // 1. 创建通信的套接字
    int fd = socket(AF_INET, SOCK_DGRAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 设置组播属性
    struct in_addr opt;
    // 将组播地址初始化到这个结构体成员中即可
    inet_pton(AF_INET, "239.0.1.10", &opt.s_addr);
    setsockopt(fd, IPPROTO_IP, IP_MULTICAST_IF, &opt, sizeof(opt));

    char buf[1024];
    struct sockaddr_in cliaddr;
    int len = sizeof(cliaddr);
    cliaddr.sin_family = AF_INET;
    cliaddr.sin_port = htons(9999); // 接收端需要绑定9999端口
    // 发送组播消息, 需要使用组播地址, 和设置组播属性使用的组播地址一致就可以
    inet_pton(AF_INET, "239.0.1.10", &cliaddr.sin_addr.s_addr);
    // 3. 通信
    int num = 0;
    while(1)
    {
        sprintf(buf, "hello, client...%d\n", num++);
        // 数据广播
        sendto(fd, buf, strlen(buf)+1, 0, (struct sockaddr*)&cliaddr, len);
        printf("发送的组播的数据: %s\n", buf);
        sleep(1);
    }

    close(fd);

    return 0;
}
```

注意事项：在组播数据的发送端，需要先设置组播属性，发送的数据是通过sendto()函数发送到某一个组播地址上，并且在程序中数据发送到了接收端的9999端口，因此接收端程序必须要绑定这个端口才能收到组播消息。

### 12.4.2 接收端

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>
#include <net/if.h>

int main()
{
    // 1. 创建通信的套接字
    int fd = socket(AF_INET, SOCK_DGRAM, 0);
    if(fd == -1)
    {
        perror("socket");
        exit(0);
    }

    // 2. 通信的套接字和本地的IP与端口绑定
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);    // 大端
    addr.sin_addr.s_addr = INADDR_ANY;  // 0.0.0.0
    int ret = bind(fd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("bind");
        exit(0);
    }

    // 3. 加入到多播组
    struct ip_mreqn opt;
    // 要加入到哪个多播组, 通过组播地址来区分
    inet_pton(AF_INET, "239.0.1.10", &opt.imr_multiaddr.s_addr);
    opt.imr_address.s_addr = INADDR_ANY;
    opt.imr_ifindex = if_nametoindex("ens33");
    setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &opt, sizeof(opt));

    char buf[1024];
    // 3. 通信
    while(1)
    {
        // 接收广播消息
        memset(buf, 0, sizeof(buf));
        // 阻塞等待数据达到
        recvfrom(fd, buf, sizeof(buf), 0, NULL, NULL);
        printf("接收到的组播消息: %s\n", buf);
    }

    close(fd);

    return 0;
}
```

**注意事项：作为组播消息的接收端，必须要先绑定一个固定端口（发送端就可以把数据发送到这个固定的端口上了），然后加入到组播的群组中（一个组播地址可以看做是一个群组），这样就可以接收到组播消息了**。

