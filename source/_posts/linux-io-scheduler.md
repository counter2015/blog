# Linux I/O调度器备忘

title: Linux I/O调度器备忘
date: 2019-12-16 23:31:25
tags: [Linux, 备忘] 
categories: [技术]

------





## 前言

当程序写好了，需要对它的性能做一个压测，系统的瓶颈，简要来说可以分成

- CPU
- 磁盘 I/O
- 网络 I/O

（这里没有考虑分布式情况下的高可用等问题）

本文对于磁盘I/O方面，就Linux I/O调度器方面的知识做了点简单整理，作为备忘。



## Linux I/O 系统简介

这一部分可以参见[IBM文档](https://www.ibm.com/developerworks/cn/linux/l-lo-io-scheduler-optimize-performance/index.html)[^1]，这里就不赘述了。

![]( https://counter2015.com/picture/linux-io-scheduler-1.png )



▲ 图片来源：IBM 《调整 Linux I/O 调度器优化系统性能》



## 预备知识-磁盘

### 机械硬盘(HDD)

机械硬盘以扇区为磁盘寻址的单位，读取数据时，需要先把磁臂移动到指定磁道，然后将扇区旋转到磁头下。

这里有个经典的调度算法就是[电梯(SCAN)调度算法](https://en.wikipedia.org/wiki/Elevator_algorithm)

### 固态硬盘(SSD)

 固态用电压从晶体里储存读取数据，这方面来说，不会像HDD一样受到旋转速度的限制，不存在机械操作。



引用一个不准(tong)确(su)的例子

> 我给你举个特别不恰当的比方吧，因为我实在是想不出更恰当的了。
>
> 抗日战争片看过吧？摇把子电话见过吧？这种电话是没有拨号盘的，那怎么用呢？
>
> 首先你要摇电话上的把手，然后接线员会接你的电话，你跟接线员说我要接司令部，接线员就找到司令部的电话线接口，然后把你的电话线插！在！上！面！
>
> 
>
> 这就相当于是机械寻址。
>
> 
>
> 影响接线员速度的因素有：接线员手速（硬盘转速），接线员的数量（磁头数）。可以想象的是电话号码越多找目标电话号码越慢。因为需要一个一个找啊。
>
> 
>
> 而你现在打电话只需要拨个号，程控交换机就会帮你接好线路。这个寻找目标电话号码的任务你就可以认为是电子寻址。这种电子寻址几乎不会因为号码空间的大小而产生明显的不同。
>
>  关于固态硬盘的寻址原理和寻址速度的一个疑问？ - Zign的回答 - 知乎 https://www.zhihu.com/question/58752580/answer/159003701 



可以通过以下命令区分HDD和SSD

```shell
# 通过lsblk 查看, ROTA意思为rotational device, 0为假，1为真
$ lsblk -d -o name,rota
NAME ROTA
sda     0
sdb     1

# 通过sys信息查看
$ cat /sys/block/sda/queue/rotational
0
$ cat /sys/block/sdb/queue/rotational
1

# 也可用用smartctl查看详细信息
$ sudo smartctl -a /dev/sda
```

简单粗暴地来说，旋转的是机械硬盘，非旋转的是固态硬盘。



## I/O调度器

下面以**CentOS**为例

```shell
$ hostnamectl
   Static hostname: ***
         Icon name: ***
           Chassis: ***
        Machine ID: ***
           Boot ID: ***
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-957.el7.x86_64
      Architecture: x86-64

# 查看当前可用的调度器
$ dmesg | grep -i scheduler
[    1.374555] io scheduler noop registered
[    1.374558] io scheduler deadline registered (default)
[    1.374594] io scheduler cfq registered
[    1.374597] io scheduler mq-deadline registered
[    1.374600] io scheduler kyber registered
```



可以看到，现在可用的调度器有5个

- NOOP

  全称为No Operation,中文名称电梯调度器，I/O请求首先进入一个FIFO接受队列，然后合并后，从调度队列大致按照到达的先后依次进行操作，由于不需要对请求再次排序，这个调度方式和固态硬盘配合最好。

- Deadline(默认使用)

  中文名称为截止时间调度器，它会把请求划分成读操作和写操作，它将读写操作分开处理。使用了四个队列，两个分别按处理读写，按初始扇区排序，另外两个队列为读/写请求的，按最后截止时间排序的队列。默认情况下，读取请求的到期时间是500ms，而写入请求的到期时间是5s。它能避免有些请求长时间不被处理的情况。

- CFQ

  CFQ全称Completely Fair Scheduler（<del>这是怎么缩写的</del>），中文名称完全公平调度器，它的主要目标是在所有进程间公平地分配I/O带宽，为此，会使用大量(默认值为64)已经排序的队列来分别存储来自不同进程的请求。I/O请求到达时，通过哈希函数对PID求值放入到相应标识的队列末尾，因此同一进程的请求总是会在同一队列中被依次处理。

- MQ-Deadline

  适用于多线程版本的Deadline调度器， 该调度程序适用于大多数用例，但特别是那些读操作比写操作更频繁发生的用例。 

- Kyber

  Kyber适用于快速多队列任务，它把I/O请求分为两个队列，一个用于同步请求，一个用于异步请求。它通过限制发送到调度队列的I/O操作，确保较高优先级的请求能快速被完成，可以通过设置延迟来调整等待时间，默认读操作是2ms,写操作是10ms，把这两个值调小，可以保证较低的延迟，但也会由于缺少合并，而影响到吞吐量。这方面资料很少，有兴趣的猛士可以看下[源码](https://github.com/torvalds/linux/blob/master/block/kyber-iosched.c)。



对于不同的场景下，[RedHat](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-storage_and_file_systems-configuration_tools)有提供参考的配置


| Use case                                                     | Disk scheduler recommendation                                |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| Traditional HDD with a SCSI interface                        | Use `mq-deadline` or `bfq`.                                  |
| High-performance SSD or a CPU-bound system with fast storage | Use `none`, especially when running enterprise applications. Alternatively, use `kyber`. |
| Desktop or interactive tasks                                 | Use `bfq`.                                                   |
| Virtual guest                                                | Use `mq-deadline`. With a multi-queue host bus adapter (HBA), use `none`. |

渣翻下

| 使用场景                          | 推荐的磁盘调度方式                      |
| --------------------------------- | --------------------------------------- |
| 机械硬盘                          | `mq-deadline`                           |
| 固态硬盘 \| CPU绑定的快速存储设备 | `none`                                  |
| PC程序 \| 交互任务                | `bfq`                                   |
| 虚拟化                            | `mq-deadline`, 如果有FC HBA卡，用`none` |

具体的配置需要根据自己程序的读写特点分别测试，这个表只是个参考。

## 调度器的配置与切换

### I/O调度器的选择

不同版本启用的默认调度器不同，主要分为两类

支持多队列的：`mq-deadline`, `kyber`, `noop`

不支持多队列的：`noop`, `cfq`, `deadline`

这两类配置无法同时启用

同类配置中，I/O调度器的切换不需要重启机器

```shell
# 修改I/O调度器
$ echo "cfq" | sudo tee /sys/block/sda/queue/scheduler
```

假如要切换到多队列调度器，操作稍微复杂点

```shell
# 修改grub配置文件
$ sudo vim /etc/default/grub
```

在配置项`GRUB_CMDLINE_LINUX`的引号内，末尾添加`scsi_mod.use_blk_mq=1`

添加完成后，需要重新生成`grub2.cfg`

不同的设备类型，接下来输出的配置文件目录也不同，可以通过如下命令来确认设备类型

```shell
$ test -d /sys/firmware/efi && echo "UEFI" || echo "Legacy"
```

以`Legacy`设备为例(`UEFI`的配置可参见[此处](https://abelsu7.top/2019/02/18/centos7-grub2/))

```shell
$ grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.el7.x86_64.img
Found linux image: /boot/vmlinuz-0-rescue-ffbf5b85fb92405ab2466889fe1358d2
Found initrd image: /boot/initramfs-0-rescue-ffbf5b85fb92405ab2466889fe1358d2.img
done
```

设置完成后，重启机器即可生效

```shell
$ sudo reboot
$ cat /sys/block/sda/queue/scheduler
[mq-deadline] kyber none
```



## 参考资料

1. [Ubuntu Wiki: IOSchedulers](https://wiki.ubuntu.com/Kernel/Reference/IOSchedulers)

2. [Improving Linux System Performance with I/O Scheduler Tuning](https://blog.codeship.com/linux-io-scheduler-tuning/)
3. [Two new block I/O schedulers for 4.12](https://lwn.net/Articles/720675/)
4. [Improving the performance of the BFQ I/O scheduler](https://lwn.net/Articles/784267/)
5. [StackExchange: how to know if a disk is an ssd or an hdd](https://unix.stackexchange.com/questions/65595/how-to-know-if-a-disk-is-an-ssd-or-an-hdd) 
6. [IBM: Linux 调度器内幕](https://www.ibm.com/developerworks/cn/linux/l-scheduler/index.html)
7. [Manjaro Linux优化设置分享](https://zhuanlan.zhihu.com/p/48963253)
8.  [硬盘的原理以及SQL Server如何利用硬盘原理减少IO](https://www.cnblogs.com/CareySon/archive/2012/08/20/2647017.html)


[^1]: 2022年，这篇文章的链接地址已经失效







 





