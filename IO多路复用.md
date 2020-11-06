# IO MULTIPLEXING

## SELECT
时间复杂度O(n)

select的几大缺点：  

1.每次调用select，都需要把FD集合从用户态拷贝到内核态，这个开销在FD很多时会很大  

2.同时每次调用select都需要在内核遍历传递进来的所有FD，这个开销在FD很多时也很大  

3.select支持的文件描述符数量受限，默认是1024  

```c 
posix_types.h
#define __FD_SETSIZE    1024

#include <sys/select.h>
#include <sys/time.h>

int select(int maxfdp1,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout)
返回值：就绪描述符的数目，超时返回0，出错返回-1
```
1.第一个参数maxfdp1指定待测试的描述字个数，它的值是待测试的最大描述字加1。  

2.中间的三个参数readset、writeset和exceptset指定我们要让内核测试读、写和异常条件的描述字。  

3.timeout告知内核等待所指定描述字中的任何一个就绪可花多少时间。  

struct fd_set可以理解为一个集合，这个集合中存放的是文件描述符，可通过以下四个宏进行设置：
```c
void FD_ZERO(fd_set *fdset);           //清空集合

void FD_SET(int fd, fd_set *fdset);   //将一个给定的文件描述符加入集合之中

void FD_CLR(int fd, fd_set *fdset);   //将一个给定的文件描述符从集合中删除

int FD_ISSET(int fd, fd_set *fdset);   // 检查集合中指定的文件描述符是否可以读写 
```

示例：
```c
  char buffer[MAXBUF];
  int fds[5];
  struct sockaddr_in addr;
  struct sockaddr_in client;
  int addrlen, n,i,max=0;;
  int sockfd, commfd;
  fd_set rset;
 
  sockfd = socket(AF_INET, SOCK_STREAM, 0);
  memset(&addr, 0, sizeof (addr));
  addr.sin_family = AF_INET;
  addr.sin_port = htons(2000);
  addr.sin_addr.s_addr = INADDR_ANY;
  bind(sockfd,(struct sockaddr*)&addr ,sizeof(addr));
  listen (sockfd, 5); 
 
  for (i=0;i<5;i++) 
  {
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    fds[i] = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    if(fds[i] > max)
    	max = fds[i];
  }
  
  while(1){
	FD_ZERO(&rset);
  	for (i = 0; i< 5; i++ ) {
  		FD_SET(fds[i],&rset);
  	}
 
   	puts("round again");
	select(max+1, &rset, NULL, NULL, NULL);
 
	for(i=0;i<5;i++) {
		if (FD_ISSET(fds[i], &rset)){
			memset(buffer,0,MAXBUF);
			read(fds[i], buffer, MAXBUF);
			puts(buffer);
		}
	}	
  }
```

## POLL
时间复杂度O(n)
poll本质上和select没有区别，只是描述FD集合的方式不同，poll使用pollfd结构而不是select的fd_set结构，所以没有最大连接数的限制。
poll和select同样存在的缺点是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。
```c
# include <poll.h>
int poll ( struct pollfd * fds, unsigned int nfds, int timeout);

struct pollfd {
      int fd;  // 文件描述符
      short events;  // 等待的事件
      short revents; // 实际发生了的事件
};

合法的事件如下：
POLLIN 　　　　　　　有数据可读。
POLLRDNORM 　　　　 有普通数据可读。
POLLRDBAND　　　　　有优先数据可读。
POLLPRI　　　　　　　有紧迫数据可读。
POLLOUT　　　　　　  写数据不会导致阻塞。
POLLWRNORM　　　　　 写普通数据不会导致阻塞。
POLLWRBAND　　　　　 写优先数据不会导致阻塞。
POLLMSGSIGPOLL 　　消息可用。
此外，revents域中还可能返回下列事件：
POLLER　　      指定的文件描述符发生错误。
POLLHUP　　     指定的文件描述符挂起事件。
POLLNVAL　　    指定的文件描述符非法。
```
返回值和错误代码：  

成功时，poll()返回结构体中revents域不为0的文件描述符个数；如果在超时前没有任何事件发生，poll()返回0；失败时，poll()返回-1，并设置errno为下列值之一：  
EBADF　　       一个或多个结构体中指定的文件描述符无效。  
EFAULTfds　　 指针指向的地址超出进程的地址空间。  
EINTR　　　　  请求的事件之前产生一个信号，调用可以重新发起。  
EINVALnfds　　参数超出PLIMIT_NOFILE值。  
ENOMEM　　     可用内存不足，无法完成请求。  

