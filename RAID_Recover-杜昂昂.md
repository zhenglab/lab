<!--
title: 尝试恢复挂载服务器硬盘的 RAID 5
author: Du, Ang
date: July 28th, 2017
-->

# 尝试恢复挂载服务器硬盘的 RAID 5

## 写在前面
前段时间实验室的 DIY 服务器重装了系统 Ubuntu 16.04，但重装后就不知道该怎么挂载原来的硬盘了。经过了解，该服务器上共有两块 SSD，一块用来装系统，一块做硬盘缓存；三块 4T 大小的硬盘，做了软 [RAID 5](https://zh.wikipedia.org/wiki/RAID)。这里仅记录一下恢复挂载该硬盘的过程，不涉及 RAID 和缓存的知识。

## 恢复过程
### 1. 查看硬盘信息
先通过 `sudo fdisk -l | grep sd` 命令查看硬盘信息，内容如下：
```text
Disk /dev/sda: 477 GiB, 512110190592 bytes, 1000215216 sectors
/dev/sda1            2048  127999999 127997952    61G 82 Linux swap / Solaris
/dev/sda2  *    128000000  368001023 240001024 114.5G 83 Linux
/dev/sda3       368003070 1000214527 632211458 301.5G  5 Extended
/dev/sda5       368003072 1000214527 632211456 301.5G 83 Linux
Disk /dev/sdb: 3.7 TiB, 4000787030016 bytes, 7814037168 sectors
Disk /dev/sdc: 3.7 TiB, 4000787030016 bytes, 7814037168 sectors
Disk /dev/sdd: 3.7 TiB, 4000787030016 bytes, 7814037168 sectors
Disk /dev/sde: 477 GiB, 512110190592 bytes, 1000215216 sectors
/dev/sde1        2046 1000214527 1000212482  477G  5 Extended
/dev/sde5        2048 1000214527 1000212480  477G 83 Linux
```
可以看到，`sdb`、`sdc`、`sdd` 就是需要挂载的三块硬盘。

### 2. 尝试直接挂载三块硬盘。
如果是普通的硬盘，可以先在 `/media/username/` 路径下新建一个文件夹 `HD8T`（任意命名），然后用 `sudo mount /dev/sdb /media/username/HD8T/` 进行挂载。

但是现在在用该命令挂载时出现了下面的错误：
```text
mount: unknown filesystem type 'linux_raid_member'
```
这正说明三块硬盘之前是装了 RAID 的，也说明重装系统没有损坏原来的 RAID，数据是可以恢复的。

### 3. 恢复硬盘上的 RAID
经过查阅资料[[2](https://ubuntuforums.org/showthread.php?t=2002217)]，得知 RAID 是可以通过 `mdadm` 创建或恢复的。

如果系统没有安装 `mdadm`，可以先通过 `sudo apt update && sudo apt install mdadm` 进行安装。安装好 `mdadm` 后，执行 `sudo mdadm --assemble --scan` 命令，它会自动检测并安装所有的 md 硬盘。输出如下：
```text
mdadm: /dev/md/0 has been started with 3 drives.
```
这样，`mdadm` 已经检测并安装了原来硬盘上的 RAID——`md0`。还可以执行 `sudo mdadm --detail /dev/md0` 命令，查看更详细的信息：
```text
dev/md0:
        Version : 1.2
  Creation Time : Thu Dec 31 15:25:00 2015
     Raid Level : raid5
     Array Size : 7813774336 (7451.80 GiB 8001.30 GB)
  Used Dev Size : 3906887168 (3725.90 GiB 4000.65 GB)
   Raid Devices : 3
  Total Devices : 3
    Persistence : Superblock is persistent

    Update Time : Mon Jul 24 13:57:48 2017
          State : clean
 Active Devices : 3
Working Devices : 3
 Failed Devices : 0
  Spare Devices : 0

         Layout : left-symmetric
     Chunk Size : 512K

           Name : CVBIOUC:0
           UUID : f9ad2c2a:16cf56f5:0956886e:cb512410
         Events : 542

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd
```

### 4. 挂载 `bcache` 文件系统的硬盘
参考[《在Linux下使用RAID（四）：创建RAID 5》](http://blog.csdn.net/wzyzzu/article/details/48500017) 这篇博客，现在可以对 RAID 硬盘 `md0` 进行挂载了。

奇怪的是，利用 `mount` 命令挂载时，出现了 `mount: unknown filesystem type 'bcache'` 的错误。看到 cache 这个词，突然想到老师之前说的用了一块 SSD 作为缓存的事情，所以硬盘的文件系统也不一样了。

查阅资料得知，这种文件系统是由 `bcache` 这个工具得到的。最初的想法是重装系统，腾出之前的那块SSD，用 `bcache` 工具恢复它们原来的样子。但后来老师说，做缓存的那块 SSD 不太稳定，不要做缓存了。所以问题一下子简单了，只有想办法把 `bcache` 文件系统的硬盘挂载上就行了。

参考 Stack Overflow 上的 [*How to revert bcache device to regulare filesystem*](https://stackoverflow.com/questions/22820492/how-to-revert-bcache-device-to-regulare-filesystem) 问题，`bcache` 的文档里给出了一种解决思路：
```text
D) Recovering data without bcache:

If bcache is not available in the kernel, a filesystem on the backing
device is still available at an 8KiB offset. So either via a loopdev
of the backing device created with --offset 8K, or any value defined by
--data-offset when you originally formatted bcache with `make-bcache`.
```
也就是说， `bcache` 只是比原文件系统多了 8KiB 的一个头，“去掉”它就可以得到原来的文件系统了。可以先通过 `sudo losetup -o 8192 /dev/loop0 /dev/your_bcache_backing_dev` 命令将“去掉” 8KiB 头的设备挂载到 `/dev/loop0`，然后再将 `/dev/loop0` 挂载到我们之前定义的 `/media/username/HD8T` 文件夹。或者简化成一步：`sudo mount -o loop,offset=8192 /dev/md0 /media/vision/HD8T`。

此外，需要注意挂载到的目录的权限问题，可以执行 `sudo chown -R username /media/username/` 等命令来获取相关权限。

### 5. 开机自动挂载
参考[[4](https://help.ubuntu.com/community/InstallingANewHardDrive)]，可以通过修改 `/etc/fstab` 文件来实现开机自动挂载硬盘。例如在 `/etc/fstab` 文件的末尾添加以下内容，就是为了在开机时将 `/dev/sdb` 自动挂载到 `/media/mynewdrive` 目录：
```text
/dev/sdb1    /media/mynewdrive   ext4    defaults     0        2
```
但是 Ubuntu 更推荐使用 UUID 来标识设备。所有分区和设备，包括上面的 RAID 分区 `/dev/md0`，都有唯一的 UUID。用 UUID 来标识设备的好处在于它们与磁盘顺序无关。如果你在 BIOS 中改变了你的存储设备顺序，或是重新拔插了存储设备，或是因为一些 BIOS 可能会随机地改变存储设备的顺序，那么用 UUID 来表示将更有效。可以通过 `ls -l /dev/disk/by-uuid` 命令来查看硬盘设备的 UUID。[[5](https://liquidat.wordpress.com/2007/10/15/short-tip-get-uuid-of-hard-disks/)]
在 `/etc/fstab` 文件中使用 UUID 的格式如下：
```text
UUID=d4f1d3fb-6177-4fba-72f4-43189a8b1638    /media/mynewdrive    ext4    defaults    0    2
```

普通的硬盘用上述方式很容易就实现开机自动挂载了。但是我要挂载的硬盘是 `bcache` 文件系统的，“去掉” 8KiB 的头后才是 `ext4` 文件系统。那应该怎么办呢？

有两种解决的思路：
1. 把 `sudo mount -o loop,offset=8192 /dev/md0 /media/vision/HD8T` 命令写入 `/etc/init/*.conf` 文件， 在系统启动时执行；[[6](https://askubuntu.com/questions/54970/how-to-set-up-a-loop-device-at-boot-time)]
2. 考虑把 `loop,offset=8192` 这个选项加到 `/etc/fstab` 文件里，也是在系统启动时自动执行。[[7](https://bbs.archlinux.org/viewtopic.php?id=149560)] [[8](https://unix.stackexchange.com/questions/45018/how-to-mount-ext3-ext4-sitting-on-vdi-virtualbox-hdd)]

上面两种思路应该都是可行的，最后我只采用了第二种思路。这是因为我不光要挂载恢复的 RAID 硬盘，还要挂载一块 SSD，SSD 只要通过简单地修改常用的 `/etc/fstab` 文件就行了，不用改动其他文件。为了尽量保持操作的一致性，所以倾向于仅修改一个文件。此外，也有基于上面 UUID 优点的考虑，通过修改 `/etc/fstab` 文件，两块硬盘都可以通过 UUID 的方式被挂载。

最终在 `/etc/fstab` 中添加的内容如下，成功实现了开机自动挂载硬盘：
```text
# SSD 512G just for storage
UUID=5ab4e96e-d9b1-4de4-9437-cf03351fc650    /media/username/SSD512G    ext4    defaults    0    2

# Hard Disk 8T with bcache file system
UUID=76fb6f9c-d02d-42r7-83aa-76d984bfe381    /media/username/HD8T    auto    defaults,loop,offset=8192    0    2
```

注意上面的 `defaults,loop,offset=8192` 配置里 `,` 前后不要加空格，否则可能会出现问题。我在尝试的过程中就因为把这个文件写错了，导致无法进入桌面系统，只能进入命名行界面。

## 参考资料
1. [RAID，维基百科](https://zh.wikipedia.org/wiki/RAID)
2. [Reinstall Ubuntu, do i keep the software RAID?](https://ubuntuforums.org/showthread.php?t=2002217)
3. [在Linux下使用RAID（四）：创建RAID 5](http://blog.csdn.net/wzyzzu/article/details/48500017)
4. [Installing A New Hard Drive](https://help.ubuntu.com/community/InstallingANewHardDrive)
5. [Short Tip: Get UUID of Hard Disks](https://liquidat.wordpress.com/2007/10/15/short-tip-get-uuid-of-hard-disks/)
6. [How to set up a loop device at boot time?](https://askubuntu.com/questions/54970/how-to-set-up-a-loop-device-at-boot-time)
7. [losetup and mount during boot](https://bbs.archlinux.org/viewtopic.php?id=149560)
8. [How to mount ext3,ext4 sitting on VDI VirtualBox HDD?](https://unix.stackexchange.com/questions/45018/how-to-mount-ext3-ext4-sitting-on-vdi-virtualbox-hdd)
