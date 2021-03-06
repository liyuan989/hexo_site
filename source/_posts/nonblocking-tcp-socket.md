---
title: 非阻塞TCP套接字的要点
id: 255
categories:
  - 网络编程
date: 2015-04-22 20:13:08
tags:
---

套接字的默认状态是阻塞的。如果一个套接字不能立即完成相应的调用，那么该线程就会被投入睡眠，等待相应的操作完成。阻塞一个套接字的操作可能是输入操作、输出操作、接受外来连接、发起外出连接这四种操作中的一种。如果把套接字改为非阻塞的话，这些操作就会变的不一样了。

*   输入操作，包括read、readv、recv、recvfrom和recvmsg这五个函数（aio系列函数除外，其为异步IO）。对于非阻塞的套接字，如果输入操作不能被满足（TCP套接字默认是至少有一个字节的数据可读，UDP套接字默认是有一个完整的数据报可读），相应的输入调用将立即返回EWOULDBLOCK错误（在Linux上EWOULDBLOCK与EAGAIN等价）
*   输出操作，包括write、writev、send、sendto和sendmsg这5个函数（aio系列函数除外，其为异步IO）。对于非阻塞的TCP套接字，如果其发送缓冲区根本没有空间，相关函数调用将立即返回EWOULDBLOCK错误。如果其发送缓冲区中有一些空间单不足以装下所有用户数据，返回值将是内核能够从用户缓冲区复制到内核发送缓冲区的字节数，也称「不足计数」。
*   接受外来连接，即accept函数。对于非阻塞的TCP套接字，如果调用accept函数但无连接到达，则立即返回一个EWOULDBLOCK错误。
*   发起外出连接，即connect函数。对于非阻塞的TCP套接字，如果调用connect并且连接不能立即建立，那么连接的建立可以照样发起，不过会返回一个EINPROGRESS错误。但此时并不能确定是连接发生错误还是已成功建立连接，具体内容后文再讨论。

## connect

在非阻塞TCP套接字上调用connect函数，将立即返回一个EINPROGRESS错误，不过已经发起的TCP三路握手继续进行（即调用connect使套接字发送SYN分节后立即返回EINPROGRESS）。然后就可以用epoll/poll/select检测该连接的成功或者失败。

相对于阻塞connect，非阻塞connect有下面几个用途：

1.  可以在TCP三次握手的时候进行其他操作。因为客户发生一个SYN分节，然后等到收到服务端的ACK分节后，客户端的连接才建立起来，这其中经过了一个RTT时间。但是RTT的实际波动范围较大，从局域网的几毫秒到广域网的几秒，非阻塞connect可以把这段时间利用起来处理其他事物。
2.  可以同时建立多个连接。
3.  可以通过epoll/poll/select设置超时时间。默认的阻塞connect的超时一般为75秒，如果想要设置更短的超时时间就可以通过非阻塞connect配合IO复用来完成。
    <!--more-->

非阻塞套接字不像阻塞套接字那样简单，有一些细节需要注意和处理：

*   尽管套接字是非阻塞的，如果连接的对端服务器在同一个主机上的话，那么调用connect时，连接通常是立即建立而不是返回EINPROGRESS错误，所以编程的时候得处理这种情形。
*   原则上说，当非阻塞connect与IO复用配合时是这样的：当建立连接成功时，描述符可写，当建立连接遇到错误时，描述变得既可写又可读。但是这就带来一个问题，假设在调用epoll_wait/poll/select之前连接已经建立并且已经收到了对端发送过来的数据（这是完全有可能的），这就和建立连接遇到错误时的读写条件一致了，都是既可读也可写。所以个人推荐的判断套件字连接建立成功的条件时，调用getsockopt检查获取套接字上的待处理错误，若没有错误则代表连接建立成功。

