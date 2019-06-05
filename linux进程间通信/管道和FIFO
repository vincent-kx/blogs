#《unix网络编程-进程间通信》管道和FIFO
管道是unix最初的ipc形式，所有样式的unix都提供管道，它由pipe函数创建。
由于pipe函数创建的管道只能用于有亲缘关系的进程间的通信，也就是父子进程之间的通信，
为了能在无亲缘关系间实现管道通信，1982年system III Unix引入了FIFO（named pipe有名管道），
管道和FIFO有“随进程持续性”，当最后一个打开管道的进程退出的时候，管道会自动销毁，其中的数据也
自动丢弃
###管道
```c
#include <unistd.h>

/* On Alpha, IA-64, MIPS, SuperH, and SPARC/SPARC64; see NOTES */
struct fd_pair {
    long fd[2];
};
struct fd_pair pipe();

/* On all other architectures */
int pipe(int pipefd[2]);

#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <fcntl.h>              /* Obtain O_* constant definitions */
#include <unistd.h>

int pipe2(int pipefd[2], int flags);
```
pipe函数会同时打开两个fd，其中pipefd[0]用于读取，pipefd[1]用于写入，
如果两个进程通信需求为一读一写那么一个管道就够了，通信模式如下：
```
        write  ------------- read
process1 ------>   pipe     ------> process2
               -------------
```
如果两个进程都需要同时读写，那么同学模式如下：
```
      write   ------------- read
    ------->      pipe1     ---------->
    |         -------------           |
    |                                 |
process1                           process2
    |                                 |
    |    read  ------------- write    |
    <----------    pipe2     <------  | 
               -------------
```
这种模式下父子进程需要分别关闭两个管道的一个读fd和一个写fd，假设有两个进程：
parent，child使用pipe1_fd和pipe2_fd进行通信，那么parent应该关闭pipe1_fd[0]
和pipe2_fd[1]，而child应该关闭pipe1_fd[1]和pipe2_fd[0],这样parent使用
pipe1_fd[1]进行写入，child则从pipe1_fd[0]读出parent的写入；child使用
pipe2_fd[1]进行写入，parent使用pipe2_fd[0]读出child的写入，这样就形成了上面
所说的第二种模式的通信。针对这中模式我们下面分别来看两个简单的例子：
```c
//头文件
#include <iostream>
//for pipe
#include <unistd.h>
//for exit
#include <stdlib.h>
//for fcntl
#include <fcntl.h>
//for strerror
#include <string.h>
//for errno
#include <errno.h>
//for waitpid
#include <sys/wait.h>
```
```c
//sample-1 管道
void test_pipe_mode1()
{
    int pipeFd[2] = {0};
    int ret = pipe(pipeFd);
    if(0 != ret) {
        print_errmsg("failed to make pipe");
        exit(-1);
    }

    pid_t cpid = fork();
    if(cpid == 0) { //child
        //从管道的写入端写入数据，（尝试了一下如果我们使用pipeFd[0]写入数据，读取的时候什么都读不到，这里没有检查写入是否成功，说明pipeFd[0]是无法写入数据的）
        string child_say = "老爸，我要吃糖果";
        write(pipeFd[1],child_say.c_str(),child_say.length());
        close(pipeFd[1]);
        exit(0);
    } else if (cpid > 0) { //parent
        sleep(1); //等待子进程数据写入完成
        int flags = fcntl(pipeFd[0],F_GETFL);
        if(flags < 0) {
            print_errmsg("fcntl failed");
            exit(-1);
        }
        //设置为非阻塞模式读，否则读完数据之后进程会阻塞住，无法自动退出
        fcntl(pipeFd[0],F_SETFL, flags | O_NONBLOCK);
        char farther_listen[64];
        int nbytes = 0;
        write(STDOUT_FILENO,"my son said:",strlen("my son said:"));
        //每次64字节，从管道读取端pipeFd[0]读出子进程写入的数据
        while((nbytes = read(pipeFd[0],farther_listen,64)) > 0) {
            write(STDOUT_FILENO,farther_listen,nbytes);
        }
        
        write(STDOUT_FILENO,"\n",1);
        close(pipeFd[0]);
        waitpid(cpid,NULL,0);
        exit(0);
    } else {
        print_errmsg("failed to fork");
    }
}
```
```c
//sample-2 管道
int set_non_block(int fd) {
    int flags = fcntl(fd,F_GETFL);
    if(flags < 0) {
        print_errmsg("fcntl failed");
        return -1;
    }
    return fcntl(fd,F_SETFL, flags | O_NONBLOCK);
}

void producer(int readFd,int writeFd) {
    if(0 != set_non_block(writeFd)) exit(-1);
    string request = "AAAAAAAAAAAABBBBBBBBBBBBBBBBCCCCCCCCCCCCCCCCCDDDDDDDDDDDDDDDDDDD\n";
    char response[64];
    ssize_t nbytes = write(writeFd,request.c_str(),request.length());
    close(writeFd);
    while( (nbytes = read(readFd,response,64)) > 0) {
        if(strcmp(response,"OK")) {
            cout << "product recv ok response" << endl;
            close(readFd);
            return;
        }
    }
}

void consumer(int readFd,int writeFd) {
    if(0 != set_non_block(writeFd)) exit(-1);
    sleep(1);
    char request[64];
    string response = "OK";
    ssize_t nbytes = 0;
    write(STDOUT_FILENO,"consumer recv:",strlen("consumer recv:"));
    while((nbytes = read(readFd,request,64)) > 0) {
        write(STDOUT_FILENO,request,nbytes);
    }
    close(readFd);
    write(writeFd,"OK",2);
    close(writeFd);
}

void test_pipe_mode2() {
    int pipeFd1[2] = {0};
    int pipeFd2[2] = {0};

    int ret = pipe(pipeFd1);
    if(0 != ret) {
        print_errmsg("failed to make pipe");
        exit(-1);
    }

    ret = pipe(pipeFd2);
    if(0 != ret) {
        close(pipeFd1[0]);
        close(pipeFd1[1]);
        print_errmsg("failed to make pipe");
        exit(-1);
    }
    pid_t cpid = fork();
    if(cpid == 0) { //child
        close(pipeFd1[1]);
        close(pipeFd2[0]);
        producer(pipeFd1[0],pipeFd2[1]);
        exit(0);
    }else if (cpid > 0) { //parent
        close(pipeFd1[0]);
        close(pipeFd2[1]);
        consumer(pipeFd2[0],pipeFd1[1]);
        waitpid(cpid,NULL,0);
        exit(0);
    } else {
        print_errmsg("failed to fork");
    }
}

该部分的测试代码源文件请参考pipe_test.cpp
```
###FIFO
FIFO可以用作有亲缘关系的进程间通信，也可以用户无亲缘关系的进程间通信
```c
#include <sys/types.h>
#include <sys/stat.h>
/**
使用pathname指定的路径创建或者打开一个FIFO
mode：O_RDWR O_NONBLOCK O_CREAT O_TRUNC O_CLOEXEC ...
使用mkfifo创建的FIFO可以被任何打开它的进程用来读写，但是在FIFO的两端都被打开之前，
任何读写操作都无法执行，当打开来读的时候会阻塞，直到对端有数据写入，同样，打开来写的
时候也会阻塞，直到对端被某个进程打开来读取数据。
内核不会将FIFO中交换的数据写入文件系统，FIFO可以使用非阻塞模式打开，这种情况下只有以
read-only的方式打开才会成功，而已write-only方式打开时会返回一个ENXIO失败，除非对端
已经被打开了（在linux系统下以阻塞和非阻塞方式打开都能成功，POSIX并未对此进行定义）
*/
int mkfifo(const char * pathname,mode_t mode);

#include <fcntl.h>
#include <sys/stat.h>
/**
完成和mkfifo相同的工作，不同点在于：
如果pathname指定的是一个相对路径，那么这个相对路径是相对于dirfd指向的目录，
而不是相对于当前目录（mkfifo是相对于当前目录）
如果pathname指定的是一个绝对路径，那么dirfd参数将被忽略掉
*/
int mkfifoat(int dirfd,const char * pathname,mode_t mode);
```
```c
#include <iostream>
//for pipe
#include <unistd.h>
//for exit
#include <stdlib.h>
//for fcntl
#include <fcntl.h>
// for strerror
#include <string.h>
//for errno
#include <errno.h>
//for waitpid
#include <sys/wait.h>
//for mkfifo
#include <sys/stat.h>

using namespace std;

#define FILE_MODE (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH)
#define BUF_SIZE 16

void print_errmsg(string errmsg)
{
    cout << errmsg << ",errno:" << errno << ",errmsg:" << strerror(errno) << endl;
    //reset
    errno = 0;
}

int set_non_block(int fd)
{
    int flags = fcntl(fd, F_GETFL);
    if (flags < 0)
    {
        print_errmsg("fcntl failed");
        return -1;
    }
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

void client()
{
    const char * fifo1 = "./test.fifo1";
    const char * fifo2 = "./test.fifo2";
    const char * file = "./fifo.test";

    int fifo1_fd = open(fifo1,O_WRONLY);
    if(fifo1_fd <= 0) {
        unlink(fifo1);
        unlink(fifo2);
        exit(-1);
    }

    int fifo2_fd = open(fifo2,O_RDONLY);
    if(fifo1_fd <= 0) {
        close(fifo1_fd);
        unlink(fifo1);
        unlink(fifo2);
        exit(-1);
    }
    int fd = open(file,O_RDONLY);
    if(fd < 0) {
        close(fifo1_fd);
        close(fifo2_fd);
        unlink(fifo1);
        unlink(fifo2);
        exit(-1);
    }
    set_non_block(fd);
    set_non_block(fifo1_fd);
    char line[BUF_SIZE] = {0};
    ssize_t nbytes = 0;
    //读出文件中的内容，发送给server
    while( (nbytes = read(fd,line,BUF_SIZE)) > 0) {
        write(fifo1_fd,line,nbytes);
        memset(line,0,BUF_SIZE);//clear
        //read server response
        read(fifo2_fd,line,BUF_SIZE);
        cout << "client recv:" << string(line);
    }
    // write(fifo1_fd,"OK",2);
}

void server()
{
    const char * fifo1 = "./test.fifo1";
    const char * fifo2 = "./test.fifo2";
    int ret = mkfifo(fifo1,FILE_MODE);
    if(ret < 0) {
        print_errmsg("server mkfifo failed");
        exit(-1);
    }
    ret = mkfifo(fifo2,FILE_MODE);
    if(ret < 0) {
        unlink(fifo1);
        print_errmsg("server mkfifo failed");
        exit(-1);
    }
    int fifo1_fd = open(fifo1,O_RDONLY);
    if(fifo1_fd <= 0) {
        print_errmsg("open fifo1 failed");
        unlink(fifo1);
        unlink(fifo2);
        exit(-1);
    }

    int fifo2_fd = open(fifo2,O_WRONLY);
    if(fifo2_fd <= 0) {
        print_errmsg("open fifo2 failed");
        close(fifo1_fd);
        unlink(fifo1);
        unlink(fifo2);
        exit(-1);
    }

    set_non_block(fifo2_fd);
    ssize_t nbytes = 0;
    char line[BUF_SIZE] = {0};
    while( (nbytes = read(fifo1_fd,line,BUF_SIZE)) > 0) {
        cout << "server recv:" << string(line);
        if(strcmp(line,"\n") == 0) { //文件传输结束了
            cout << "server end" << endl;
            close(fifo1_fd);
            close(fifo2_fd);
            unlink(fifo1);
            unlink(fifo2);
            sleep(1);
            break;
        }
        write(fifo2_fd,line,nbytes); //将接收到的数据写会给client
        memset(line,0,BUF_SIZE);
    }
    cout << endl;
}

int main()
{
    // client();
    server();
}
```
说明：上面的测试程序为了方便观察，特意设定了fifo.test文件的内容每行为15个字符+一个换行符

