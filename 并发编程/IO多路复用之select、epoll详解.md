## 目录

- IO多路复用
- 为什么出现IO多路复用机制?
- select函数定义
- select流程
- select应用
- epoll函数定义
- epoll流程
- epoll应用
- 完整代码示例

### 一、IO多路复用

**定义**

- IO多路复用是一种同步IO模型，实现一个线程可以监视多个文件句柄，一旦某个文件句柄就绪，就能够通知应用程序进行相应的读写操作。**本质是把accept与recv两个阻塞操作分开**

**优点**

- IO多路复用相比于单线程阻塞IO模型的优势，可以同时accept更多请求
- IO多路复用相比于多线程的优势在于系统的开销小，系统不必创建和维护进程或线程，免去了线程或进程的切换带来的开销。

**应用**

- 操作系统支持IO多路复用的系统调用有select，poll和epoll，比如nginx中采用epoll实现高并发

### 二、为什么有IO多路复用机制?

**之前的做法**

- 服务器端采用单线程，当accept一个请求后，在recv或send调用阻塞时，将无法accept其他请求（必须等上一个请求处理完）
- 服务器端采用多线程，当accept一个请求后，开启线程进行recv，但大量的线程占用很大的内存空间，并且线程切换会带来很大的开销

**之后的做法**

- 服务器端采用单线程通过select/epoll等系统调用获取fd列表，遍历有事件的fd进行accept/recv/send，使其能支持更多的并发连接请求

### 三、select函数定义

```
​#include <sys/select.h>
#include <sys/time.h>
​
int select (int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
/* Returns: positive count of ready descriptors, 0 on timeout, –1 on error */
```

### 四、select流程

![select流程图](http://assets.processon.com/chart_image/5e3024ade4b006a43ae6abe5.png)

### 五、select应用（c语言版）

```
#include <sys/types.h>
#include <sys/socket.h>
#include <stdio.h>
#include <netinet/in.h>
#include <sys/time.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
    int server_sockfd, client_sockfd;
    int server_len, client_len;
    struct sockaddr_in server_address;
    struct sockaddr_in client_address;
    int result;
    fd_set readfds, testfds;
    server_sockfd = socket(AF_INET, SOCK_STREAM, 0);//建立服务器端socket
    server_address.sin_family = AF_INET;
    server_address.sin_addr.s_addr = htonl(INADDR_ANY);
    server_address.sin_port = htons(8000);
    server_len = sizeof(server_address);
    bind(server_sockfd, (struct sockaddr *)&server_address, server_len);
    listen(server_sockfd, 5);
    FD_ZERO(&readfds);
    FD_SET(server_sockfd, &readfds);//将服务器端socket加入到集合中
    while(1)
    {
        int fd;
        int nread;
        testfds = readfds;
        printf("server waiting\n");

        /*无限期阻塞，并测试文件描述符变动 */
        // 1024个文件描述符
        result = select(FD_SETSIZE, &testfds, (fd_set *)0,(fd_set *)0, (struct timeval *) 0);
        if(result < 1)
        {
            printf("select error");
        }

        /*扫描所有的文件描述符*/
        for(fd = 0; fd < FD_SETSIZE; fd++)
        {
            /*找到相关文件描述符*/
            if(FD_ISSET(fd,&testfds))
            {
                /*判断是否为服务器套接字，是则表示为客户请求连接。*/
                if(fd == server_sockfd)
                {
                    client_len = sizeof(client_address);
                    client_sockfd = accept(server_sockfd, (struct sockaddr *)&client_address, &client_len);
                    FD_SET(client_sockfd, &readfds);//将客户端socket加入到集合中
                    printf("connected client:%d\n", client_sockfd);
                }
                /*客户端socket中有数据请求时*/
                else
                {
                    ioctl(fd, FIONREAD, &nread);//取得数据量交给nread

                    /*客户数据请求完毕，关闭套接字，从集合中清除相应描述符 */
                    if(nread == 0)
                    {
                        close(fd);
                        FD_CLR(fd, &readfds);
                        printf("removing client on fd %d/n", fd);
                    }
                    /*处理客户数据请求*/
                    else
                    {
                        char buff[4096];
                        read(fd, buff, 4096);
                        printf("recv msg from client connectfd: %d, content:%s\n",fd, &buff);
                        write(fd, buff, sizeof(buff))   ;
                    }
                }
            }
        }
    }
}
```

### 六、epoll函数定义

```
#include <sys/epoll.h>

int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

### 七、epoll流程

![epoll流程图](http://assets.processon.com/chart_image/5e2fe670e4b0781d52b03d7e.png)

### 八、epoll应用（c语言版）

```
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <sys/types.h>

#define    MAXLINE        4096
#define    LISTENQ        20
#define    SERV_PORT    8000

int main(int argc, char* argv[])
{
    int i, maxi, listenfd, connfd, sockfd, epfd,nfds;
    ssize_t n;
    char BUF[MAXLINE];
    socklen_t clilen;

    //ev用于注册事件,数组用于回传要处理的事件

    struct epoll_event ev,events[20];
    //生成用于处理accept的epoll专用的文件描述符

    epfd=epoll_create(256);
    struct sockaddr_in cliaddr, servaddr;
    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    //setnonblocking(listenfd);
    //设置与要处理的事件相关的文件描述符

    ev.data.fd=listenfd;
    ev.events=EPOLLIN|EPOLLET;
    //注册epoll事件

    epoll_ctl(epfd,EPOLL_CTL_ADD,listenfd,&ev);

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl (INADDR_ANY);
    servaddr.sin_port = htons (SERV_PORT);
    bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));
    listen(listenfd, LISTENQ);
    maxi = 0; //useless
    for ( ; ; ) {
        nfds=epoll_wait(epfd,events,20,0);

        for(i=0;i<nfds;++i)
        {
            if(events[i].data.fd==listenfd)//如果新监测到一个SOCKET用户连接到了绑定的SOCKET端口，建立新的连接。
            {
                connfd = accept(listenfd,(struct sockaddr *)&cliaddr, &clilen);
                if(connfd<0){
                    perror("connfd<0");
                    exit(0);
                }
                char *str = inet_ntoa(cliaddr.sin_addr);
                printf("connected client:%s\n", connfd);

                ev.data.fd=connfd;
                ev.events=EPOLLIN|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev);
            }
            else if (events[i].events&EPOLLIN) //如果是已经连接的用户，并且收到数据，那么进行读入。

            {
                if ( (sockfd = events[i].data.fd) < 0)
                    continue;
                if ( (n = read(sockfd, BUF, MAXLINE)) < 0) {
                    if (errno == ECONNRESET) {
                        close(sockfd);
                        events[i].data.fd = -1;
                    } else
                        printf("readline error\n");
                } else if (n == 0) {
                    close(sockfd);
                    events[i].data.fd = -1;
                }
                BUF[n] = '\0';
                printf("recv msg from client connectfd: %d, content:%s\n",sockfd, BUF);

                ev.data.fd=sockfd;
                ev.events=EPOLLOUT|EPOLLET;
                //读完后准备写
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);

            }
            else if(events[i].events&EPOLLOUT) // 如果有数据发送

            {
                sockfd = events[i].data.fd;
                write(sockfd, BUF, n);

                ev.data.fd=sockfd;
                ev.events=EPOLLIN|EPOLLET;
                //写完后，这个sockfd准备读
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);
            }
        }
    }
    return 0;
}
```

### 九、完整代码示例

https://github.com/caijinlin/learning-pratice/tree/master/linux/io