另外关于connect，在[man](http://linux.die.net/man/2/connect)手册上也提到了可以使用getsockopt检查SO_ERROR选项返回的值为0来判断连接的成功建立：

> The socket is nonblocking and the connection cannot be completed immediately. It is possible to _**[select](http://linux.die.net/man/2/select)**(2)_ or _**[poll](http://linux.die.net/man/2/poll)**(2)_ for completion by selecting the socket for writing. After _**[select](http://linux.die.net/man/2/select)**(2)_ indicates writability, use _**[getsockopt](http://linux.die.net/man/2/getsockopt)**(2)_ to read the**SO_ERROR** option at level **SOL_SOCKET** to determine whether **connect**() completed successfully (**SO_ERROR** is zero) or unsuccessfully (**SO_ERROR** is one of the usual error codes listed here, explaining the reason for the failure).

需要注意的是，通过getsockopt的SO_ERROR选项返回的值为0来判断操作成功只有这一种情形是正确的。其他情况这样判断是不可取的，是未定义行为。

另外对于阻塞的connect，当被系统调用或者信号中断，且内核不能自动重启的话，connect将EINTR，这个时候不能再次调用connect继续等待未完成的连接，这样做将回返回EADDRINUSE错误。这种情况，只能关闭该套接字，然后再通过socket获取新的套接字重新调用connect发起新的连接，或者通过非阻塞connect配合IO复用避免遇到这种情形。

## accept

当accept和IO复用epoll/poll/select配合时，如果有连接已经完成可以被accept时，将触发监听描述符的可读事情。所以，似乎把accept设置为非阻塞没什么用处，因为IO复用已经通知有新的连接完成了，所以再accept的时候就不会阻塞了。

我们可以想象一下下面这种情形：

1.  服务端使用阻塞的accept配合IO复用接受新连接，且服务端负载较高。
2.  某客户向服务端发起一个TCP连接，当连接建立成功后立即通过setsockopt设置SOL_LINGER选项使得close操作发送RST分节，设置完成后立即调用close。
3.  由于服务器端负载较高，IO复用的监听套接字可读操作就绪时没能立刻调用accept，在调用accept之前服务端主机已经收到了来自客户的RST分节，所以该连接被从内核中已完成的连接队列中删除，然后当服务器再调用accept时，由于没有已完成的连接，accept将导致该线程阻塞，由于accept的阻塞导致无法执行循环中的epoll_wait/poll/select，后续的到来的新连接将无法被通知，于是该线程就永远阻塞下去。

所以解决的办法就是把监听套接字设为非阻塞，再用accept配合IO复用。然后对于accept的返回值，忽略EWOULDBLOCK（BSD中的客户终止连接）、EAGAIN、ECONNABORTED（POSIX实现中的客户终止连接）、EPROTO（SVR4中的客户终止连接）、EINTR（被信号中断）。

## 输入和输出

一般对于非阻塞套接字的输入读取操作来说，可以忽略返回的错误或者将具体的错误打印到log中即可。其他具体的业务逻辑需具体对待。对于向一个收到RST分节的套接字进行读操作（正常关闭是第一次read返回0，第二次read返回ECONNRESET），若是采用的select，则需判断读操作的返回是否为ECONNRESET，若采用epoll/poll则无需处理，因为会触发EPOLLHUP/POLLHUP事件。

由于向一个收到RST分节的套接字进行写操作会导致内核向该进程发送sigpipe信号，所以一般最好在程序一开始就忽略掉该信号，然后通过检查写操作的返回值是否为EPIPE来判断此事件的发生。另外建议检查写操作返回值是否为ECONNRESET来判断向已关闭写的套接字的写操作，其他情况可类似与输入操作的处理，忽略返回错误或者将具体错误打印到log信息中。

## 参考文献

1.  W.Richard Stevens. [UNIX网络编程 卷1：套接字联网API（第3版）, 2010](http://book.douban.com/subject/4859464/)
2.  Linux man page. [connect(2)](http://linux.die.net/man/2/connect)
