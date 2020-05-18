title: ftplib源码分析
author: Salamander
tags:
  - c
categories:
  - c
date: 2020-05-13 15:00:00
---
## FTP协议
相比其他协议，如 HTTP 协议，FTP 协议要复杂一些。与一般的 C/S 应用不同点在于一般的C/S 应用程序一般只会建立一个 Socket 连接，这个连接同时处理服务器端和客户端的连接命令和数据传输。而FTP协议中将命令与数据分开传送的方法提高了效率。 


<!-- more -->


本文环境：
* OS：Ubuntu 18.04.4 LTS 还有 Windows 10专业版
* ftplib：V4.0-1
* gcc： 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)


### 命令结构
FTP 每个命令都有 3 到 4 个大写字母组成，命令后面跟参数，用空格分开。每个命令都以 "\r\n"结束（应答也是用"\r\n"结束），例如发送一个`CWD`命令，那要要发送数据就是：`CWD dirname\r\n`。  

常见的FTP命令有：  
![](https://s1.ax1x.com/2020/05/12/YNtW3F.png)



## 源码分析
ftplib在这里[下载](https://nbpfaus.net/~pfau/ftplib/)。


登录了FTP服务器后，肯定需要一个句柄的量，在这个`ftplib`中是`netbuf`：
```C
typedef struct NetBuf netbuf;

struct NetBuf {
    char *cput,*cget;
    int handle;
    int cavail,cleft;
    char *buf;
    int dir;
    netbuf *ctrl;
    netbuf *data;    
    int cmode;
    struct timeval idletime;
    FtpCallback idlecb;
    void *idlearg;
    unsigned long int xfered;
    unsigned long int cbbytes;
    unsigned long int xfered1;
    char response[RESPONSE_BUFSIZ];
};
```
`handle`字段其实就存了tcp握手成功后的`socket`。

### FtpSendCmd函数

首先来看`FtpSendCmd`函数，这个函数顾名思义，就是用来发送FTP命令，你在`FtpPwd`、`FtpNlst`、`FtpDir`、`FtpGet`等函数中都可以看到它：
```C++
static int FtpSendCmd(const char *cmd, char expresp, netbuf *nControl)
{
    char buf[TMP_BUFSIZ];
    if (nControl->dir != FTPLIB_CONTROL)
        return 0;
    if (ftplib_debug > 2)
        fprintf(stderr,"%s\n",cmd);
    if ((strlen(cmd) + 3) > sizeof(buf))
        return 0;
    sprintf(buf,"%s\r\n",cmd);
    if (net_write(nControl->handle, buf, strlen(buf)) <= 0)
    {
        if (ftplib_debug)
            perror("write");
        return 0;
    }
    return readresp(expresp, nControl);
}
```
这个函数的过程：
1. 用`net_write`函数操作socket发送数据
2. `readresp`函数读取服务器响应


#### net_write函数
我们查看一下`net_write`，其实它就是封装了一下`write`函数，因为**TCP通信对于应用程序来说是完全异步**，你调用`write`写入5个字节，返回不一定是5个字节，可能是3个，4个，所以`net_write`多次调用了`write`（在《UNIX网络编程 卷1》中，作者也有类似的封装）。另外，`write`返回成功了也只代表buf中的数据被复制到了kernel中的TCP发送缓冲区，至于数据什么时候被发往网络，什么时候被对方主机接收，什么时候被对方进程读取，系统调用层面不会给予任何保证和通知。
```C
int net_write(int fd, const char *buf, size_t len)
{
    int done = 0;
    while ( len > 0 )
    {
        int c = write( fd, buf, len );
        if ( c == -1 )
        {
            if ( errno != EINTR && errno != EAGAIN )
            return -1;
        }
        else if ( c == 0 )
        {
            return done;
        }
        else
        {
            buf += c;
            done += c;
            len -= c;
        }
    }
    return done;
}
```

#### readresp函数
发送了FTP的命令数据后，就需要用socket接受响应数据了，**切记TCP是流式传输的**，所以你需要自己做应用层的解析。
FTP的消息块的分割符是`\r\n`，看`readline`函数名应该是读取一行数据
```C++
static int readresp(char c, netbuf *nControl)
{
    char match[5];
    if (readline(nControl->response, RESPONSE_BUFSIZ, nControl) == -1)
    {
        if (ftplib_debug)
            perror("Control socket read failed");
        return 0;
    }
    if (ftplib_debug > 1)
        fprintf(stderr,"%s",nControl->response);
    if (nControl->response[3] == '-')
    {
        strncpy(match,nControl->response,3);
        match[3] = ' ';
        match[4] = '\0';
        do
        {
            if (readline(nControl->response, RESPONSE_BUFSIZ, nControl) == -1)
            {
                if (ftplib_debug)
                    perror("Control socket read failed");
                return 0;
            }
            if (ftplib_debug > 1)
                fprintf(stderr,"%s",nControl->response);
        }
        while (strncmp(nControl->response, match, 4));
    }
    if (nControl->response[0] == c)
          return 1;
    return 0;
}
```



#### readline函数
```C++
static int readline(char *buf, int max, netbuf *ctl)
{
    int x, retval = 0;
    char *end,*bp=buf;
    int eof = 0;

    if ((ctl->dir != FTPLIB_CONTROL) && (ctl->dir != FTPLIB_READ))
        return -1;
    if (max == 0)
        return 0;
    do
    {
        if (ctl->cavail > 0)
        {
            x = (max >= ctl->cavail) ? ctl->cavail : max-1;
            end = memccpy(bp, ctl->cget, '\n',x);
            if (end != NULL)
                x = end - bp;
            retval += x;
            bp += x;
            *bp = '\0';
            max -= x;
            ctl->cget += x;
            ctl->cavail -= x;
            if (end != NULL)
            {
                bp -= 2;
                if (strcmp(bp,"\r\n") == 0)
                {
                    *bp++ = '\n';
                    *bp++ = '\0';
                    --retval;
                }
                break;
            }
        }
        if (max == 1)
        {
            *buf = '\0';
            break;
        }
        if (ctl->cput == ctl->cget)
        {
            ctl->cput = ctl->cget = ctl->buf;
            ctl->cavail = 0;
            ctl->cleft = FTPLIB_BUFSIZ;
        }
        if (eof)
        {
            if (retval == 0)
                retval = -1;
            break;
        }
        if (!socket_wait(ctl))
            return retval;
        if ((x = net_read(ctl->handle, ctl->cput, ctl->cleft)) == -1)
        {
            if (ftplib_debug)
                perror("read");
            retval = -1;
            break;
        }
        if (x == 0)
            eof = 1;
        ctl->cleft -= x;
        ctl->cavail += x;
        ctl->cput += x;
    }
    while (1);
    return retval;
}
```

#### socket_wait()函数
这个函数用了`select`系统函数，用来检测`ctl->handle`是否可读（这里可读的时候就是服务端发过来响应数据了）。select函数的返回值：返回-1表示调用select函数时有错误发生，具体的错误在Linux可通过errno输出来查看；返回0，表示select函数超时；返回正数即调用select函数成功，表示集合中文件描述符的数量，集合也会被修改以显示哪一个文件描述符已准备就绪。  
不过在用来发送命令的`socket`上（也就是调用`FtpConnect`函数得到的那个`socket`），因为`ctl->dir`是`FTPLIB_CONTROL`（`ctl->idlecb`也是NULL），所以直接返回了1。

```C++
/*
 * socket_wait - wait for socket to receive or flush data
 *
 * return 1 if no user callback, otherwise, return value returned by
 * user callback
 */
static int socket_wait(netbuf *ctl)
{
    fd_set fd,*rfd = NULL,*wfd = NULL;
    struct timeval tv;
    int rv = 0;
    if ((ctl->dir == FTPLIB_CONTROL) || (ctl->idlecb == NULL))
        return 1;
    if (ctl->dir == FTPLIB_WRITE)
        wfd = &fd;
    else
        rfd = &fd;
    FD_ZERO(&fd);
    do
    {
        FD_SET(ctl->handle,&fd);
        tv = ctl->idletime;
        rv = select(ctl->handle+1, rfd, wfd, NULL, &tv);
        if (rv == -1)
        {
            rv = 0;
            strncpy(ctl->ctrl->response, strerror(errno),
                        sizeof(ctl->ctrl->response));
            break;
        }
        else if (rv > 0)
        {
            rv = 1;
            break;
        }
    }
    while ((rv = ctl->idlecb(ctl, ctl->xfered, ctl->idlearg)));
    return rv;
}
```

#### net_read函数
这个函数简单，用了`read`函数读到了数据，就立马返回，但也要注意，你读10个字节，也不一定能读取10个字节，可能会比10个字节小，因为`read`总是在接收缓冲区有数据时立即返回，而不是等到给定的read buffer填满时返回。
```
int net_read(int fd, char *buf, size_t len)
{
    while ( 1 )
    {
        int c = read(fd, buf, len);
        if ( c == -1 )
        {
            if ( errno != EINTR && errno != EAGAIN )
            return -1;
        }
        else
        {
            return c;
        }
    }
}
```




参考：
* [使用 Socket 通信实现 FTP 客户端程序](https://www.ibm.com/developerworks/cn/linux/l-cn-socketftp/index.html)
* [网络编程中的read，write函数](https://www.cnblogs.com/wuchanming/p/3783650.html)