# OSLab6
@(标签)[BUAA][OSLab6]
##实验概况
首先本次实验中用到的一个模型思想就是客户端-服务器的模型思想，需要运行的进程中`sh.b`是无法直接启动的，只能通过`icode.x`和`serv.b`运行起来，再通过函数`spawn`来启动`sh.b`的shell进程。虽说实验的主要目的是填写`pipe.c`但其实光填写了`pipe.c`远远达不到本次实验的要求，实验对`fork.c`的权限位设置准确要求很高，因为实验引入了父子进程的共享内存来进行父子进程通信，这一点就像是Unix系统中的不命名管道一样。同时，实验中`fd.c`、`fsipc.c`等文件可以说准确地实现了一个服务器和客户端的通信模型。大概了解这么多，现在来看一下要填写的这些函数。

###_pipeisclosed###
```C
static int
_pipeisclosed(struct Fd *fd, struct Pipe *p)
{
        // Check pageref(fd) and pageref(p),
        // returning 1 if they're the same, 0 otherwise.
        检查fd的引用次数和p的引用次数，如果相等返回1，否则返回0。
        // The logic here is that pageref(p) is the total
        // number of readers *and* writers, whereas pageref(fd)
        // is the number of file descriptors like fd (readers if fd is
        // a reader, writers if fd is a writer).
        其实逻辑上来说因为p是被读者和写者共享的，而fd作为文件描述只是单方面的引用描述。
        这两者是否相等决定着当前两者之间是否有`trace`，即两者之间是否会有竞争。
        // If the number of file descriptors like fd is equal
        // to the total number of readers and writers, then
        // everybody left is what fd is.  So the other end of
        // the pipe is closed.
        如果fd的文件描述符的引用次数和读者、写者的总引用次数，那么就可以认定只有读者或目前只有写者在调度。
        int pfd,pfp,runs,r;
        r=0;
        do{
            pfd = pageref(fd);
            runs = env->env_runs;
            pfp = pageref(p);
        }while(runs != env->env_runs);

        if(pfd == pfp)
            r = 1;

        return r;
```
一开始其实我并没有填写关于`do while`结构的部分，因为觉得`env_runs`这个东西其实在`trace`里并没有什么用，即使可以同步，也是需要以不断的循环作为代价，感觉有一点取巧。同时`runs=env->env_runs`这条所在的顺序一定得是`pfd`和`pfp`的赋值语句之中，否则达不到我们想要的同步的效果。

###pipeisclosed###
这个才是正宗的pipeisclosed函数，让我们来看一下它是如何实现的。
```C
int
pipeisclosed(int fdnum)
{
        struct Fd *fd;
        struct Pipe *p;
        int r;

        if ((r = fd_lookup(fdnum, &fd)) < 0)
                return r;
        p = (struct Pipe*)fd2data(fd);
        return _pipeisclosed(fd, p);
}
```
首先我们获取到了文件描述符`fdnum`对应的fd，然后让`p = (struct Pipe*)fd2data(fd)`。这一句其实是异常巧妙的一点，p是一个管道的指针，那么为什么我们可以通过一个文件描述符的简简单单的`fd2data(fd)`就能获取到管道的指针呢？这里实际上是把管道当作文件来看的，通过观察`fd2data`函数，我们了解了如下内容：
```C
u_int
fd2data(struct Fd *fd)
{
        return INDEX2DATA(fd2num(fd));
        //#define INDEX2DATA(i)   (FILEBASE+(i)*PDMAP)
        //i

}

int
fd2num(struct Fd *fd)
{
        return ((u_int)fd - FDTABLE)/BY2PG;
}
```
参考MIT-JOS的描述，我们知道
`Return the file data pointer for file descriptor index i #define INDEX2DATA(i)	((char*) (FILEBASE + (i)*PTSIZE))`这个宏的意义在于返回这个文件描述符对应的文件的指针。之前我们把管道看作文件，所以可以直接使用`p = (struct Pipe*)fd2data(fd)`来获取管道的指针。
获取到管道指针之后，我们通过`fd`和`p`的引用次数是否相同来判断管道的另一端是否关闭。

