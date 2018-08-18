struct msghdr的使用
#include<sys/socket.h>
struct msghdr  { 
    void  * msg_name ;   / *  消息的协议地址  * / 协议地址和套接口信息，在非连接的UDP中，发送者要指定对方地址端口，接受方用于的到数据来源，如果不需要的话可以设置为NULL（在TCP或者连接的UDP中，一般设置为NULL）
    socklen_t msg_namelen ;   / *  地址的长度  * / 
    struct iovec  * msg_iov ;   / *  多io缓冲区的地址  * / 
     int  msg_iovlen ;   / *  缓冲区的个数  * / 
    void  * msg_control ;   / *  辅助数据的地址  * / 
    socklen_t msg_controllen ;   / *  辅助数据的长度  * / 
     int  msg_flags ;   / *  接收消息的标识  * / 
} ;

ssize_t recvmsg ( int  sockfd ,  struct msghdr  * msg ,   int  flags ) ; 
ssize_t sendmsg ( int  sockfd ,  struct msghdr  * msg ,   int  flags ) ;
成功时候返回读写字节数，出错时候返回-1.
 
这2个函数只用于套接口，不能用于普通的I/O读写，参数sockfd则是指明要读写的套接口。
flags用于传入控制信息，一般包括以下几个
MSG_DONTROUTE             send可用
MSG_DONWAIT                 send与recv都可用
MSG_PEEK                        recv可用
MSG_WAITALL                   recv可用
MSG_OOB                         send可用
MSG_EOR                          send recv可用
 
返回信息都记录在struct msghdr * msg中

msg_name是指向一个结构体struct sockaddr的指针
msg.msg_name = { sa_family= AF_INET, sin_port = 0, sin_addr.s_addr = 172.16.48.1 }；
msg.msg_namelen = 16;长度一半设置为16

多缓冲区的发送和接收处理就是一个struct iovec的数组，每个成员的io_base都指向了不同的buffer的地址。io_len是指该buffer中的数据长度。而在struct msghdr中的msg_iovlen是指buffer缓冲区的个数，即iovec数组的长度。 
msg_control字段的也是指向一段内存，msg_controllen是指该内存的总大小长度，通常该内存被用来存储辅助数据，辅助数据可用于一些特殊的处理。msg_control通常指向一个控制消息头部，其结构体如下所示: 

struct cmsghdr  { 
  socklen_t cmsg_len ;   / *  包含该头部的数据长度  * / 
   int  cmsg_level ;   / *  具体的协议标识  * / 
   int  cmsg_type ;   / *  协议中的类型  * / 
} ; 

样例1，在TCP中使用 sendmsg与recvmsg
 
服务器
......
#define MAXSIZE 100


int main(int argc, char ** argu) {
        .......
        struct msghdr msg;//初始化struct msghdr
        msg.msg_name = NULL; //在tcp中，可以设置为NULL
        struct iovec io;//初始化返回数据
        io.iov_base = buf; //只用了一个缓冲区
        io.iov_len = MAXSIZE; //定义返回数据长度
        msg.msg_iov = &io;
        msg.msg_iovlen = 1;//只用了一个缓冲区，所以长度为1


        ...................
        ssize_t recv_size = recvmsg(connfd, &msg, 0);
        char * temp = msg.msg_iov[0].iov_base;//获取得到的数据
        temp[recv_size] = '\0';//为数据末尾添加结束符
        printf("get message:%s", temp);
        ..........................
}
 
客户端
..................
#define MAXSIZE 100
int main(int argc, char ** argv) {
        .................
        struct msghdr msg;//初始化发送信息
        msg.msg_name = NULL;
        struct iovec io;
        io.iov_base = send_buff;
        io.iov_len = sizeof(send_buff);
        msg.msg_iov = &io;
        msg.msg_iovlen = 1;


        if(argc != 2) {
                printf("please input port");
                exit(1);
        }
        ............
        ssize_t size = sendmsg(sockfd, &msg, 0);
        close(sockfd);
        exit(0);
}
 
这里控制信息都设置成0，主要是初始化返回信息struct msghdr结构

未连接的UDP套接口
服务器
#include "/programe/net/head.h"
#include "stdio.h"
#include "stdlib.h"
#include "string.h"
#include "unistd.h"
#include "sys/wait.h"
#include "sys/select.h"
#include "sys/poll.h"

