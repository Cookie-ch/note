# Linux文件系统

内核kernel：
	1.对CPU进行调度
	2.内存的分配管理
	3.文件系统的管理
	4.进程管理
	5.其他硬件管理（网络、显卡、声卡等）

内存分为用户空间和内核空间，拿mysql来举例，mysql本身是在用户空间，用户要对数据进行读写，mysql就会调用内核空间的某些方法（对吸盘进行读写）

用户空间的进程不能随意访问内核空间，内核空间会有对应接口，通过系统调用api来实现相应方法

`fdisk`查看硬盘分区表，`df`查看分区使用情况，`du`查看文件占用空间情况

## 磁盘

磁盘的结构：
磁道track：63个扇区
柱面cylinder：不同盘片上的相同磁盘组成
扇区sector：512字节，最小物理存储数据的单元
磁头header

扇区：系统访问硬盘最小的单位 

磁盘接口：

1. **IDE (Integrated Drive Electronics):**
   - **描述：** IDE是一种传统的磁盘接口标准，也称为ATA（AT Attachment），它用于连接计算机主板和硬盘驱动器。
   - **特点：** IDE接口采用平行数据传输方式，适用于早期的个人计算机。IDE接口有时被称为PATA（Parallel ATA），以区别于SATA。
2. **SATA (Serial ATA):**
   - **描述：** SATA是一种现代的串行磁盘接口标准，用于连接计算机主板和硬盘驱动器、光盘驱动器等设备。
   - **特点：** SATA接口使用串行数据传输方式，相对于IDE有更高的数据传输速度。它是目前个人计算机中最常见的硬盘接口之一。
3. **NVMe (Non-Volatile Memory Express):**
   - **描述：** NVMe是一种为固态硬盘设计的高性能接口和协议标准。它通过PCI Express总线连接到计算机主板。
   - **特点：** NVMe接口相比于传统的SATA接口有更高的数据传输速度和更低的延迟。NVMe SSDs通常用于需要极高性能的应用，如游戏、数据中心等。
4. **SCSI (Small Computer System Interface):**
   - **描述：** SCSI是一种通用的、灵活的磁盘接口和设备通信协议，用于连接各种外部设备，包括硬盘、光盘驱动器、打印机等。
   - **特点：** SCSI接口支持多设备连接，有多种不同的实现，包括SCSI-1、SCSI-2、UltraSCSI等。它通常用于服务器和工作站等环境。

## 磁盘速度

IOPS：input output per second磁盘每秒读写次数

机械盘存取速度一般比较慢，100M/s/~150M/s
ssd固态盘是电子芯片，读取速度较快，500M/s/~5G/s

测试磁盘速度，产生约1G数据需要多少时间

 ```
[root@localhost ~]# dd if=/dev/zero of=/root/chen.dd count=1000 bs=1M
记录了1000+0 的读入
记录了1000+0 的写出
1048576000字节(1.0 GB)已复制，5.57267 秒，188 MB/秒

 ```

`if`输入文件，`/dev/zero` 是一个特殊的设备文件，读取它会生成连续的null字节，`of`输出文件，每次写1M大小的数据，写1000次

- "记录了1000+0 的读入" 表示已经从输入源读取了1000个块。
- "记录了1000+0 的写出" 表示已经向输出文件写入了1000个块。
- "1048576000字节(1.0 GB)已复制" 表示总共复制了1GB的数据。
- "5.57267 秒" 表示整个操作花费了5.57267秒。

## 磁盘性能查看

`dstat`能够提供有关各种系统资源的实时信息，包括CPU使用率、磁盘活动、网络活动、系统内存使用等

```
# 显示与分区sdb1有关的磁盘活动信息
dstat -D sdb1
```

`glances`提供系统性能的交互式全面视图，包括 CPU 使用情况、内存利用率、磁盘 I/O、网络活动等详细信息

`iostat`用于报告系统上设备和分区的输入/输出 (I/O) 统计信息，安装sysstat就可以使用iostat命令了

