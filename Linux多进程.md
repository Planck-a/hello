1、多进程下的文件描述符
---
父进程打开文件，父子进程同时往文件中写
```
int fd=open("1.txt",0_RDWR);
int pid=fork();
if(pid >0)
{
  //parent
  write(fd,"parent",6);
  close(fd);
}
if(pid == 0)
{
  write(fd,"child",6);
  close(fd);
  exit(0);
}
```
fork()出子进程后，父子进程谁先执行写操作，顺序是不确定的

父子进程拥有各自的内存空间，每个内存空间有文件描述符表，fd0是标注输入，fd1是标准输出，fd2是标准库，之后空白的文件描述符用来装载打开的文件描述符，fd3可能就是我们打开的文件描述符。
fd3指向了文件状态标识表+文件表，所以父子进程通过各自的fd指向同一个文件表+文件状态标识表；

由于父子进程中打开了该文件，所以相当于文件状态的引用计数是2，关闭时父子进程都要close；

## 父子进程共享文件描述符

* 1、父进程和子进程可以共享打开的文件描述符。
* 2、父子进程共享文件描述符的条件：在fork之前打开文件。
* 3、对于两个完全不相关的进程，文件描述符不能共享。
* 4、父子进程文件描述符是共享的，但是关闭的时候可以分别关闭，也可以同时在公有代码中关闭

2 进程终止的5种方式
--
正常退出：(1)main函数返回 (2)exit  (3)_exit  

异常退出：(4)调用abort    (5)ctrl+c 

(1)exit是c库函数，相当于封装了_exit，先做了清除缓冲区操作，再调用_exit
```
void main()
{
  printf("abcd");
  exit(0);
}
```
会先打印出abcd，再退出；

(2)_exit是系统调用，直接陷入Linux内核，终止进程运行
- 情况一：
```
void main()
{
  printf("abcd");
  _exit(0);
}
```
不会打印abcd，直接退出了
- 情况二：
```
void main()
{
  printf("abcd\n");
  _exit(0);
}
```
\n会刷新缓冲区，所以又能先打印出abcd，再退出；
- 情况三：
```
void main()
{
  printf("abcd");
  fflush(stdout);
  _exit(0);
}
```
手动刷新缓冲区，所以又能先打印出abcd，再退出；
- 情况四：
注册回调函数，exit退出后自动调用回调函数；可以注册多个回调函数，先注册的后调用;exit之后会调用这个函数。
```
void bye1()
{
  printf("bye1");
}
void bye2()
{
  printf("bye2");
}

void main()
{
  atexit(bye1);
  atexit(bye2);
  printf("abcd");
  fflush(stdout);
  _exit(0);
}
```
3 进程睡眠
---
进程睡眠分为可中断睡眠和不可中断睡眠两种，ctrl+c能杀死的就是可中断睡眠

举例：
```
void main()
{
  sleep(100);//可中断睡眠，ctrl+c可以杀死
}
```

4、查看某个进程下活跃的线程数： 
---
 1.根据进程号进行查询：

pstree -p 进程号

top -Hp 进程号

2.如果不知道进程号，通过awk得到后作为输入进行查询：

pstree -p `ps -e | grep server | awk '{print $1}'`

pstree -p `ps -e | grep server | awk '{print $1}'` | wc -l

这里利用了管道和命令替换，

关于命令替换，就是说用/`/`括起来的命令会优先执行，然后以其输出作为其他命令的参数，

上述就是用 ps -e | grep server | awk '{print $1}' 的输出（进程号），作为 pstree -p 的参数

5、linux下最多可以出来创建多少个进程
---
ulimit-a|grep process   查看当前系统下最大进程数目

或者： ulimit -u 查看当前系统下最多可以多少线程。一般查询出 1024，但是这属于软限制，是可以改变的


- Ulimit-a 是linux各种参数的汇总，配合grep可以达到ulimit-u效果。
- Linux下最大进程数：ulimit-u  查看是1024，可以自己修改
- Linux下最大线程数：ulimit-s 查看线程的栈空间大小，一般为8M。3G/8M,大约380个

最大线程数目和每个线程栈大小有关（ ulimit -s 可以查看默认的线程栈大小一般情况下，这个值是 8M）

## 进程下最大线程数目的计算：

32 位 linux 下的进程用户空间是 3G 的大小，也就是 3072M，线程栈大小一般情况下值是 8M，用 3072M 除以 8M 得 384，。进程的代码段和数据段等还要占用一些空间，所以这个值应该向下取整到 383。再减去主线程，得到 382。 linuxthreads 还需要一个管理线程，所以实际可用的最大线程数目就是381。

为了突破内存的限制，可以有两种方法：

-  用 ulimit -s 1024 减小默认的栈大小

-  调用 pthread_create 的时候用 pthread_attr_getstacksize 设置一个栈较小的子线程
 


6、最大文件描述符的数量
---
65535

 
 
 
 
 
 
 
