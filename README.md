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
