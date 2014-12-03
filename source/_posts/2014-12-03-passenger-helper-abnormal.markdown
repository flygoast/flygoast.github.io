---
layout: post
title: "PassengerHelperAgent开机启动异常分析"
date: 2014-12-03 10:44:12 +0800
comments: true
categories: PASSENGER
---
异常现象：

在/etc/rc.local中添加/usr/local/nginx/sbin/nginx来开机自动启动NGINX时，PassengerHelperAgent进程不停反复重启，而从shell上手动启动NGINX时一切正常。

追查过程:

查阅异常时的error.log日志发现以下错误：
```
[ pid=3413 thr=140583025772288 file=ext/nginx/HelperAgent.cpp:963 time=2014-09-30 11:05:20.925 ]: Uncaught exception in PassengerServer client thread:
   exception: Cannot accept new connection: Too many open files (24)
   backtrace:
     in 'Passenger::FileDescriptor Client::acceptConnection()' (HelperAgent.cpp:429)
     in 'void Client::threadMain()' (HelperAgent.cpp:952)
```

根据日志错误信息，可以确定是PassengerHelperAgent进程文件描述符达到了上限。

找到ext/nginx/HelperAgent.cpp文件的963行：
```cpp
        void threadMain() {
                TRACE_POINT();
                try {
                        while (true) {
                                UPDATE_TRACE_POINT();
                                inactivityTimer.start();
                                FileDescriptor fd(acceptConnection());
                                inactivityTimer.stop();
                                handleRequest(fd);
                        }
                } catch (const boost::thread_interrupted &) {
                        P_TRACE(2, "Client thread " << this << " interrupted.");
                } catch (const tracable_exception &e) {
                        P_ERROR("Uncaught exception in PassengerServer client thread:\n"
                                << "   exception: " << e.what() << "\n"
                                << "   backtrace:\n" << e.backtrace());
                        abort();
                }
        }
```
通过上下文可以确定是调用acceptConnection()出错，查看acceptConnection()代码，确定是由该函数抛出的异常。
```cpp
        FileDescriptor acceptConnection() {
                TRACE_POINT();
                struct sockaddr_un addr;
                socklen_t addrlen = sizeof(addr);
                int fd = syscalls::accept(serverSocket,
                        (struct sockaddr *) &addr,
                        &addrlen);
                if (fd == -1) {
                        throw SystemException("Cannot accept new connection", errno);
                } else {
                        return FileDescriptor(fd);
                }
        }
```
syscalls::accept是对系统调用accept的简单封装。
```cpp
syscalls::accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen) {
        int ret;
        CHECK_INTERRUPTION(
                ret == -1,
                ret = ::accept(sockfd, addr, addrlen)
        );
        return ret;
}
```
错误原因就是accept由于进程文件描述符达到上限而出错返回了。

接下来追查为什么进程文件描述符数会达到上限。