示例：
```c
 for (i=0;i<5;i++) 
  {
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    pollfds[i].fd = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    pollfds[i].events = POLLIN;
  }
  sleep(1);
  while(1){
  	puts("round again");
	poll(pollfds, 5, 50000);
 
	for(i=0;i<5;i++) {
		if (pollfds[i].revents & POLLIN){
			pollfds[i].revents = 0;
			memset(buffer,0,MAXBUF);
			read(pollfds[i].fd, buffer, MAXBUF);
			puts(buffer);
		}
	}
  }
```

## EPOLL
时间复杂度O(1)
epoll可以理解为event poll，不同于忙轮询和无差别轮询，epoll会把哪个流发生了怎样的I/O事件通知我们。所以我们说epoll实际上是事件驱动（每个事件关联上fd）的，此时我们对这些流的操作都是有意义的。（复杂度降低到了O(1)）

epoll的优点：  

1.没有最大并发连接的限制，能打开的FD的上限远大于1024  

2.效率提升，不是轮询的方式，只有活跃可用的FD才会调用callback函数  

即epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，epoll的效率就会远远高于select和poll。  

3.内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递；即epoll使用mmap减少复制开销  

### 工作模式
epoll有两种工作方式

ET：Edge Triggered，边缘触发。仅当状态发生变化时才会通知，epoll_wait返回。换句话，就是对于一个事件，只通知一次。且只支持非阻塞的socket。所以在ET模式下，read一个FD的时候一定要把它的buffer读光，也就是说一直读到read的返回值小于请求值，或者 遇到EAGAIN错误。  

LT：Level Triggered，电平触发（默认工作方式）。类似select/poll,只要还有没有处理的事件就会一直通知，以LT方式调用epoll接口的时候，它就相当于一个速度比较快的poll.支持阻塞和不阻塞的socket。  

 ### epoll用法
1.epoll_create
```c
int   epoll_create(int size);
```
创建一个epoll的句柄。需要注意的是，当创建好epoll句柄后，它就是会占用一个FD值，在linux下如果查看/proc/进程id/fd/，是能够看到这个FD的，所以在使用完epoll后，必须调用close()关闭，否则可能导致FD被耗尽。  

返回值：成功返回一个epoll句柄，失败返回-1；  

2.epoll_ctl
```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
epoll的事件注册函数,它不同于select()是在监听事件时告诉内核要监听什么类型的事件,而是在这里先注册要监听的事件类型。(一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知。)

返回值：成功返回0，失败返回-1；  

epfd参数：epoll_create创建的一个epoll句柄。

op参数：表示要执行的动作，用三个宏来表示：
EPOLL_CTL_ADD：注册新的fd到epfd中;
EPOLL_CTL_MOD：修改已经注册的fd的监听事件;
EPOLL_CTL_DEL：从epfd中删除一个fd;

fd参数：需要监听的文件描述符。

event参数：告诉内核需要监听什么事。struct epoll_event结构如下:
```c
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
```

events有如下值：

EPOLLIN ：表示对应的文件描述符可以读(包括对端SOCKET正常关闭);

EPOLLOUT：表示对应的文件描述符可以写;
EPOLLPRI：表示对应的文件描述符有紧急的数据可读(这里应该表示示有带外数据到来);
EPOLLERR：表示对应的文件描述符发生错误;
EPOLLHUP：表示对应的文件描述符被挂断;
EPOLLET：将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水水平触发(Level
Triggered)来说的。（epoll默认为水平触发）

EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里。

3.epoll_wait
```c
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```
监听在epoll监控的事件中已经发送的事件。  

返回值：成功返回监听文件描述符集中已经准备好的文件描述符，返回0代表timeout，失败返回-1。  

epoll参数：epoll_create创建的epoll句柄。  

events参数：输出型参数，保存监听文件描述符集中已经准备好的文件描述符集。  

maxevents参数：events数组的大小。  

timeout参数：超时时间。单位为毫秒。  

示例：
```c
  struct epoll_event events[5];
  int epfd = epoll_create(10);
  ...
  ...
  for (i=0;i<5;i++) 
  {
    static struct epoll_event ev;
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    ev.data.fd = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    ev.events = EPOLLIN;
    epoll_ctl(epfd, EPOLL_CTL_ADD, ev.data.fd, &ev); 
  }
  
  while(1){
  	puts("round again");
  	nfds = epoll_wait(epfd, events, 5, 10000);
	
	for(i=0;i<nfds;i++) {
			memset(buffer,0,MAXBUF);
			read(events[i].data.fd, buffer, MAXBUF);
			puts(buffer);
	}
  }