#define MAXSIZE 100
int main(int argc, char ** argv) {
        int sockfd;
        struct sockaddr_in serv_socket;
        struct sockaddr_in  * client_socket = (struct sockaddr_in *) malloc (sizeof(struct sockaddr_in));
        char buf[MAXSIZE + 1];

        sockfd = socket(AF_INET, SOCK_DGRAM, 0);
        bzero(&serv_socket, sizeof(serv_socket));
        serv_socket.sin_family = AF_INET;
        serv_socket.sin_addr.s_addr = htonl(INADDR_ANY);
        serv_socket.sin_port = htons(atoi(argv[1]));
        bind(sockfd, (struct sockaddr *)&serv_socket, sizeof(serv_socket));

        struct msghdr msg;
        msg.msg_name = client_socket;
        //如果想得到对方的地址和端口，一定要把初始化完毕的内存头指针放入msg之中
        msg.msg_namelen = sizeof(struct sockaddr_in);//长度也要指定
        struct iovec io;
        io.iov_base = buf;
        io.iov_len = MAXSIZE;
        msg.msg_iov = &io;
        msg.msg_iovlen = 1;


        ssize_t len = recvmsg(sockfd, &msg, 0);
        client_socket = (struct sockaddr_in *)msg.msg_name;
        char ip[16];
        inet_ntop(AF_INET, &(client_socket->sin_addr), ip, sizeof(ip));
        int port = ntohs(client_socket->sin_port);
        char * temp = msg.msg_iov[0].iov_base;
        temp[len] = '\0';
        printf("get message from %s[%d]: %s\n", ip, port, temp);
        close(sockfd);
}
 
客户端
#include "/programe/net/head.h"
#include "stdio.h"
#include "stdlib.h"
#include "string.h"
#include "sys/select.h"

#define MAXSIZE 100
int main(int argc, char ** argv) {
        int sockfd;
        struct sockaddr_in serv_socket;
        int maxfdpl;
        char send[] = "hello yuna";
        if(argc != 2) {
                printf("please input port");
                exit(1);
        }
        sockfd = socket(AF_INET, SOCK_DGRAM, 0);
        bzero(&serv_socket, sizeof(serv_socket));
        serv_socket.sin_family = AF_INET;
        serv_socket.sin_port = htons(atoi(argv[1]));
        inet_pton(AF_INET, "192.168.1.235", &serv_socket.sin_addr);

        struct msghdr msg;
        msg.msg_name = &serv_socket;
        msg.msg_namelen = sizeof(struct sockaddr_in);
        struct iovec io;
        io.iov_base = send;
        io.iov_len = sizeof(send);
        msg.msg_iov = &io;
        msg.msg_iovlen = 1;

        ssize_t send_size = sendmsg(sockfd, &msg, 0);
        close(sockfd);
        exit(0);
}
iovec结构体定义及使用 
readv(2)与writev(2)函数都使用一个I/O向量的概念。这是由所包含的文件定义的：
#include <sys/uio.h>
头文件定义了struct iovc，其定义如下：
struct iovec {
    ptr_t iov_base; /* Starting address */
    size_t iov_len; /* Length in bytes */
};
struct iovec定义了一个向量元素。通常，这个结构用作一个多元素的数组。对于每一个传输的元素，指针成员iov_base指向一个缓冲区，这个缓冲区是存放的是readv所接收的数据或是writev将要发送的数据。成员iov_len在各种情况下分别确定了接收的最大长度以及实际写入的长度。

int readv(int fd, const struct iovec *vector, int count);
int writev(int fd, const struct iovec *vector, int count);
这些函数需要三个参数：
        要在其上进行读或是写的文件描述符fd
        读或写所用的I/O向量(vector)
        要使用的向量元素个数(count)
这些函数的返回值是readv所读取的字节数或是writev所写入的字节数。如果有错误发生，就会返回-1，而errno存有错误代码。注意，也其他I/O函数类似，可以返回错误码EINTR来表明他被一个信号所中断。

/*
* writev.c
*
* Short writev(2) demo:
*/
#include
int main(int argc,char **argv)
{
    static char part2[] = "THIS IS FROM WRITEV";
    static char part3[]    = "]\n";
    static char part1[] = "[";
    struct iovec iov[3];
    iov[0].iov_base = part1;
    iov[0].iov_len = strlen(part1);
    iov[1].iov_base = part2;
    iov[1].iov_len = strlen(part2);
    iov[2].iov_base = part3;
    iov[2].iov_len = strlen(part3);
    writev(1,iov,3);
    return 0;
}

文章参考：

http://blog.sina.cn/dpool/blog/s/blog_c2b97b1d01016tra.html

http://blog.163.com/lichuan0502@126/blog/static/9933534820111033228285/

http://blog.csdn.net/newnewman80/article/details/8000533


