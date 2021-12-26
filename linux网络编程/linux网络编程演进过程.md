# Linux网络编程演进过程

## 一、IO模型：阻塞、非阻塞、同步、异步的概念

阻塞IO

非阻塞IO

同步IO

异步IO




## 二、演进过程
TCP/IP基本socket->IO多路复用->reactor->proactor
linux的IO多路复用：select、poll、epoll





对于listenfd，如果是阻塞IO，则会阻塞到accept()，对于connfd，如果是阻塞IO，则会阻塞到recv();

```C
#include "sys/epoll.h"
#include "sys/socket.h"
#include "sys/types.h"
#include <arpa/inet.h>
#include <assert.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <stdlib.h>
#include <string.h>
#include <strings.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include "stdio.h"
#include "errno.h"

int setnoblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option|O_NONBLOCK;
    fcntl(fd, F_SETFL,new_option);
    return old_option;
}

void recv_data(int connfd)
{
    char buf[1024];
    memset(buf, 0, 1024);
    int ret = recv(connfd,buf,1024-1,0);//若connfd是阻塞IO，则会被阻塞在这里
    if(ret<=0)
    {
        if(EAGAIN == errno || errno==EWOULDBLOCK) //若connfd是非阻塞IO，则errno会返回非阻塞的再次尝试
            printf("not blocking, next time read\n");
        else
        {
            close(connfd);
            printf("client socket has been closed\n");
        }
    }
    else
        printf("recv data: %s\n",buf);
}

int main(int argc,char* argv[])
{
    if(argc<=2)
    {
        printf("usage: excBinName ip port\n");
    }
    u_int8_t ret;
    const char* ip = argv[1];
    uint16_t port = atoi(argv[2]);

    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(PF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);

    int listenfd = socket(PF_INET,SOCK_STREAM,0);
    ret = bind(listenfd,(struct sockaddr*)&address,sizeof(address));
    assert(ret!=1);
    ret = listen(listenfd, 5);
    // setnoblocking(listenfd);  //设置listenfd非阻塞IO
    assert(ret!=1);

    while(1) //但是这样浪费CPU资源
    {
        struct sockaddr_in client;
        socklen_t client_len = sizeof(client);
        int connfd = accept(listenfd, (struct sockaddr*)&client, &client_len);//若listenfd是阻塞IO，则会被阻塞在这里
        if(connfd>0)
        {
            printf("accepted\n");
            // setnoblocking(connfd);  //设置connfd非阻塞IO
            recv_data(connfd);
            break;
        }
        else if(EAGAIN == errno || errno==EWOULDBLOCK) //若listenfd是非阻塞IO，则errno会返回非阻塞的再次尝试
        {
            printf("nothing accepted,not blocking\n");
            continue;
        }
        else
            printf("something wrong\n");
    }
    close(listenfd);
    return 0;
}
```



//参考：《Linux高性能服务器编程》、小林Coding公众号