## raid

RAID（Redundant Array of Independent Disks）磁盘阵列，是一种通过将多个硬盘组合起来形成一个逻辑存储单元的技术。RAID磁盘阵列的目标是提高数据的性能、容错性和可用性

RAID有硬件RAID和软件RAID
硬件RAID速度快，性能好，支持热插拔，但需要专门的RAID磁盘阵列卡，且价格贵
软件RAID使用mdadm软件仿真磁盘阵列功能，不需要专门硬件，设备文件标识是/dev/md0

RAID划分等级：

* **RAID 0：**条带卷striping。 数据被分成块并分布在多个硬盘上（至少两个）。同时往多个磁盘上写数据，读写速度快，供了很好的性能增益，但如果一个硬盘失效，所有数据都将丢失

* **RAID 1：** 镜像卷mirroring。数据被镜像在两个硬盘上，提供了冗余。如果一个硬盘失效，数据仍然可用，提高可用性

* **RAID 5：**条带+分布校验。 数据和奇偶校验信息交织存储在多个硬盘上（至少三个）。提供了性能和冗余。如果一个硬盘失效，数据可以通过奇偶校验信息进行恢复

* **RAID 6：** 类似于RAID 5，但提供了更多的冗余，它使用两个奇偶校验信息块。可以容忍两个硬盘的故障。RAID 6至少需要四个硬盘

* **RAID 10（也称为RAID 1+0）：**条带+镜像，结合了RAID 1和RAID 0。数据被镜像，并且这些镜像被组合成一个RAID 0阵列。提供了高性能和冗余，需要至少四个硬盘

硬件raid设置一般通过主板上的raid控制器来配置，具体是在主机启动过程中按下特定键比如ctrl+s进入raid设置界面

软件raid设置是通过mdadm管理工具来进行设置，过程为创建raid数组、格式化、挂载

```
# 创建raid数组
mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb
# 格式化
mkfs.ext4 /dev/md0
# 挂载
mkdir /mnt/raid
mount /dev/md0 /mnt/raid
# 修改/etc/fstab文件实现自动挂载
/dev/md0   /mnt/raid   ext4   defaults   0   0

```



## LVM

LVM（Logical Volume Manager）逻辑卷管理，底层是多块磁盘拼凑起来的，跟raid很像。LVM最大特点是便于动态调整磁盘容量

> /boot分区用于存放引导文件，不能应用LVM机制

LVM机制的基本概念：

* **物理卷（Physical Volume，PV）：** 这是实际硬盘驱动器或用fdisk等工具建立的普通分区，包括许多默认4M大小的PE。可以将一个或多个物理卷组合成一个卷组

* **卷组（Volume Group，VG）：** 由一个或多个物理卷组成，是LVM中的逻辑存储池。卷组是LVM的核心，逻辑卷从卷组分配空间

* **逻辑卷（Logical Volume，LV）：** 逻辑卷相当于传统硬盘上的分区，但更加灵活。它是从卷组中分配的逻辑块设备，可以被格式化并用作文件系统
* **物理区域（Physical Extent，PE）：** 物理卷被分成物理区域，通常是4MB或8MB的大小。这些物理区域是LVM中分配和管理存储空间的最小单位

### LVM的创建流程

1. **创建物理卷（PV）：** 将硬盘或分区添加到LVM中作为物理卷

   ```
   [root@localhost tt]# pvcreate /dev/sdc
     Physical volume "/dev/sdc" successfully created.
   [root@localhost tt]# pvscan
     PV /dev/sda2   VG centos          lvm2 [<19.00 GiB / 0    free]
     PV /dev/sdc                       lvm2 [50.00 GiB]
     Total: 2 [<69.00 GiB] / in use: 1 [<19.00 GiB] / in no VG: 1 [50.00 GiB]
   
   ```

   