###管道和FIFO的一些限制
> 1.OPEN_MAX 一个进程可以在任意时刻打开的最大及描述符数，Posix要求至少是16
> 2.PIPF_BUF 可原子的写入一个管道或者FIFO的最大数据量，Posix要求至少是512

这两个值可以通过sysconf(_SC_OPEN_MAX)和sysconf(_SC_PIPE_BUF)来获取
###管道和FIFO的额外属性
1. 如果请求读取的数据量多余管道或者FIFO中的可用数据量，那么只返回那些可用的数据
2. 如果请求写入的数据字节数小于PIPE_BUF，那么写入操作保证是原子的，如果请求写入的数据大于PIPE_BUF，那么不能保证是原子的
3. O_NONBLOCK标志设置不影响写入的原子性，然而当一个管道或者FIFO设置成非阻塞之后，write的返回值取决于待写入的字节数(S)和管道或者FIFO剩余空间(E)的大小

如果写入的字节数小于等于PIPE_BUF：
> E >= S 所有数据都写入成功
> E < S 返回EAGAIN（已经设置非阻塞，而且内核要保证写入的原子性，这个时候就没法写入，只能返回失败）

如果写入的字节数大于PIPE_BUF：
> E >= 1，写入E字节的数据，write的返回值也是E
> E == 0，返回EAGAIN

4.如果向一个没有为读打开着的管道或者FIFO写入，那么内核将产生一个SIGPIPE信号，如果该信号未被capture，程序默认终止，如果信号被忽略或者capture后从信号处理函数中返回，那么write返回EPIPE错误
