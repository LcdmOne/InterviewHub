# Web网络云盘项目(有修改空间)

## 1.信号量(pv操作)

信号量是一个特殊的变量，当信号量值大于零，使用`sem_wait()`信号量减一，否则这个函数会导致线程等待。使用`sem_post()`函数会使信号量加一，如果有等待的线程会唤醒。

```c++
#include <semaphore.h>
int sem_init(sem_t * sem,int pshared,unsigned int value);//初始化信号量*sem值为value，pshared为零是局部信号量，否则多进程之间共享。
int sem_destory(sem_t * sem);//销毁一个信号量
int sem_wait(sem_t*sem);//信号量减一，为零时阻塞
int sem_trywait(sem_t*sem)//sem_wait()的非阻塞版本，为0时返回-1，并设置errno为EAGAIN。
int sem_post(sem_t*sem);//信号量加一，信号量大于零时被阻塞的线程会被唤醒。
//上述函数成功返回0，失败返回-1并设置errno.
```

## 2.互斥锁

互斥锁可以保护关键代码段使其独占式访问，相当于二进制信号量。

```c++
#include<pthread.h>
int pthread_mutex_init(pthread_mutex_t*mutex,const pthread_mutexattr_t*mutexattr);//初始化互斥量，mutexattr是互斥量的属性，填NULL则使用默认属性
int pthread_mutex_destory(pthread_mutex_t*mutex);//销毁互斥量
int pthread_mutex_lock(pthread_mutex_t*mutex);//加锁，如果已经被锁上将会阻塞。
int pthread_mutex_trylock(pthread_mutex_t*mutex);//加锁，如果已经被锁上将会返回错误码EBUSY。需要注意的是不同互斥量属性，导致的结果不同。
int pthread_mutex_unlock(pthread_mutex_t*mutex);//解锁。
```

## 3.条件变量

条件变量用于线程之间同步共享数据的值。它提供了一种通知机制，当某个共享数据达到某个值的时候，唤醒等待这个共享数据的线程。

```c++
#include<pthread.h>
int pthread_cond_init(pthread_cond_t*cond,cosnt pthread_condattr*cond_attr);//初始化，cond_attr为属性，与互斥量相似
int pthread_cond_destory(pthread_cond_t*cond);//销毁
int pthread_cond_broadcast(pthread_cond_t*cond);//通知所有等待的线程。
int pthread_cond_signal(pthread_cond_t*cond);//唤醒一个等待的线程。
int pthread_cond_waitj(pthread_cond_t*cond,pthread_mutex_t*mutex);//使线程等待，为了保证原子性，需要传入一个已加锁的互斥量。
```

## 4.I/O模型

阻塞IO，非阻塞IO，信号驱动IO，IO多路复用，异步IO。其中前四种为同步IO，与异步IO的区别是异步IO是由操作系统将数据拷贝到用户区并通知程序。

## 5.事件处理模式

Reactor事件处理模式，说白了就是对IO多路复用进行封装，将其搭配多线程或多进程，充分利用服务器资源。proactor是使用异步实现（linux目前没有实现真正的异步)。在这个项目中使用的是单reactor多线程模式。

## 6.线程相关函数

```c++
#include<phtread.h>
int pthread_create(pthread_t * thread,const pthread_arrt_t*attr,void*(*start_rountine)(void*),void* arg);//运行一个线程
void pthread_exit(void * retval);//在线程函数内调用以确保安全退出，并返回函数值
int pthread_join(pthread_t thread,void ** retval);//等待一个线程结束，并接受其返回值。
int pthread_cancel(pthread_t thread);//终止一个线程。
int pthread_detach(pthread_t thread);//不关心该线程返回值，即将该线程设置为脱离线程。
```

## 7.对线程的封装

我们知道线程会调用参数为`void*`,返回值为`void*`的函数。我的思路是使用策略模式，创建一个`threadkey`的类，将这个类的对象作为参数传入到线程函数中，并在线程函数中调用`dispose()`函数。线程创建时使用`pthread_detach()`进行分离线程。线程的销毁则是使用`pthread_cancel(thread_t pt)`函数进行销毁线程，并设置立即取消。

## 8.线程池的实现

对线程，互斥量，信号量，条件变量封装，并提供一个工厂生成。线程池继承了threadkey的接口。线程池通过构造函数传入的线程数以及工厂生成线程池，每个线程会调用线程池的`dispose()`函数，在这个函数有一个无限循环，首先调用信号量的`sem_wait()`函数进入等待。线程池同时会维护一个请求队列，对外提供一个`add(threadkey* key)`函数，这个函数里会将连接添加到请求队列里，并调用sem_post()。此时dispose()会从队列里取出一个请求并调用请求的`dispose()`函数。需要注意的是请求队列的操作需要加锁。

## 10.epoll相关函数

epoll相比select没有长度限制，也解决了轮询的问题，但是兼容性不如select。

```c++
#include <sys/epoll.h>
int epoll_create(int size);//size没有啥用
int epoll_ctl(int epfd,int op,int fd,int struct epoll_event*event);//op有三种操作，EPOLL_CTL_ADD，EPOLL_CTL_MOD，EPOLL_CTL_DEL.分别为添加、修改、删除。fd是要监控的描述符，event是设置如何监控。
int epoll_wait(int epfd,struct epoll_event*events,int maxevents,int timeout);//events是我们用来接受数据的，maxevents是说这个最大的大小，timeout是等待事件。
```

## 11.对epoll进行封装。

对epoll封装也用到了策略模式。我们提供了个`epollkey`抽象类，当有事件发生时，我们通过返回值对应的运行对应`epollkey`类中的`epolldispose()`。对epoll封装中有map用于处理`fd`与`epollkey`的关系。并提供添加、修改、删除、处理函数。需要注意的是如有一个处理需要用到线程池，需要同时继承`epollkey`与`threadkey`。

