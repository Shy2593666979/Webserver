## epoll详解

在linux的网络编程中，很长的时间都在使用select来做事件触发。在linux新的内核中，有了一种替换它的机制，就是epoll。
相比于select，epoll最大的好处在于它不会随着监听fd数目的增长而降低效率。因为在内核中的select实现中，它是采用轮询来处理的，轮询的fd数目越多，自然耗时越多。并且，在linux/posix_types.h头文件有这样的声明：
\#define __FD_SETSIZE  1024
表示select最多同时监听1024个fd，当然，可以通过修改头文件再重编译内核来扩大这个数目，但这似乎并不治本。

epoll的接口非常简单，一共就三个函数：

### 1、int epoll_create(int size);

创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

注意：size参数只是告诉内核这个 epoll对象会处理的事件大致数目，而不是能够处理的事件的最大个数。在 Linux最新的一些内核版本的实现中，这个 size参数没有任何意义。



### 2、int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); 

epoll的事件注册函数，epoll_ctl向 epoll对象中添加、修改或者删除感兴趣的事件，返回0表示成功，否则返回–1，此时需要根据errno错误码判断错误类型。

它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。

epoll_wait方法返回的事件必然是通过 epoll_ctl添加到 epoll中的。

第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：
EPOLL_CTL_ADD：注册新的fd到epfd中；
EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
EPOLL_CTL_DEL：从epfd中删除一个fd；
第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：



```c++
 1 typedef union epoll_data {
 2     void *ptr;
 3     int fd;
 4     __uint32_t u32;
 5     __uint64_t u64;
 6 } epoll_data_t;
 7 
 8 struct epoll_event {
 9     __uint32_t events; /* Epoll events */
10     epoll_data_t data; /* User data variabl */
11 };
```

events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里



### 3、int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

​		等待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个 maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。如果返回–1，则表示出现错误，需要检查 errno错误码判断错误类型。

第1个参数 epfd是 epoll的描述符。

第2个参数 events则是分配好的 epoll_event结构体数组，epoll将会把发生的事件复制到 events数组中（events不可以是空指针，内核只负责把数据复制到这个 events数组中，不会去帮助我们在用户态中分配内存。内核这种做法效率很高）。

第3个参数 maxevents表示本次可以返回的最大事件数目，通常 maxevents参数与预分配的events数组的大小是相等的。

第4个参数 timeout表示在没有检测到事件发生时最多等待的时间（单位为毫秒），如果 timeout为0，则表示 epoll_wait在 rdllist链表中为空，立刻返回，不会等待。



### 4、关于ET、LT两种工作模式

epoll有两种工作模式：LT（水平触发）模式和ET（边缘触发）模式。

默认情况下，epoll采用 LT模式工作，这时可以处理阻塞和非阻塞套接字，而上表中的 EPOLLET表示可以将一个事件改为 ET模式。ET模式的效率要比 LT模式高，它只支持非阻塞套接字。

**ET模式与LT模式的区别在于：**

当一个新的事件到来时，ET模式下当然可以从 epoll_wait调用中获取到这个事件，可是如果这次没有把这个事件对应的套接字缓冲区处理完，在这个套接字没有新的事件再次到来时，在 ET模式下是无法再次从 epoll_wait调用中获取这个事件的；而 LT模式则相反，只要一个事件对应的套接字缓冲区还有数据，就总能从 epoll_wait中获取这个事件。因此，在 LT模式下开发基于 epoll的应用要简单一些，不太容易出错，而在 ET模式下事件发生时，如果没有彻底地将缓冲区数据处理完，则会导致缓冲区中的用户请求得不到响应。默认情况下，Nginx是通过 ET模式使用 epoll的。



### 5、结论

ET模式仅当状态发生变化的时候才获得通知,这里所谓的状态的变化并不包括缓冲区中还有未处理的数据,也就是说,如果要采用ET模式,需要一直read/write直到出错为止,很多人反映为什么采用ET模式只接收了一部分数据就再也得不到通知了,大多因为这样;而LT模式是只要有数据没有处理就会一直通知下去的.


那么究竟如何来使用epoll呢？其实非常简单。
通过在包含一个头文件#include <sys/epoll.h> 以及几个简单的API将可以大大的提高你的网络服务器的支持人数。

