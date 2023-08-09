---
layout: default
---

## IO多路复用

### 一、什么是IO多路复用

IO多路复用即使用一个进程或线程监听多个socket连接。

I/O多路复用的优势在于，当处理的消耗对比IO几乎可以忽略不计时，可以处理大量的并发IO，而不用消耗太多CPU/内存。这就像是一个工作很高效的人，手上一个todo list，他高效的依次处理每个任务。这比每个任务单独安排一个人要节省。

典型的例子是nginx做代理，代理的转发逻辑相对比较简单直接，那么IO多路复用很适合。相反，如果是一个做复杂计算的场景，计算本身可能是个 指数复杂度的东西，IO不是瓶颈。那么怎么充分利用CPU或者显卡的核心多干活才是关键。

### 二、如何实现IO多路复用

1. 使用一个容器管理这些socket连接(即文件描述符)。有新连接进入时就向容器中添加对应的元素，连接中断或超时就删除对应元素。
2. 使用一个进程/线程去遍历这个容器，当其中有一个或多个连接读或/写事件准备就绪时，就进行对应处理。

### 三、需要解决那些问题

1. 尽可能的避免用户态与内核态之间的切换，用户态与内核态之间的切换代价很大。
2. 尽可能的避免无效文件描述符在内核空间和用户空间来回复制。
3. 尽可能的减少遍历次数。

### 四、Linux关于IO多路复用的三种实现

Linux有三个关于IO多路复用的系统调用：select、poll、epoll

#### (1) Select

```c
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

readfds、writefds、exceptfds是三类文件描述符集合，其中元素值为对应文件描述符的索引值/序号。n是文件描述符的最大序号，主要用来减少遍历。timeout是等待时间。

select通过一个bitmap来标识文件描述符的状态：0表示未就绪，1表示已就绪。

我们调用此方法时，传入一个文件描述符集合，以及其它相应参数，系统将该文件描述符集合从用户空间复制到内核空间，对其进行遍历，若其中有1个或多个文件描述符就绪，则修改索引在位图中对应的值，并将就绪个数返回，若没有文件描述符就绪，则阻塞。

这个过程中存在以下几个问题：

1. 每次都需要将文件描述符集合中所有元素从用户态复制到内核态，然而其中活跃的只占少数。
2. bitmap的大小是有限制的，这意味着我们可以管理的socket连接数目将受限于此。
3. select返回后，我们还需要重新遍历一次fds，根据bitmap找出就绪的文件描述符集合。
4. bitmap元素值被改变后没有恢复，因此不可复用。

#### (2) Poll

```c
struct pollfd {
    int fd; 		/* file descriptor 文件描述符索引值*/
    short events;   /* requested events to watch 要监视的event*/
    short revents;  /* returned events witnessed 发生的event*/
};

int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

poll的实现思路与select基本相同，不同之处在于poll使用了一个pollfd数组(可扩容)来对文件描述符集合进行统一管理，而select使用一个数组来存储文件描述符索引，又使用了一个bitmap来存储文件描述符的就绪状态。此外，在poll返回后重新遍历fds时会将revents恢复。

综上，poll相较于select解决了问题2和4。

#### (3)epoll

```c
struct eventpoll{
	...
    // 红黑树的根节点，这棵树中存储着所有添加到epoll中的需要监控的事件
    struct rb_root rbr;
    // 双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件
    struct list_head rdlist;
    ...
}

/*
在epoll中，对于每一个事件，都会建立一个epitem结构体
当调用epoll_wait检查是否有事件发生时，
只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可。
如果rdlist不为空，则把发生的事件复制到用户态，同时将事件数量返回给用户 。
*/
struct epitem{
    struct rb_node rbn;       //红黑树节点
    struct list_head rdllink; //双向链表节点
    struct wpoll_filefd ffd;  //事件句柄信息
    struct evntpoll *ep;	  //指向其所属的eventpoll对象
    struct epoll_event event; //期待发生的事件类型   
}

//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_create(int size)；
    
//用于向内核注册新的描述符或者是改变某个文件描述符的状态.已注册的描述符在内核中会被维护在一棵红黑树上
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
    
//通过回调函数内核会将 I/O 准备好的描述符添加到rdlist双链表管理
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```