##管道读写##
在开始写这两个函数之前，我还去参考了一下unix系统中的管道实现，因为在今年上unix课时老师提到过父子进程的管道读写，所以记忆比较深刻。关于不命名管道的读写实际上是这样，使用`dup`函数将父子进程的标准输入或标准输出和管道的一端进行绑定，然后即可以顺利实现管道。
但完全实现起来还是有困难，首先来看关于`pipe`的结构体：
```C
struct Pipe {
        u_int p_rpos;           // read position
        u_int p_wpos;           // write position
        u_char p_buf[BY2PIPE];  // data buffer
};
```
其中`p_rpos`是管道的读指针,`p_wpos`是管道的写指针,`p_buf`是管道的缓冲区，类似于一片小的缓冲区域。可惜我们的这个管道超小，只有32字节。
下面来看关于`piperead`的实现：
###piperead###
```C
static int
piperead(struct Fd *fd, void *vbuf, u_int n, u_int offset)
{
        // Your code here.  See the lab text for a description of
        // what piperead needs to do.  Write a loop that
        // transfers one byte at a time.  If you decide you need
        // to yield (because the pipe is empty), only yield if
        // you have not yet copied any bytes.  (If you have copied
        // some bytes, return what you have instead of yielding.)
        // If the pipe is empty and closed and you didn't copy any data out, return 0.
        // Use _pipeisclosed to check whether the pipe is closed.
        int i;
        struct Pipe *p;
        char *rbuf;
        p = (struct Pipe*)fd2data(fd);
        rbuf = vbuf;

        for(i = 0;i < n;i++){
            while ( p->p_rpos >= p->p_wpos) {
                if( i >0 )
                    return i;
                if(_pipeisclosed(fd,p))
                    return 0;
                syscall_yield();
            }
            rbuf[i] = p->p_buf[p->p_rpos % BY2PIPE];
        //    writef("rbuf:%c\n",rbuf[i]);
            p->p_rpos++;
        }
        return n;
}
```
管道读函数首先是要获取所打开的管道，那么使用刚才使用过的`fd2data`可以解决这个问题。获取到文件描述符打开的管道后，我们需要从管道中读字节，即将管道中的字节读到`vbuf`中去，而且注意这过程中我们需要移动读指针。
`  while ( p->p_rpos >= p->p_wpos) `当我们的读指针超过写指针时，即意味着要读一些未知之数，这样的话得到的数据是错误的，我选择直接返回`i`,这样一方面可以防止一开始的错误并传达错误信息，也可以在正常的情况末尾正确地返回`n`。在读的过程中我们需要判断管道是否关闭。同时如果读者是因为写者未写够而阻塞，则会一直调度写者，督促写者向管道中写入内容。  
如果我们的读指针是有效的(<写指针),那么我们可以开始读写了，读比较简单，将管道`p`的缓冲区中的内容读到`rbuf`中即可。并且注意读指针的移动。最后我们要返回读取的字符串的长度以通过`testpipe`里的匹配测试(正常返回的标志)。  

###pipewrite###
```C
static int
pipewrite(struct Fd *fd, const void *vbuf, u_int n, u_int offset)
{
        int i;
        struct Pipe *p;
        char *wbuf;
        p = (struct Pipe*)fd2data(fd);
        wbuf = vbuf;
        while(p->p_wpos - p->p_rpos >= BY2PIPE){
            if(_pipeisclosed(fd,p))
                return 0;
            syscall_yield();
        }
        i = 0;
        while(i<n){
            p->p_buf[p->p_wpos % BY2PIPE] = wbuf[i];
            p->p_wpos ++;
            i++;
            while((i<n) && (p->p_wpos - p->p_rpos>=BY2PIPE))
                syscall_yield();
            //When the p_buf is full and the message is not sent all.
        }

        return n;
}
```
写者大部分的思路和读者是一致的，不同的一点是初始时判断写指针的位置有效性时，因为写者是创造者，所以它担心的是缓冲区溢出，所以我们要保证`p->p_wpos-p->p_rpos <= BY2PIPE`，所以如果`>=`则需要调度让读者去读。  
注意这里如果写的字节没有够字符串的长度，写者不能自行退出，要被阻塞，需要写够字符串长度后才可结束。  

其实这两个函数写完如果之前的`fork.c`里的`LIBRARY`权限位设置没有问题的话，就应该可以顺利通过`testpipe`的测试。而这两个函数只是微观上的管道读写，我们需要有宏观的读写来统一管道读写机制，即在`sh.c`中的补充内容。  

##shell程序##
那么我们只剩下`sh.c`需要被填写，填写这个函数，我所填如下：
```C
case '|':
                        pipe(p);
                        if((r=fork())<0)
                            writef("fork error");
                        else if(r==0){
                            //in child
                            dup(p[0],0);
                            close(p[0]);
                            close(p[1]);
                            goto again;
                        }
                        else{
                            dup(p[1],1);
                            //close(p[0]);
                            close(p[1]);
                            close(p[0]);
                            goto runit;
                        }
```
这是要填写的部分，其实这部分流程也比较简单：  
shell在fork之后，未命名管道实现机制如下:  
+ 父进程的控制的写文件复制到标准输出文件（1号文件）里
+ 子进程控制的读文件复制到标准输入文件（0号文件）
+ 然后各自解析命令并运行程序。
其实这里的dup是映射，其实也是绑定的意思，在我们的实验中绑定这些即可。  

##总结##
其实lab6本身的内容并不是很难，难的地方在于之前的许多坑，比如`fork.c`,`pmap.c`,`syscall_all.c`,还有连着的`writef`的bug，许多的bug交织在一起，就会显得毫无调试头绪。

2015/7/4 乾