```

##  KQUEUE
kqueue是FreeBSD上的一种的多路复用机制。它是针对传统的select/poll处理大量的文件描述符性能较低效而开发出来的。注册一批描述符到kqueue以后，当其中的描述符状态发生变化时，kqueue将一次性通知应用程序哪些描述符可读、可写或出错。

kqueue的接口包括 kqueue()、kevent() 两个系统调用和 struct kevent 结构：  
kqueue() 生成一个内核事件队列，返回该队列的文件描述符。其它API通过该描述符操作这个kqueue。  
kevent() 提供向内核注册/反注册事件和返回就绪事件或错误事件。  

struct kevent 就是kevent()操作的最基本的事件结构。  
```c
struct kevent { 
     uintptr_t ident;        // 事件 ID
     short     filter;       // 事件过滤器
     u_short   flags;        // 行为标识
     u_int     fflags;       // 过滤器标识值
     intptr_t  data;         // 过滤器数据
     void      *udata;       // 应用透传数据
 };
```
示例：
```c
// 为文件描述符打开对应状态位的工具函数
void turn_on_flags(int fd, int flags){
    int current_flags;
    // 获取给定文件描述符现有的flag
    // 其中fcntl的第二个参数F_GETFL表示要获取fd的状态
    if( (current_flags = fcntl(fd, F_GETFL)) < 0 ) exit(1);

    // 施加新的状态位
    current_flags |= flags;
    if( fcntl(fd, F_SETFL, current_flags) < 0 ) exit(1);
}

const static int FD_NUM = 2; // 两个文件描述符，分别为标准输入与输出
const static int BUFFER_SIZE = 1024; // 缓冲区大小

// 完全以IO复用的方式读入标准输入流数据，输出到标准输出流中
int main(){
    struct kevent changes[FD_NUM]; // 要监视的事件列表
    struct kevent events[FD_NUM];  // kevent返回的事件列表（参考后面的kevent函数）

    // 创建一个kqueue
    int kq;
    if( (kq = kqueue()) == -1 ) quit("kqueue()");

    // 准备从标准输入流中读数据
    int stdin_fd = STDIN_FILENO;
    int stdout_fd = STDOUT_FILENO;

    // 设置为非阻塞
    turn_on_flags(stdin_fd, O_NONBLOCK);
    turn_on_flags(stdout_fd, O_NONBLOCK);

    // 注册监听事件

    // 在changes列表中注册标准输入流的读事件 以及 标准输出流的写事件
    // 最后一个参数可以是任意的附加数据（void * 类型），在这里给事件附上了当前的文件描述符，后面会用到
    int k = 0;
    EV_SET(&changes[k++], stdin_fd, EVFILT_READ, EV_ADD | EV_ENABLE, 0, 0, &stdin_fd);
    EV_SET(&changes[k++], stdout_fd, EVFILT_WRITE, EV_ADD | EV_ENABLE, 0, 0, &stdout_fd);

    int nev, nread, nwrote = 0; // 发生事件的数量, 已读字节数, 已写字节数
    char buffer[BUFFER_SIZE];

    while(1){
        // 进行kevent函数调用，如果changes列表里有任何就绪的fd，则把该事件对应的结构体放进events列表里面
        // 返回值是这次调用得到了几个就绪的事件 (nev = number of events)
        nev = kevent(kq, changes, FD_NUM, events, FD_NUM, NULL); // 已经就绪的文件描述符数量
        if( nev <= 0 ) quit("kevent()");

        int i;
        for(i=0; i<nev; i++){
            struct kevent event = events[i]; // 一个个取出已经就绪的事件
            if( event.flags & EV_ERROR ) quit("Event error");  // 退出程序

            int ev_fd = *((int *)event.udata); // 从附加数据里面取回文件描述符的值

            // 输入流就绪 且 缓冲区还有空间能继续读
            if( ev_fd == stdin_fd && nread < BUFFER_SIZE ){
                int new_nread;
                if( (new_nread = read(ev_fd, buffer + nread, sizeof(buffer) - nread)) <= 0 )
                    quit("read()"); // 由于可读事件已经发生，因此如果读出0个字节也是不正常的
                
                nread += new_nread; // 递增已读数据字节数
            }

            // 输出流就绪 且 缓冲区有内容可以写出
            if( ev_fd == stdout_fd && nread > 0 ){
                if( (nwrote = write(stdout_fd, buffer, nread)) <=0 )
                    quit("write()");

                memmove(buffer, buffer+nwrote, nwrote); // 为了使实现的代码更简洁，这里把还没有写出去的数据往前移动
                nread -= nwrote; // 减去已经写出去的字节数
            }
        }
    }

    return 0;
}
```