首先通过create_epoll(int maxfds)来创建一个epoll的句柄，其中maxfds为你epoll所支持的最大句柄数。这个函数会返回一个新的epoll句柄，之后的所有操作将通过这个句柄来进行操作。在用完之后，记得用close()来关闭这个创建出来的epoll句柄。

之后在你的网络主循环里面，每一帧的调用epoll_wait(int epfd, epoll_event events, int max events, int timeout)来查询所有的网络接口，看哪一个可以读，哪一个可以写了。基本的语法为：
nfds = epoll_wait(kdpfd, events, maxevents, -1);
其中kdpfd为用epoll_create创建之后的句柄，events是一个epoll_event*的指针，当epoll_wait这个函数操作成功之后，epoll_events里面将储存所有的读写事件。max_events是当前需要监听的所有socket句柄数。最后一个timeout是 epoll_wait的超时，为0的时候表示马上返回，为-1的时候表示一直等下去，直到有事件范围，为任意正整数的时候表示等这么长的时间，如果一直没有事件，则范围。一般如果网络主循环是单独的线程的话，可以用-1来等，这样可以保证一些效率，如果是和主逻辑在同一个线程的话，则可以用0来保证主循环的效率。

epoll_wait范围之后应该是一个循环，遍利所有的事件。

几乎所有的epoll程序都使用下面的框架：



```c++
 1 for( ; ; )
 2     {
 3         nfds = epoll_wait(epfd,events,20,500);
 4         for(i=0;i<nfds;++i)
 5         {
 6             if(events[i].data.fd==listenfd) //有新的连接
 7             {
 8                 connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen); //accept这个连接
 9                 ev.data.fd=connfd;
10                 ev.events=EPOLLIN|EPOLLET;
11                 epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev); //将新的fd添加到epoll的监听队列中
12             }
13             else if( events[i].events&EPOLLIN ) //接收到数据，读socket
14             {
15                 n = read(sockfd, line, MAXLINE)) < 0    //读
16                 ev.data.ptr = md;     //md为自定义类型，添加数据
17                 ev.events=EPOLLOUT|EPOLLET;
18                 epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);//修改标识符，等待下一个循环时发送数据，异步处理的精髓
19             }
20             else if(events[i].events&EPOLLOUT) //有数据待发送，写socket
21             {
22                 struct myepoll_data* md = (myepoll_data*)events[i].data.ptr;    //取数据
23                 sockfd = md->fd;
24                 send( sockfd, md->ptr, strlen((char*)md->ptr), 0 );        //发送数据
25                 ev.data.fd=sockfd;
26                 ev.events=EPOLLIN|EPOLLET;
27                 epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev); //修改标识符，等待下一个循环时接收数据
28             }
29             else
30             {
31                 //其他的处理
32             }
33         }
34     }
```

 

下面给出一个完整的服务器端例子：

