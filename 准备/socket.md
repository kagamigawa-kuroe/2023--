#### socket

简单的socket工作的基本流程

客户端 : 

1. 用socket函数创建一个套接字，如: **sockfd = socket(AF_INIT,SOCK_STREAM,0)**

   得到一个文件描述符

2. 用想要进行连接的服务器的地址和端口，创建一个**sockaddr/sockaddr_in**变量

3. 调用connect函数，建立连接，如：**connect(sockfd, (struct sockaddr*)&address,sizeof(address))**

4. 在循环中一直read, 获取内容，如: **read(sockfd,recvline,Maxline)**

服务端 :

1. 用socket函数创建一个套接字，如: **sockfd = socket(AF_INIT,SOCK_STREAM,0)**

   得到一个文件描述符 这一步和服务端一样

2. 也要创建一个**sockaddr/sockaddr_in** 只不过地址用**INDDR_ANY**表示所有

3. 调用bind函数, 绑定上述两个变量，如：**bind(sockfd, (struct sockaddr*)&address,sizeof(address))** 

4. 调用listen函数，表示监听这个描述符，如：**Listen(sockfd,LISTENQ)**

5. 调用accept函数，从这个描述符中创建连接，即创建一个新的描述符用于通讯，如：**connfd=Accept(sockfd,NULL,NULL)**

   注意 后续在write read的时候都会用到这个connfd描述符，而之前的sockfd，只作为一个监听并且建立连接的描述符存在。

---

首先是IPv4的基本结构，即sockaddr_in的基本结构:

```c
#include <netinet/in.h>

struct sockaddr_in {
    uint8_t          sin_length;   // 长度 在某些版本中不存在
    short            sin_family;   // e.g. AF_INET
    unsigned short   sin_port;     // e.g. htons(3490)
    struct in_addr   sin_addr;     // see struct in_addr, below
    char             sin_zero[8];  // zero this if you want to
};

struct in_addr {
    unsigned long s_addr;  // load with inet_aton()
};
```

然后是IPv6的基本结构：

```c
struct sockaddr_in6 {
    uint16_t        sin6_family;   /* AF_INET6 */
    uint16_t        sin6_port;     /* numéro de port */
    uint32_t        sin6_flowinfo; /* information de flux IPv6 IPv6流标识符*/
    struct in6_addr sin6_addr;     /* adresse IPv6 */
    uint32_t        sin6_scope_id; /* Scope ID (nouveauté 2.4) sin6_scope_id是一个ID，具体取决于地址范围*/
};

struct in6_addr {
    unsigned char   s6_addr[16];   /* adresse IPv6 */
};
```

通用套接字地址sockddr

```c
struct sockaddr {
	u_short sa_family;
	char	sa_data[14];
};
```

当我们使用sockaddr_in时，一般都会传入他的地址到socket中，一般我们会将其转换成一个sockaddr类型的指针.

---

#### socket函数 

创建套接字描述符 三个参数 

- 第一个为协议簇 常见的有AF_INIT ipv4, AF_INIT6 ipv6等
- 第二个为采用的套接字类型 常见的有SOCK_STREAM 字节流 SOCK_DGRAM 数据报
- 第三个参数可以直接指定某一种协议 而不采用前两个参数的组合 比如直接指定TCP协议 或者UDP协议

#### connect函数

用于与服务器建立一个连接

connect有三个参数：

- 第一个是上一步创建的描述符
- 第二个是一个指向套接字地址结构的指针 （struct sockaddr*）
- 第三个是该结构的大小

connect就是三次握手建立连接的过程

成功后返回一个文件描述符 可以对其进行读写 就可以进行通讯

connect可能会返回的几类错误：

- 若再发送SYN帧后无响应 则会隔一段时间再发送 多次之后 就返回ETIMEDOUT错误
- 如果收到了返回值 但是是一个RST 复位帧 就表示服务器上并没有监听 返回ECONNREFUSED错误
- 还有一种可能 在数据流传输过程中 路由不可达 返回了错误

#### bind函数

bind函数是在服务器端使用的 将一个地址(自己的地址)绑定给一个套接字。

- 第一个参数是一个socket描述符
- 第二个是一个指向套接字地址结构的指针 （struct sockaddr*）
- 第三个是该结构的大小

可以看出 和connect很类似

bind如果发现地址已经被使用 就会返回EADDRINUSED错误

#### listen函数