## 12.LT与ET模式

这个是指epoll文件莫舒服的操作方式，LT与ET模式。ET模式就是说当缓存区有数据时只会通知一次，所以说我们要一次性全部取出缓存区数据，使用函数读取，当返回值为-1，并设置errno为ENGAIN。使用ET我们需要使用fcntl设置为非阻塞。

## 13.EPOLLONESHOT

我们对一个http请求只需要一个线程处理，所以我们设置EPOLLONESHOT，是让epoll只监视一次，需要再次使用时需要重新注册事件（修改）

## 14.http请求报文

请求报文由请求行、请求头、空行、请求数据四个部分组成。

其中请求行由请求方法、要访问的资源、http版本。

请求头部，来说明服务器要使用的附加信息。

```http
HOST,给出请求资源所在服务器的域名
User-Agent,由客户端发送的信息。由浏览器定义
Accept,说明用户代理可以处理的媒体类型
Accept-Encoding,说明用户代理支持的编码内容
Accept-Language,说明用户代理能处理的自然语言
Content-Type,说明请求数据的类型
Content-Length,说明请求数据的大小
Connection,连接管理，由Keep-Alive与close
```

空行，用来隔离请求头部与请求体。

请求数据，也叫请求主体。通过`content-length`可以获取其长度。

## 15.http响应报文

响应报文由状态行、响应头、空行、响应正文组成。

其中响应头与空行与请求报文类似。状态行由http版本号，状态码，状态消息组成。常见的状态消息有

```http
1xx,表示已接受,继续处理
2xx,表示成功处理
3xx,重定向，要完成需要进一步操作
4xx,客户端错误
5xx,服务器错误
```

这个项目中用到的状态有，200 OK：表示成功处理，404 Bad Request：表示客户端错误，500 Internal Server Error:表示服务器错误。

## 16.如何处理读取http请求报文

项目使用的是ET模式，所以要一次性将缓冲区的数据读完，但需要注意的是缓冲区里数据并不一定是完整的HTTP请求。我们这里用到了状态设计模式，即将分析数据放到其他类中，这些类继承同一个接口`httpkey`，http类会调用这些类的`dispose()`函数，这个函数有三种返回结果，成功，失败，错误。很明显错误就关闭连接，并且在epoll中删除对应的监视。如果成功就会调用其他函数处理这个已经成功的数据，失败则不进行处理继续重新注册事件等待接受。成功读取的请求报文，会进行检测，目前服务器只能处理http1.1的get与post请求，否则会返回客户端错误。状态的转移分别是请求行转移请求头，请求头到空行判断是否有请求体，没有就ok，有就转移请求体处理。

## 17.如何处理get请求

get请求主要是分为已有静态页面，比如说登录注册页面或者是一个文件，我们可以直接获取这个文件的地址，如果是一个目录，我选择的方法是将生成动态html文件，再向写入函数传入这个文件的地址。这样做可以使写入函数统一，只需要维护最多两个iovec结构体。

## 18.如何处理post请求

post请求只有两种情况，一种是注册，一种是登录。读取请求体的数据后，会进行验证，如果验证成功，则加载云盘的主目录页面，如果失败则返回失败页面。

## 19.怎么判断是否是文件还是目录

通过stat获取文件的属性，通过`st_mode`参数判断是目录还是普通文件，如果是目录则使用`opendir()`函数打开目录并使用readdir进行遍历，生成对应的html文件并返回。如果是文件则直接返回其地址即可。

## 20.如何将数据发送给浏览器

使用的是writev函数，需要注意的是当发送数据过大，会导致缓冲区满，我们需要动态维护iovec的开始地址与长度。其中开始地址是mmap将文件映射到内存的地址。当有数据没有发送完成时要动态维护这个开始地址与长度。此时我们重新向epoll注册的是写事件，当可写时再次进行写，直到发送完整数据，才可以注册读事件。

## 21.怎么对post的登录注册进行验证

通过对数据库封装的单例类，进行验证。每次创建数据库连接都比较耗时间，所以我们可以在初始化时就创建多个连接，之后通过队列维护。使用互斥量对该队列进行访问限制，通过信号量维护该队列是否有连接。同时提供了执行sql语句的方法，http拿到连接后就进行使用sql语句进行查询与添加操作。

## 22.post怎么获取账号密码

以&号为分割符，前面为账号，后面为密码。

## 23.数据库相关函数。

```c++
#include<mysql.h>
mysql_real_connect(MYSQL* con,char * user,char * password,char * dbname,int port,NULL,0);//连接数据库，没有找到具体含函数
mysql_query(MYSQL* con,char * sql);//查询
MYSQL_RES* mysql_store_reault(MYSQL*con);//获取完整结果集
MYSQL_ROW mysql_fetch_row(MYSQL_RES* result);//获取对应的数组。
```

## 24.说说你认为这个项目的缺点

从功能性上讲，没有实现文件上传这个核心功能，从代码角度来讲，没有实现cookie等维持用户持久登录的方式，当http连接断开用户就需要重新登录。http类没有过于臃肿，各种功能都堆积到一个类。有些消息传递使用赋值传递，效率过低。还有就是没有对长时间没有传消息的连接进行断开连接的操作(考虑为epoll添加一个最大监视数量，这样就可以使用LRU这种结构）。还有就是项目没有一个日志系统。在现在我看来，这个项目还是有很多优化空间的。

## 25.说说这个项目的收获(优点)

优点是让我对知识有一个融汇贯通的过程，项目并不是比赛，不但要考虑功能的实现，更应该考虑扩展性以及可维护性。

## 26.项目的压力测试做了吗

使用webbench压力测试，可以抗住8000+的并发。



