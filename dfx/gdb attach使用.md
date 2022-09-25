# Linux下GDB中的 attach pid 如何使用？

https://blog.csdn.net/lovemysea/article/details/78397532

https://blog.csdn.net/tuijiangmeng87/article/details/84069292

https://blog.csdn.net/qq_36414647/article/details/96013436
https://blog.csdn.net/fengbingchun/article/details/99417062

linux下使用gdb可以很好的跟踪代码。

当然，让我觉得神奇的是它竟然能跟踪正在运行的进程。

下面，我将用我的例子演示一下怎么使用的。

### 第一步：获得正在运行的进程的进程号

```
ps -ef | grep <进程名>
```

我的就是：

[![](https://img-blog.csdnimg.cn/2018111416474438.png)](https://img-blog.csdnimg.cn/2018111416474438.png)

找到该进程的进程id，我的就是2486400， 下面根据这个进程号，attach到这个进程上去。

### 第二步： gdb attach <pid>

根据上一步获得进程号，现在attach上去：

[![](https://img-blog.csdnimg.cn/20181114173152561.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3R1aWppYW5nbWVuZzg3,size_16,color_FFFFFF,t_70)](https://img-blog.csdnimg.cn/20181114173152561.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3R1aWppYW5nbWVuZzg3,size_16,color_FFFFFF,t_70)

### 第三步：打断点

gdb有两种打断点的方式：

1. gdb + 行号： 如果是当前文件，则直接加上行号。gdb进入的不是当前文件，则需要加上文件名。类似与相对路径和绝对路径

我的例子就是需要加上文件名，我需要在   /home/ceph/src/client/fuse_ll.cc 源码文件中的 591行处打断点，也就是fuse_ll_write函数入口：

[![](https://img-blog.csdnimg.cn/20181114173701403.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3R1aWppYW5nbWVuZzg3,size_16,color_FFFFFF,t_70)](https://img-blog.csdnimg.cn/20181114173701403.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3R1aWppYW5nbWVuZzg3,size_16,color_FFFFFF,t_70)

```
(gdb) b /home/ceph/src/client/fuse_ll.cc:591
Breakpoint 1 at 0x556ae37be908: file /home/ceph/src/client/fuse_ll.cc, line 591.
(gdb)
```

2. 第二种打断点的方法是，gdb + 函数名，在我的例子里，就是：

```
b /home/ceph/src/client/fuse_ll.cc:fuse_ll_write
```

这两种打断点的方法有一样的效果。

### 第四步：触发断点

触发断点有很多情况，比如，输入某个特定的值，删除某个文件，发送某个进程的信号。

在我这里，是写某个文件：

[![](https://img-blog.csdnimg.cn/20181114175728274.png)](https://img-blog.csdnimg.cn/20181114175728274.png)

可以看到，进程暂停了，没有正确返回。

现在，我们去gdb去查看是不是真的触发了断点，进程停到断点处：

我们输入c（continue的意思）：

```
(gdb) c
Continuing.
[Switching to Thread 0x7f285e0a1700 (LWP 2486426)]

Thread 23 "ceph-fuse" hit Breakpoint 1, fuse_ll_write (req=0x556aed447780, ino=1099511627799, buf=0x556aed754050 "hellon", size=6, off=0, fi=0x7f285e0a03c0)
    at /home/ceph/src/client/fuse_ll.cc:594
warning: Source file is more recent than executable.
594	  CephFuse::Handle *cfuse = fuse_ll_req_prepare(req);
(gdb)
```

[![](https://img-blog.csdnimg.cn/20181114175931301.png)](https://img-blog.csdnimg.cn/20181114175931301.png)

现在，我们就可以从断点处追踪程序了（输入n或者s执行下一步，输入n 进行下一步时会跳过函数，输入s会进入每一个调用函数）。

大功告成！

