# TheLinuxWays

## 一切皆文件

前面讲了那么多SoC理论课, 你可能已经有点疲惫了. 让我们快点回到Linux本身吧.  

Linux的世界, 一切皆文件. 如何理解?  
你去看看`/dev`目录就好了. `cd /dev`. 在你的Linux开发板或者Ubuntu虚拟机上敲一下看看! 不出意外, 你会看到这样的东西:  

```bash
root@ubuntu:/dev# ls
autofs           fd0u1440  loop5               ram11   ram7      tty11  tty27  tty42  tty58      ttyS11  ttyS27  ttyS6    vcsa1   vcsa4   vcsa7
block            fd0u1680  loop6               ram12   ram8      tty12  tty28  tty43  tty59      ttyS12  ttyS28  ttyS7    vcsa2   vcsa5   vcsa8
bsg              fd0u1722  loop7               ram13   ram9      tty13  tty29  tty44  tty6       ttyS13  ttyS29  ttyS8    vcsa3   vcsa6   vcsa9
btrfs-control    fd0u1743  loop-control        ram14   random    tty14  tty3   tty45  tty60      ttyS14  ttyS3   ttyS9    vcsa4   vcsa7   vfio
bus              fd0u1760  mapper              ram15   rfkill    tty15  tty30  tty46  tty61      ttyS15  ttyS30  uhid     vcsa5   vcsa8   vga_arbiter
char             fd0u1800  mcelog              ram2    rtc       tty16  tty31  tty47  tty62      ttyS16  ttyS31  uinput   vcsa6   vcsa9   vhci
console          fd0u1840  mei0                ram3    rtc0      tty17  tty32  tty48  tty63      ttyS17  ttyS4   urandom  vcsa7   vfio    vhost-net
core             fd0u1920  mem                 ram4    sda       tty18  tty33  tty49  tty7       ttyS18  ttyS5   v4l      vcsa8   vga_arbiter  vhost-vsock
cpu_dma_latency  fd0u2880  memory_bandwidth    ram5    sda1      tty19  tty34  tty5   tty8       ttyS19  ttyS6   vcs      vcsa9   vhci    video0
cuse             fd0u360   mqueue              ram6    sda2      tty2   tty35  tty50  tty9       ttyS2   ttyS7   vcs1     vga_arbiter  vhost-net  zero
```

这些是什么? 对, 设备. 设备也变成文件了. 非常神奇是不是?  你甚至可以操作这些文件, 读写这些文件, 从而控制硬件外设:  
⚠️ **WARNING! 请不要随意乱写这些设备, 除非你知道自己在干什么! 否则, 你可能会让操作系统崩溃, 或者造成不可撤销的数据丢失!**

```bash
root@ubuntu:/dev# hexdump -C /dev/sda | head -n 10
00000000  eb 3c 90 4d 53 44 4f 53  35 2e 30 00 02 08 20 00  |.<.MSDOS5.0... .|
00000010  02 00 00 00 00 f8 00 00  3f 00 ff 00 00 08 00 00  |........?.......|
00000020  00 00 00 00 80 00 29 0b  1f 00 00 00 00 00 00 00  |......).........|
00000030  80 00 01 00 00 00 00 00  29 0b 1f 00 00 00 00 00  |........).......|
00000040  00 00 00 00 00 00 00 00  02 00 00 00 01 00 06 00  |................|
00000050  00 00 00 00 00 00 00 00  00 00 29 0b 1f 00 00 00  |..........).....|
00000060  00 00 00 00 00 00 00 00  00 00 00 00 29 0b 1f 00  |............)...|
00000070  00 00 00 00 00 00 00 00  00 00 00 00 29 0b 1f 00  |............)...|
00000080  00 00 00 00 00 00 00 00  00 00 00 00 29 0b 1f 00  |............)...|
00000090  00 00 00 00 00 00 00 00  00 00 00 00 29 0b 1f 00  |............)...|
```

可以看到, 这个操作成功的把硬盘的第一个扇区读取出来了. 当然, 如果往这里写东西, 那后果肯定是灾难性的.  

除此之外, 甚至进程也变成了文件, 让我们来看看`proc`目录:  

```bash
root@ubuntu:# ls /proc
1      10     12     14     16     18     2      20     22     24     26     28     3      30     32     34     36     38     4      40     42     44     46     48     5      50     52     54     56     58     6      60     7      9      acpi              buddyinfo         cmdline           crypto            diskstats         execdomains       filesystems       interrupts        kallsyms          kcore             kmsg              kpagecgroup       kpagecount        kpageflags        loadavg           locks             mdstat            meminfo           misc              modules           mounts            mtrr              pagetypeinfo      partitions        sched_debug       schedstat         scsi              self              slabinfo          softirqs          stat              swaps             sys               sysrq-trigger     sysvipc           timer_list        timer_stats       tty               uptime            version           version_signature vmallocinfo       vmstat            zoneinfo
```