```c++
  1 #include <iostream>
  2 #include <sys/socket.h>
  3 #include <sys/epoll.h>
  4 #include <netinet/in.h>
  5 #include <arpa/inet.h>
  6 #include <fcntl.h>
  7 #include <unistd.h>
  8 #include <stdio.h>
  9 #include <errno.h>
 10 
 11 using namespace std;
 12 
 13 #define MAXLINE 5
 14 #define OPEN_MAX 100
 15 #define LISTENQ 20
 16 #define SERV_PORT 5000
 17 #define INFTIM 1000
 18 
 19 void setnonblocking(int sock)
 20 {
 21     int opts;
 22     opts=fcntl(sock,F_GETFL);
 23     if(opts<0)
 24     {
 25         perror("fcntl(sock,GETFL)");
 26         exit(1);
 27     }
 28     opts = opts|O_NONBLOCK;
 29     if(fcntl(sock,F_SETFL,opts)<0)
 30     {
 31         perror("fcntl(sock,SETFL,opts)");
 32         exit(1);
 33     }
 34 }
 35 
 36 int main(int argc, char* argv[])
 37 {
 38     int i, maxi, listenfd, connfd, sockfd,epfd,nfds, portnumber;
 39     ssize_t n;
 40     char line[MAXLINE];
 41     socklen_t clilen;
 42 
 43 
 44     if ( 2 == argc )
 45     {
 46         if( (portnumber = atoi(argv[1])) < 0 )
 47         {
 48             fprintf(stderr,"Usage:%s portnumber/a/n",argv[0]);
 49             return 1;
 50         }
 51     }
 52     else
 53     {
 54         fprintf(stderr,"Usage:%s portnumber/a/n",argv[0]);
 55         return 1;
 56     }
 57 
 58 
 59 
 60     //声明epoll_event结构体的变量,ev用于注册事件,数组用于回传要处理的事件
 61 
 62     struct epoll_event ev,events[20];
 63     //生成用于处理accept的epoll专用的文件描述符
 64 
 65     epfd=epoll_create(256);
 66     struct sockaddr_in clientaddr;
 67     struct sockaddr_in serveraddr;
 68     listenfd = socket(AF_INET, SOCK_STREAM, 0);
 69     //把socket设置为非阻塞方式
 70 
 71     //setnonblocking(listenfd);
 72 
 73     //设置与要处理的事件相关的文件描述符
 74 
 75     ev.data.fd=listenfd;
 76     //设置要处理的事件类型
 77 
 78     ev.events=EPOLLIN|EPOLLET;
 79     //ev.events=EPOLLIN;
 80 
 81     //注册epoll事件
 82 
 83     epoll_ctl(epfd,EPOLL_CTL_ADD,listenfd,&ev);
 84     bzero(&serveraddr, sizeof(serveraddr));
 85     serveraddr.sin_family = AF_INET;
 86     char *local_addr="127.0.0.1";
 87     inet_aton(local_addr,&(serveraddr.sin_addr));//htons(portnumber);
 88 
 89     serveraddr.sin_port=htons(portnumber);
 90     bind(listenfd,(sockaddr *)&serveraddr, sizeof(serveraddr));
 91     listen(listenfd, LISTENQ);
 92     maxi = 0;
 93     for ( ; ; ) {
 94         //等待epoll事件的发生
 95 
 96         nfds=epoll_wait(epfd,events,20,500);
 97         //处理所发生的所有事件
 98 
 99         for(i=0;i<nfds;++i)
100         {
101             if(events[i].data.fd==listenfd)//如果新监测到一个SOCKET用户连接到了绑定的SOCKET端口，建立新的连接。
102 
103             {
104                 connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen);
105                 if(connfd<0){
106                     perror("connfd<0");
107                     exit(1);
108                 }
109                 //setnonblocking(connfd);
110 
111                 char *str = inet_ntoa(clientaddr.sin_addr);
112                 cout << "accapt a connection from " << str << endl;
113                 //设置用于读操作的文件描述符
114 
115                 ev.data.fd=connfd;
116                 //设置用于注测的读操作事件
117 
118                 ev.events=EPOLLIN|EPOLLET;
119                 //ev.events=EPOLLIN;
120 
121                 //注册ev
122 
123                 epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev);
124             }
125             else if(events[i].events&EPOLLIN)//如果是已经连接的用户，并且收到数据，那么进行读入。
126 
127             {
128                 cout << "EPOLLIN" << endl;
129                 if ( (sockfd = events[i].data.fd) < 0)
130                     continue;
131                 if ( (n = read(sockfd, line, MAXLINE)) < 0) {
132                     if (errno == ECONNRESET) {
133                         close(sockfd);
134                         events[i].data.fd = -1;
135                     } else
136                         std::cout<<"readline error"<<std::endl;
137                 } else if (n == 0) {
138                     close(sockfd);
139                     events[i].data.fd = -1;
140                 }
141                 line[n] = '/0';
142                 cout << "read " << line << endl;
143                 //设置用于写操作的文件描述符
144 
145                 ev.data.fd=sockfd;
146                 //设置用于注测的写操作事件
147 
148                 ev.events=EPOLLOUT|EPOLLET;
149                 //修改sockfd上要处理的事件为EPOLLOUT
150 
151                 //epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);
152 
153             }
154             else if(events[i].events&EPOLLOUT) // 如果有数据发送
155 
156             {
157                 sockfd = events[i].data.fd;
158                 write(sockfd, line, n);
159                 //设置用于读操作的文件描述符
160 
161                 ev.data.fd=sockfd;
162                 //设置用于注测的读操作事件
163 
164                 ev.events=EPOLLIN|EPOLLET;
165                 //修改sockfd上要处理的事件为EPOLIN
166 
167                 epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);
168             }
169         }
170     }
171     return 0;
172 }
```