2. **创建卷组（VG）：** 将一个或多个物理卷组合成卷组

   ```
   [root@localhost tt]# vgcreate mail_store /dev/sdc
     Volume group "mail_store" successfully created
   [root@localhost tt]# vgscan
     Reading volume groups from cache.
     Found volume group "centos" using metadata type lvm2
     Found volume group "mail_store" using metadata type lvm2
   
   ```

   

3. **创建逻辑卷（LV）：** 从卷组中分配逻辑卷

   ```
   [root@localhost tt]# lvcreate -L 40G -n chen_mail mail_store
     Logical volume "chen_mail" created.
   [root@localhost tt]# lvscan
     ACTIVE            '/dev/centos/swap' [2.00 GiB] inherit
     ACTIVE            '/dev/centos/root' [<17.00 GiB] inherit
     ACTIVE            '/dev/mail_store/chen_mail' [40.00 GiB] inherit
   
   ```

   

4. **格式化逻辑卷：** 类似于在传统硬盘上创建文件系统

   ```
   [root@localhost tt]# mkfs.xfs /dev/mail_store/chen_mail 
   meta-data=/dev/mail_store/chen_mail isize=512    agcount=4, agsize=2621440 blks
            =                       sectsz=512   attr=2, projid32bit=1
            =                       crc=1        finobt=0, sparse=0
   data     =                       bsize=4096   blocks=10485760, imaxpct=25
            =                       sunit=0      swidth=0 blks
   naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
   log      =internal log           bsize=4096   blocks=5120, version=2
            =                       sectsz=512   sunit=0 blks, lazy-count=1
   realtime =none                   extsz=4096   blocks=0, rtextents=0
   
   ```

   

5. **挂载逻辑卷：** 将逻辑卷挂载到文件系统的特定目录

   ```
   [root@localhost tt]# mount /dev/mail_store/chen_mail /test
   [root@localhost tt]# vim /etc/fstab 
   [root@localhost tt]# tail -n 1 /etc/fstab 
   /dev/mail_store/chen_mail /test xfs defaults 0 0
   
   ```

命令：

| 功能        | 物理卷    | 卷组      | 逻辑卷    |
| ----------- | --------- | --------- | --------- |
| scan扫描    | pvscan    | vgscan    | lvscan    |
| create建立  | pvcreate  | vgcreate  | lvcreate  |
| display显示 | pvdisplay | vgdisplay | lvdisplay |
| remove删除  | pvremove  | vgremove  | lvremove  |
| extend扩展  |           | vgextend  | lvextend  |
| reduce减少  |           | vgreduce  | lvresize  |

### 扩容

```
# 扩容
lvextend -L +5G /dev/mail_store/chen_mail
# 让Linux内核重新识别lv的大小
xfs_growfs /dev/mail_store/chen_mail
```





## 新加磁盘的完整步骤

物理连接 --> 分区 --> 格式化 --> 挂载 --> 更新/etc/fstab文件实现永久挂载

### 磁盘分区

主分区primary用来安装操作系统、存放数据，可以引导操作系统，基本磁盘上可以建立一到四个主分区（1-4）

剩余空间可以作为扩展分区，扩展分区只能有一个，用来突破四个主分区的限制，不能存放数据，要划分成逻辑分区来存储。扩展分区可以划分为多个逻辑分区（从5开始）。扩展分区会占用一个主分区位置（一共最多四个分区）

**为什么要进行分区？**
可以把不同资料分别放入不同分区中管理，降低风险
大硬盘搜索范围大，效率低
/home、/var、/usr、/local经常是单独分区，因为经常操作，容易产生碎片

Linux中，`/dev`目录用来存放设备文件（device）

#### 磁盘文件命名

Linux磁盘分区表有很多不同的标准：

* GPT：使用全局唯一标识符GUID来表示分区和设备
* IRIX：运行在服务器和工作站上，逐渐淡出市场
* DOS：磁盘操作系统
* Sun：运行在服务器和工作站上
* LVM：进行磁盘管理的逻辑卷管理系统，允许对磁盘进行动态管理

默认使用DOS