首先看一下PassengerHelperAgent的整体代码逻辑:
```cpp
int
main(int argc, char *argv[]) {
        TRACE_POINT();
        VariantMap options = initializeAgent(argc, argv, "PassengerHelperAgent");
        pid_t   webServerPid  = options.getPid("web_server_pid");
        string  tempDir       = options.get("temp_dir");
        bool    userSwitching = options.getBool("user_switching");
        string  defaultUser   = options.get("default_user");
        string  defaultGroup  = options.get("default_group");
        string  passengerRoot = options.get("passenger_root");
        string  rubyCommand   = options.get("ruby");
        unsigned int generationNumber   = options.getInt("generation_number");
        unsigned int maxPoolSize        = options.getInt("max_pool_size");
        unsigned int maxInstancesPerApp = options.getInt("max_instances_per_app");
        unsigned int poolIdleTime       = options.getInt("pool_idle_time");

        try {
                UPDATE_TRACE_POINT();
                Server server(FEEDBACK_FD, webServerPid, tempDir,
                        userSwitching, defaultUser, defaultGroup,
                        passengerRoot, rubyCommand, generationNumber,
                        maxPoolSize, maxInstancesPerApp, poolIdleTime,
                        options);
                P_DEBUG("Passenger helper agent started on PID " << getpid());

                UPDATE_TRACE_POINT();
                server.mainLoop();
        } catch (const tracable_exception &e) {
                P_ERROR(e.what() << "\n" << e.backtrace());
                return 1;
        } catch (const std::exception &e) {
                P_ERROR(e.what());
                return 1;
        }

        P_TRACE(2, "Helper agent exited.");
        return 0;
}
```
首先创建一个Server对象，然后调用Server对象的mainLoop成员函数。Server对象构造函数会调用成员函数startListening。
```cpp
        void startListening() {
                this_thread::disable_syscall_interruption dsi;
                requestSocket = createUnixServer(getRequestSocketFilename().c_str());

                int ret;
                do {
                        ret = chmod(getRequestSocketFilename().c_str(), S_ISVTX |
                                S_IRUSR | S_IWUSR | S_IXUSR |
                                S_IRGRP | S_IWGRP | S_IXGRP |
                                S_IROTH | S_IWOTH | S_IXOTH);
                } while (ret == -1 && errno == EINTR);
        }
```
createUnixServer函数会创建一个socket文件，然后监听这个文件。NGINX收到请求后，会由Passenger模块转发请求到该socket文件。
mainLoop会调用成员函数startClientHandlerThreads，它会创建numberOfThreads个Client对象。
```cpp
        void startClientHandlerThreads() {
                for (unsigned int i = 0; i < numberOfThreads; i++) {
                        ClientPtr client(new Client(i + 1, pool, requestSocketPassword,
                                defaultUser, defaultGroup, requestSocket,
                                analyticsLogger));
                        clients.insert(client);
                }
        }
```
Client对象构造函数会启动一个线程执行threadMain。threadMain就是我们上面出错的函数。每个线程等待接收通过socket文件发来的请求，接收请求后调用handleRequest进行处理。
```cpp
        Client(unsigned int number, ApplicationPool::Ptr pool,
               const string &password, const string &defaultUser,
               const string &defaultGroup, int serverSocket,
               const AnalyticsLoggerPtr &logger)
                : inactivityTimer(false)
        {
                this->number = number;
                this->pool = pool;
                this->password = password;
                this->defaultUser = defaultUser;
                this->defaultGroup = defaultGroup;
                this->serverSocket = serverSocket;
                this->analyticsLogger = logger;

                sbmh_init(&statusFinder.ctx, NULL, NULL, 0);
                sbmh_init(&transferEncodingFinder.ctx, NULL, NULL, 0);

                thr = new oxt::thread(
                        boost::bind(&Client::threadMain, this),
                        "Client thread " + toString(number),
                        CLIENT_THREAD_STACK_SIZE
                );
        }
```
问题出在线程调用accept等待接收请求时。我们所创建的线程数量numberOfThreads是在Server对象被创建时指定的。
```cpp
numberOfThreads     = maxPoolSize * 4;
```
而maxPoolSize由passenger_max_pool_size配置项指定，我们指定的是256。256 × 4 ＝ 1024，开机启动时PassengerHelperAgent进程的文件描述符上限就是1024。这个数字值得怀疑。因而我将配置修改为128，果然正常了。
```nginx
passenger_max_pool_size 256;
```
可以确定每个线程中占用了文件描述符。然而从代码中并没有找到打开文件相关的逻辑。当accept成功返回时，会返回一个新的文件描述符。开始怀疑accept在还没有接收到请求时就预先占用了一个文件描述符。通过一个简单程序来验证。
```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/un.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <pthread.h>
#include <assert.h>

void *start_accept(void *arg) {
    int                 fd = (int) arg;
    int                 newfd;
    struct sockaddr_un  addr;
    socklen_t           addrlen;

    newfd = accept(fd, (struct sockaddr *)&addr, &addrlen);

    if (newfd < 0) {
        fprintf(stderr, "accept failed: %u: %s\n",
                pthread_self(), strerror(errno));
    }
}

int main() {
    int                 i, fd;
    struct sockaddr_un  addr;
    socklen_t           addrlen = sizeof(addr);
    pthread_t           pt;

    assert((fd = socket(PF_LOCAL, SOCK_STREAM, 0)) > 0);

    for (i = 0; i < 1020; i++) {
        assert(open("emfile.c", O_RDONLY) > 0);
    }

    addr.sun_family = AF_LOCAL;
    strncpy(addr.sun_path, "emfile.socket", sizeof("emfile.socket") - 1);
    addr.sun_path[sizeof("emfile.socket") - 1] = '\0';

    assert(bind(fd, (const struct sockaddr *)&addr, sizeof(addr)) == 0);

    assert(listen(fd, 512) == 0);

    pthread_create(&pt, NULL, start_accept, (void *) fd);

    pthread_join(pt, NULL);

    exit(0);
}
```
使用ulimit将shell文件打开数上限修改为1024:
```
$ ulimit -n 1024
```
编译验证程序，并执行
```
$ gcc emfile.c -lpthread
$ ./a.out
```
得到结果：
```
accept failed: 591611648: Too many open files
```
确实如些，那再来看一下accept的实现。accept系统调用的内核实现是sys_accept，而sys_accept是对sys_accept4的简单封装。
```c
SYSCALL_DEFINE3(accept, int, fd, struct sockaddr __user *, upeer_sockaddr,
        int __user *, upeer_addrlen)
{
    return sys_accept4(fd, upeer_sockaddr, upeer_addrlen, 0);
}
```
```c
SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr,
        int __user *, upeer_addrlen, int, flags)
{
    ...

    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (!sock)
        goto out;

    err = -ENFILE;
    if (!(newsock = sock_alloc()))
        goto out_put;

    newsock->type = sock->type;
    newsock->ops = sock->ops;

    ...

    newfd = sock_alloc_file(newsock, &newfile, flags);

    ...

    err = sock->ops->accept(sock, newsock, sock->file->f_flags);
    if (err < 0)
        goto out_fd;

    ...

    fd_install(newfd, newfile);
    err = newfd;

out_put:
    fput_light(sock->file, fput_needed);
out:
    return err;
out_fd:
    fput(newfile);
    put_unused_fd(newfd);
    goto out_put;
}
```
sys_accept4中在调用sock->ops->accept去接收网络请求前就调用sock_alloc_file来分配文件描述符。再来看sock_alloc_file这个函数:
```c
static int sock_alloc_file(struct socket *sock, struct file **f, int flags)
{
    struct qstr name = { .name = "" };
    struct path path;
    struct file *file;
    int fd;

    fd = get_unused_fd_flags(flags);
    if (unlikely(fd < 0))
        return fd;
    ...
}
```
sock_alloc_file会调用get_unused_fd_flags，这是一个宏，实际会调用函数alloc_fd, 而alloc_fd又会调用函数expand_files：
```c
int expand_files(struct files_struct *files, int nr)
{
    struct fdtable *fdt;

    fdt = files_fdtable(files);

    /*
     * N.B. For clone tasks sharing a files structure, this test
     * will limit the total number of files that can be opened.
     */
    if (nr >= current->signal->rlim[RLIMIT_NOFILE].rlim_cur)
        return -EMFILE;

    /* Do we need to expand? */
    if (nr < fdt->max_fds)
        return 0;

    /* Can we expand? */
    if (nr >= sysctl_nr_open)
        return -EMFILE;

    /* All good, so we try */
    return expand_fdtable(files, nr);
}
```
可以看到expand_files进行文件描述符限制的检查，当超过限制时返回"EMFILE"。"EMFILE"错误的提示就是"Too many open files"。
```c
#define EMFILE      24  /* Too many open files */
```
结合上面的测试程序，使用一个systemtap脚本可以捕获到上述调用路径。
```
probe  kernel.function("expand_files").return {
    if ($return == -24) {
        println("expand_files");
        printf("%d\n", $nr);
        print_backtrace();
        exit();
    }
}

probe begin {
    println("start\n");
}
```
执行stap:
```
$sudo stap emfile.stap
```
捕获结果为:
```
start

expand_files
1024
Returning from:  0xffffffff811931d0 : expand_files+0x0/0x220 [kernel]
Returning to  :  0xffffffff81193443 : alloc_fd+0x53/0x160 [kernel]
 0xffffffff814187f3 : sock_alloc_file+0x43/0x150 [kernel]
 0xffffffff8141b3dd : sys_accept4+0x11d/0x2b0 [kernel]
 0xffffffff8141b580 : sys_accept+0x10/0x20 [kernel]
 0xffffffff8100b0f2 : system_call_fastpath+0x16/0x1b [kernel]
```
最终确定异常原因:

开机自启时，进程的打开文件数限制为1024，而创建1024个线程执行accept()时。每个accept会占用一个文件描述符, 达到了进程的文件描述符上限而异常。而从shell启动时，我们shell进程的文件描述符限制是32768，因而不会出现问题。

解决方法：
创建一个启动脚本，在执行/usr/local/nginx/sbin/nginx前执行ulimit修改文件描述符限制。
```bash
ulimit -SHn 65535
/usr/local/nginx/sbin/nginx &
```
注意:

* /etc/security/limits.conf中的设置只针对登录动作发生时才生效，因而对于开机自动启动进程这种情况，这种修改该文件的方式不生效。
* Passenger版本为3.0.11
* kernel版本为CentOS 6.2内核，kernel-2.6.32-220.4.2.el6