当我们用用socket函数创建一个套接字的时候 他会是一个主动套接字 也就是说是给客户端使用的套接字

listen函数 可以将其变为一个被动监听的套接字

- 第一个参数为套接字的描述符
- 第二个为最大连接个数

内核会给任意一个监听套接字维护两个连接队列 全连接队列和半连接队列

服务器收到连接请求 并且给予回应 但是还没收到客户端回应的连接 会被放在半连接队列

而三次握手完成的 则会被转移到全连接队列中

两队列之和不超过listen第二个参数

#### accept函数

通过该函数 拿到已完成队列中头部的连接 返回一个**已连接套接字** 我们直接用listen创建的是一个**监听套接字**

- 第一个参数为监听套接字
- 第二个是一个指向套接字地址结构的指针 （struct sockaddr*）可以记录用户的ip
- 第三个是用户ip机构体的大小

这里的概念可能会有点绕 服务器首先用listen函数 将一个普通的套接字 变成监听套接字 然后 客户端一旦和服务器建立完全连接 就会将这个连接放入全连接队列中 

accept一旦发现队列不为空 就会读取队列头部的信息 并用这些信息返回一个已连接套接字，可以供服务端读写。

---

#### 五种I/O模型

1. 阻塞式IO 也就是上述我们提到的 一旦连接队列中没有消息 accept就会阻塞而不进行 本进程进入休眠
2. 非阻塞式IO 一旦accept得不到消息 就会返回一个错误 然后需要我们不断重新尝试
3. 复用IO 将所有的I/O连接全部注册在内核中 然后系统调用不断遍历 探测有无可用的IO
4. 信号驱动复用IO 比起普通复用通过系统调用不断遍历寻找 信号驱动式会在有IO可用时主动给外部进程发送信号 告知其可用
5. 以上四种都是同步IO 还有一种异步IO 可以理解为 当我们执行某一条命令时 不用等待他做完给出返回结果 就可以继续进行程序 比如当我们进行read时候，不用等确实读到了东西，而是继续执行代码 等到read这个命令执行完了 就会给予我们提醒。

---

#### select复用模型

select函数可以在一定时间内检测一系列文件描述符是否准备就绪，并将就绪的返回。

select函数有几个参数：

- 第一个参数为代测试的最大描述符+1
- 第二个为 读描述符的集合
- 第三个为 写描述符的集合
- 第四个为 异常描述符的集合
- 第五个是 最大等待时间

如果到了最大等待时间 还没有任何描述符准备好 则会返回0

若有准备好的 则会将传入集合的引用更改，集合中剩下的就是准备好的

如何定义一个描述符集合：

``` c
fd_set set;
FD_ZERO(&set);  // 清空
FD_SET(1,&set); // 加入1
FD_CLR(1,&set); // 清除1
```

---

#### poll函数复用模型

```
int poll(struct pollfd fd[], nfds_t nfds, int timeout);
```

第一个参数为描述一个fd的结构体 如下

```C
struct pollfd{
　int fd； // 文件描述符
　short event；// 请求的事件
　short revent；// 返回的事件
}
```

第二个参数为描述符数量

第三个为超时时间



poll和select都属于普通的轮询多路复用模型

---

#### UDP套接字使用

对于客户端 udp可以不用connect函数 而直接使用sendto函数向客户端发送消息 用recvfrom函数进行读取消息。

对于服务端 在用bind绑定后 我们不需要用listen函数去监听 也是直接用sendto和recvfrom进行监听即可

---

#### epoll的使用

```c
// 创建一个epoll描述符
int epoll_create(int size);

// 对特定文件fd 进行op操作(登记/删除) event为登记的事件
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};

int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);
```

边缘触发和水平触发

水平触发(level-trggered)

- 只要文件描述符关联的读内核缓冲区非空，有数据可以读取，就一直发出可读信号进行通知，
- 当文件描述符关联的内核写缓冲区不满，有空间可以写入，就一直发出可写信号进行通知

边缘触发(edge-triggered)

- 当文件描述符关联的读内核缓冲区由空转化为非空的时候，则发出可读信号进行通知，
- 当文件描述符关联的内核写缓冲区由满转化为不满的时候，则发出可写信号进行通知

基本用法是，服务端将listenfd 监听套接字登记到epoll中 一旦有listenfd有事件发生 就accept一个新的连接 加入到epoll中

如果是普通连接有事件发生 则根据读写事件采取不同的处理

---

#### 异步IO