这里的每一个数字都是一个进程, 你可以`cat /proc/1/cmdline`来看看, 你会发现, 这个进程就是`init`进程.  

```bash
root@ubuntu:# cat /proc/1/cmdline
/sbin/init
```

甚至还可以指向自己的进程, 让我们来看看:
```bash
root@ubuntu:# cat /proc/self/cmdline
/bin/bash
```

而`/sys`目录也是非常重要的, 它包含了当前系统的所有硬件信息和各种驱动的参数等. 这里不再赘述, 如果你感兴趣, 可以自行去看看.

## 这是如何做到的?
没错, 一切都是文件, 这是Linux的一个非常重要的设计思想. 但是, 这是如何做到的呢?  
要研究是怎么做到的, 我们不得不聊一下Linux的**VFS**.

**VFS**, 全称是**Virtual File System**, 中文名叫**虚拟文件系统**. 和我们常用的Windows不一样, Linux额外实现了一个虚拟层.  
这一层不针对任何硬件. 它本身是空的, 什么东西都没有. 但是, 任何一个硬件都可以把自己`挂载`到这个虚拟层的特定位置上. 
让我们`mount`一下, 看看都挂载了什么东西:  

```bash
root@ubuntu:# mount
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
udev on /dev type devtmpfs (rw,nosuid,relatime,size=1018848k,nr_inodes=254712,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,noexec,relatime,size=204800k,mode=755)
/dev/sda2 on / type ext4 (rw,relatime,errors=remount-ro)
/dev/sdb1 on /mnt/usb type ext4 (rw,relatime)
```

我们可以看到这里面有sysfs, proc, devtmpfs, devpts, tmpfs, 还有我们熟悉的sda2和sdb1, 之前在`/dev`目录下看到的设备.  
这些设备都被挂载到了虚拟文件系统的不同位置上, 它们共同构成了Linux的文件系统.  

值得注意的是sysfs, proc udev, tmpfs这类的特殊类型文件系统. 这些文件系统由Linux中的**内核模块**虚拟出来的, 里面的内容是通过代码动态生成的.  
也就是说, 实际上并没有真正的这几个设备. 它们接管了文件系统的接口, 当你访问到了它们的时候, 它们会根据你的请求, 查询Linux系统当前的状态, 然后动态生成文件内容.  

至于怎么样是访问到它们, 这就是Linux的**VFS**要做的事情了. 现在让我们试一试**VFS**的魔力!  
理论上, 你可以任意指定一个路径, 把它挂在到虚拟文件系统的任意位置上, 只要指明type是proc, sysfs, udev, tmpfs就可以了. 例如, 你的/dev又何必必须是/dev?

```bash
root@ubuntu:# mkdir /tmp/dev
root@ubuntu:# mount -t devtmpfs devtmpfs /tmp/dev
root@ubuntu:# ls /tmp/dev
```

你会发现, 这里面的内容和刚刚的/dev是一样的. 其实它们就是同一个东西, 只不过挂载到了不同的位置上.  

这就是VFS的灵活性所在!  

## 思路, 重在思路
Linux这样的设计其实是一个非常棒的设计. 这允许你快速对整个系统进行调试, 而不需要一大把复杂的工具.  
仅仅是一个`echo`和`cat`, 就几乎可以在整个系统中快速操作任何东西.  
这样的设计, 是之前在Windows平台上几乎想都不敢想的. 它可以实现的东西有很多, 例如用户态动态修改硬件的参数设置, 只需要文件打开写入即可, 解决简单的应用也要支持库堆成山的问题.  

玩法很多, 唯独你自己去尝试了, 才会发现原来Linux系统的接口是那么的易用.  

在将来的某一天, 你在设计内核模块时, 不妨也使用seqfile, debugfs, procfs等接口, 把你经常需要修改或者查看的参数暴露出来.  
这样, 可比自己设计一个ioctl通讯要简单太多了.  

此外, 如果你加载了一些内核模块驱动, 而且这个驱动不能正常工作, 除了查看`dmesg`寻找报错信息之外, 如果驱动还提供了这样的一些接口, 也不妨直接对这些接口发起请求, 看看驱动的内部状态.  