- IDE硬盘的设备文件通常以 `hdx` 表示，其中 `x` 是字母，例如 `/dev/hda`, `/dev/hdb`。
- SATA硬盘的设备文件通常以 `sdx` 表示，其中 `x` 是字母

- SCSI和SAS硬盘的设备文件通常以 `sdxy` 表示，其中 `x` 是字母，`y` 是分区编号，例如 `/dev/sda1`, `/dev/sdb2`。

- NVMe硬盘的设备文件通常以 `nvme0n1p1` 表示，其中 `0` 是控制器编号，`n1` 是设备编号，`p1` 是分区编号。

- USB闪存驱动器的设备文件通常以 `sdx` 表示，其中 `x` 是字母，例如 `/dev/sdc`。

- 软驱的设备文件通常以 `fdx` 表示，其中 `x` 是字母，例如 `/dev/fd0`。
- RAID设备的设备文件通常以 `mdx` 表示，其中 `x` 是数字，例如 `/dev/md0`。

- RAM磁盘的设备文件通常以 `ramx` 或 `rdx` 表示，其中 `x` 是数字，例如 `/dev/ram0` 或 `/dev/rd0`。

#### fdisk

`fdisk -l`查看所有磁盘以及卷的信息

![image-20231125154432027](https://github.com/chen03210/note/assets/79464052/d8611a9a-ff96-4ba0-9c53-5453649e27ab)


这里sda1的Boot下有一个*号，说明sda1是启动盘

`fdisk /dev/sdb`对磁盘sdb进行分区

将磁盘sdb分出一个大小为20G的主分区sdb1

```
[root@localhost ~]# fdisk /dev/sdb
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。

Device does not contain a recognized partition table
使用磁盘标识符 0x11b298d9 创建新的 DOS 磁盘标签。

命令(输入 m 获取帮助)：p

磁盘 /dev/sdb：107.4 GB, 107374182400 字节，209715200 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x11b298d9

   设备 Boot      Start         End      Blocks   Id  System

命令(输入 m 获取帮助)：n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): 
Using default response p
分区号 (1-4，默认 1)：
起始 扇区 (2048-209715199，默认为 2048)：
将使用默认值 2048
Last 扇区, +扇区 or +size{K,M,G} (2048-209715199，默认为 209715199)：+20G
分区 1 已设置为 Linux 类型，大小设为 20 GiB

命令(输入 m 获取帮助)：p

磁盘 /dev/sdb：107.4 GB, 107374182400 字节，209715200 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x11b298d9

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    41945087    20971520   83  Linux

命令(输入 m 获取帮助)：w
The partition table has been altered!

Calling ioctl() to re-read partition table.
正在同步磁盘。

```

将磁盘sdb分出一个扩展分区

```
[root@localhost ~]# fdisk /dev/sdb
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。


命令(输入 m 获取帮助)：p

磁盘 /dev/sdb：107.4 GB, 107374182400 字节，209715200 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x11b298d9

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    41945087    20971520   83  Linux

命令(输入 m 获取帮助)：n
Partition type:
   p   primary (1 primary, 0 extended, 3 free)
   e   extended
Select (default p): e
分区号 (2-4，默认 2)：
起始 扇区 (41945088-209715199，默认为 41945088)：
将使用默认值 41945088
Last 扇区, +扇区 or +size{K,M,G} (41945088-209715199，默认为 209715199)：
将使用默认值 209715199
分区 2 已设置为 Extended 类型，大小设为 80 GiB

命令(输入 m 获取帮助)：p

磁盘 /dev/sdb：107.4 GB, 107374182400 字节，209715200 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x11b298d9

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    41945087    20971520   83  Linux
/dev/sdb2        41945088   209715199    83885056    5  Extended

命令(输入 m 获取帮助)：w
The partition table has been altered!

Calling ioctl() to re-read partition table.
正在同步磁盘。

```

将磁盘sdb分出一个逻辑分区

```
[root@localhost ~]# fdisk /dev/sdb
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。


命令(输入 m 获取帮助)：n
Partition type:
   p   primary (1 primary, 1 extended, 2 free)
   l   logical (numbered from 5)
Select (default p): l
添加逻辑分区 5
起始 扇区 (41947136-209715199，默认为 41947136)：
将使用默认值 41947136
Last 扇区, +扇区 or +size{K,M,G} (41947136-209715199，默认为 209715199)：+20G
分区 5 已设置为 Linux 类型，大小设为 20 GiB

命令(输入 m 获取帮助)：p

磁盘 /dev/sdb：107.4 GB, 107374182400 字节，209715200 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x11b298d9

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    41945087    20971520   83  Linux
/dev/sdb2        41945088   209715199    83885056    5  Extended
/dev/sdb5        41947136    83890175    20971520   83  Linux

命令(输入 m 获取帮助)：w
The partition table has been altered!

Calling ioctl() to re-read partition table.
正在同步磁盘。

```



![image-20231125152628265](https://github.com/chen03210/note/assets/79464052/17437d20-23fb-4a43-ab34-40268eb71233)


但DOS类型的分区表最多只支持2TiB

往虚拟机添加一个8000G的硬盘MBR，使用`fdisk`进行分区，会有警告信息，建议使用GPT类型的分区表

![image-20231125195240012](https://github.com/chen03210/note/assets/79464052/ca8298ef-4240-414b-b63d-a3cd6f2f784b)


#### parted

parted可以对大容量的磁盘进行分区

```
[root@localhost ~]# parted /dev/sdd
GNU Parted 3.1
使用 /dev/sdd
Welcome to GNU Parted! Type 'help' to view a list of commands.
# 创建GPT分区表
(parted) mklabel gpt                                            
# 创建分区
(parted) mkpart chen 1 10000
# 列出分区
(parted) print                                                            
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdd: 8590GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  标志
 1      1049kB  10.0GB  9999MB               chen

(parted) mkpart chen2 10001 30000
(parted) print                                                            
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdd: 8590GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name   标志
 1      1049kB  10.0GB  9999MB               chen
 2      10.0GB  30.0GB  20.0GB               chen2

(parted) quit                                                             
信息: You may need to update /etc/fstab.

[root@localhost ~]# parted /dev/sdd print
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdd: 8590GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name   标志
 1      1049kB  10.0GB  9999MB               chen
 2      10.0GB  30.0GB  20.0GB               chen2

```

`fdisk`较`parted`更安全，分区的时候先缓存在内存里，输入w保存后才会去修改分区表，若不输入w而直接输入q可以不保存。`parted`则是直接生效

#### MBR

MBR（Master Boot Record）主引导记录，每块磁盘中都有MBR

磁盘的0柱面、0磁头、0扇区成为主引导扇区

MBR大小为512字节，分为三个部分：主引导程序446字节、硬盘分区表DPT64字节、分区结束标记2字节

DPT磁盘分区表的四个主分区分别用16字节描述，扩展分区也要占用16字节的主分区空间

下面模拟破坏sdb的MBR，看看还能不能用fdisk命令显示出分区信息

```
# sdb的分区信息
[root@localhost ~]# fdisk -l /dev/sdb

磁盘 /dev/sdb：107.4 GB, 107374182400 字节，209715200 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x11b298d9

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    41945087    20971520   83  Linux
/dev/sdb2        41945088   209715199    83885056    5  Extended
/dev/sdb5        41947136    83890175    20971520   83  Linux

# 建立sdb的MBR备份
[root@localhost ~]# mkdir /mbr
[root@localhost ~]# dd if=/dev/sdb of=/mbr/sdb.mbr bs=512 count=1
记录了1+0 的读入
记录了1+0 的写出
512字节(512 B)已复制，0.000424829 秒，1.2 MB/秒

# 破坏MBR
[root@localhost ~]# dd if=/dev/zero of=/dev/sdb bs=512 count=1
记录了1+0 的读入
记录了1+0 的写出
512字节(512 B)已复制，0.000965753 秒，530 kB/秒

# 查看sdb的分区信息
[root@localhost ~]# fdisk -l /dev/sdb

磁盘 /dev/sdb：107.4 GB, 107374182400 字节，209715200 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节

```

这里看到已经显示不出分区信息了，我们把备份还原回去

```
[root@localhost ~]# dd if=/mbr/sdb.mbr of=/dev/sdb bs=512 count=1
记录了1+0 的读入
记录了1+0 的写出
512字节(512 B)已复制，0.00154552 秒，331 kB/秒
[root@localhost ~]# fdisk -l /dev/sdb

磁盘 /dev/sdb：107.4 GB, 107374182400 字节，209715200 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x11b298d9

   设备 Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    41945087    20971520   83  Linux
/dev/sdb2        41945088   209715199    83885056    5  Extended
/dev/sdb5        41947136    83890175    20971520   83  Linux

```

现在能够显示分区信息了

#### 经典分区方案

默认分为三个部分：

* /：根分区

* /boot：boot分区，启动Linux启动所需要的文件存放在这个分区里，默认大小为1G，一般500M就够了

* swap：swap分区，做虚拟内存使用的，当真实内存不足的时候，充当内存使用

  ```
  [root@localhost ~]# cat /proc/sys/vm/swappiness 
  30
  表示当物理内存只剩30%的时候就开始使用swap分区
  ```

  * 扩展swap分区

    ```
    [root@localhost ~]# mkswap /dev/sdb5
    正在设置交换空间版本 1，大小 = 20971516 KiB
    无标签，UUID=60186739-d551-4d57-b6eb-658e557010cd
    [root@localhost ~]# cat /proc/swaps 
    Filename				Type		Size	Used	Priority
    /dev/dm-1                               partition	2097148	0	-2
    [root@localhost ~]# swapon /dev/sdb5
    [root@localhost ~]# cat /proc/swaps 
    Filename				Type		Size	Used	Priority
    /dev/dm-1                               partition	2097148	0	-2
    /dev/sdb5                               partition	20971516	0	-3
    
    ```

  * 减少swap分区

    ```
    [root@localhost ~]# swapoff /dev/sdb5
    [root@localhost ~]# cat /proc/swaps 
    Filename				Type		Size	Used	Priority
    /dev/dm-1                               partition	2097148	0	-2
    
    ```

    

### 分区格式化

格式化就是将分区的数据清除，然后划分出新的块（数据的逻辑存储单位）

格式化会产生两个区域：inode zone和block zone，inode zone存放文件属性，索引节点区，block zone存放数据data

inode区一个块占256字节；block区一个块占4096字节，也就是8个扇区

格式化命令为`mkfs`，具体要对不同文件系统类型的分区使用不同的命令

![image-20231125162732248-17009221891534](https://github.com/chen03210/note/assets/79464052/ca21e6a0-c6b6-484b-a96c-c52cf25807fb)


> df查看磁盘分区的挂载使用情况

对sdb1格式化：`mkfs.xfs /dev/sdb1`

![image-20231125162744595](https://github.com/chen03210/note/assets/79464052/c6a7c9ea-edc7-4fef-b230-daf0d6a849d5)


分区格式化后会产生：

* inode table：inode空间
* data block：数据区
* inode map：inode映射表，记录哪些inode使用了，哪些没有使用。账簿记录了inode区里的inode的使用情况
* block map：block映射表，记录哪些block使用了。账簿记录了block区里的inode使用情况
* superblock：超级块，记录此file system的整体信息，包括inode/block的总量、使用量、剩余量，以及文件系统的格式

### 挂载分区

`mount`查看所有的挂载信息

将sdb1分区挂载到/chen目录下：`mount /dev/sdb1 /chen`

修改`/etc/fstab`文件（开机时会运行这个文件），添加一行，实现开机自动挂载

```
# 分区	挂载点	文件系统类型	挂载后的分区选项
# 第一个0：dump备份（0表示不做备份，1表示每天进行备份）
# 第二个0：以fsck检测文件系统（0表示不检测，1表示检测，2表示1级别检测完之后再进行检测）
/dev/sdb1 /chen xfs defaults 0 0
```

如果/dev/fstab文件内容写错，会导致系统启动失败，ssh服务不会开启，远程是连接不上的

取消挂载`umount /chen`

如果取消挂载显示target is busy信息，这个时候使用`lsof /chen`查看/chen这个文件夹被哪个进程占用，kill掉这个进程之后就可以取消挂载了

> `lsof -p 1100`查看进程号为1100的进程打开了哪些文件



## 文件系统

Linux支持多种文件系统 ：

* ext2：Linux基本文件系统
* ext3：ext2的增强版本，Linux默认文件系统
* ext4：第四个版本
* swap：交换文件系统
* xfs：高性能、大容量文件系统
* nfs：网络文件系统，适合Linux、Unix机器间共享
* smbfs：适合Linux、Unix、Windows机器间共享
* tmpfs：临时文件系统，消耗内存空间
* iso9960：光盘文件系统，挂载的目录为只读

Linux内核采用虚拟文件系统层（VFS），不同文件系统通过VFS交流

### 重要参数

`dumpe2fs /dev/sda1|more`查看ext4文件元数据（描述文件系统的数据），`xfs_info`查看xfs文件系统

* superblock：超级块，记录此file system的整体信息，包括inode/block的总量、剩余量，以及文件系统的格式
* inode：记录文件的属性，一个文件占用一个inode，同时记录此文件的数据所在的block号
* block：实际记录文件的内容，若文件太大，会占用多个block

### 目录项

目录项包含文件名、文件的inode号和文件类型

```
# 第一列如67就是inode号
[root@localhost chen]# ll -i
总用量 0
      67 drwxr-xr-x. 2 root root 6 11月 25 21:55 haha
33574976 drwxr-xr-x. 2 root root 6 11月 25 21:55 tt
16777280 drwxr-xr-x. 2 root root 6 11月 25 21:55 xixi
```

> 执行命令`cd /chen/tt`，首先通过目录项根据文件名找到inode编号，然后去inode区去寻找chen目录对应的inode编号所在的块，读取信息，编号所在的块会有一个指针pointer指向block区里面的块，根据这个pointer读取相应的信息，递归找到tt这个文件

当使用`rm`删除文件时，文件系统中的目录项会被删除，但inode映射表和block映射表里的队友标志会被标记为可用，表示这些块可以被新文件或其他数据使用，但实际inode区和block区没有被释放，所以只要数据块没有被新数据覆盖，被删除的数据是可以通过修复软件恢复的，文件系统会有日志

有时候一个目录下不能创建文件，但是查看磁盘发现占用比并没有满，有可能是因为inode空间耗光了，删除目录下的其他文件就可以创建新文件了

### fsck

`fsck`命令，file system check用于诊断修复文件系统，可以修复xfs文件或ext4文件，xfs文件修复也有专门的命令`xfs_repair`

当出现非正常关机、突然断点、设备读写失误、文件系统的超级块信息被破坏时可以修复

```
dd if=/dev/zero of=/dev/sdb1 bs=512 count=4
ls
cd
umount /chen
mount /dev/sdb1 /chen
fsck /dev/sdb1 -y
```

### 软连接，硬链接

软链接是一个指向目标文件或目录的路径的文本字符串，会产生新的inode和block、新的目录项，block里存放的是链接的文件名

硬链接会跟原文件有一样的inode，但会有不同的目录项

```
# 创建软连接
ln -s test.txt test_soft.txt
# 创建硬链接
ln test.txt test_hard.txt
```

![image-20231126101207283](https://github.com/chen03210/note/assets/79464052/1016ec9a-0985-484c-bd15-b3d1a3cf5a2f)


软链接，删除原文件，链接文件不可用；硬链接，删除原文件，链接文件可以继续使用，但内容为空

![image-20231126101221410](https://github.com/chen03210/note/assets/79464052/336bba94-89a8-4094-88a4-e7a2a503b443)
