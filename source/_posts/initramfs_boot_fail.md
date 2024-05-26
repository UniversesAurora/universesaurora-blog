---
title: 一次 Linux 5.4 内核启动失败分析
date: 2024-05-18 23:25:38
updated: 2024-05-24 23:40:31
cover: https://cdn.pixabay.com/photo/2023/12/29/15/37/balloons-8476432_1280.jpg
thumbnail: https://cdn.pixabay.com/photo/2023/12/29/15/37/balloons-8476432_1280.jpg
categories:
- 内核
tags:
- 内核
- initramfs
toc: true
---

在我司一台 x86 服务器上遇到了一个奇怪的问题：这台服务器上一直以来使用的是从 linux-stable 主线的 lts 版本自行编译的内核，具体来说使用的是5.15.15版本内核。

由于驱动需要我尝试在上面编译和引导5.4.172内核，但无法正常启动，内核打印显示找不到任何分区。初步判断是硬盘控制器驱动没有加载，但是驱动已经以模块形式编译了（该服务器使用的是一个博通 RAID 控制器，对应驱动为 megaraid_sas）。

<!-- more -->

无奈我尝试将该驱动编译到内核，这次内核成功找到了所有分区，但却无法找到内核 cmdline 指定的 rootfs uuid：

```sh
VFS: Cannot open root device "UUID=xxxxxxxxx" or unknown-block(0,0): error -6
Please append a correct "root=" boot option; here are the available partitions:
0800 34441003008 sda
driver: sd
0801 14680000 sda1 afc0d02a-b8be-407c-8ac6-ob3ac86a26d
0802 101376 sda2 eef76d84-e51a-49cb-b16b-a76fa7155b13
0803 16384 sda3 df9f72aa-cb58-blab-bl48f56be269
0804 22021166 sda4 c11f17f3-7ac6-l667-aaf0-a748cfbc193d
0805 31236209184 sda5 yyyyyyyyyy

Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
CPU:31 PID:1 comm swapper/0 Not tainted 5.4.172 #14
Hardware name: Dell Inc. PowerEdge R750/04V528, BIOS 1.7.4 06/27/2022
```

这里 rootfs 为 sda5，uuid `xxxxxxxxx` 通过检查 `blkid` 和 `lsblk -f` 输出可以确定是正确的。列表中 sda5 后的 uuid `yyyyyyyyyy` 与 `xxxxxxxxx` 并不相同，但在 `blkid` 输出中可以看到它是 sda5 的 `PARTUUID`：

```sh
% sudo blkid
/dev/sda5: UUID="xxxxxxxxx" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="yyyyyyyyyy"
/dev/loop1: TYPE="squashfs"
/dev/loop19: TYPE="squashfs"
```

# PARTUUID 与 UUID

那么这里的 `PARTUUID` 是什么？
其实 `UUID` 指的是 `Filesystem UUID`，即文件系统 UUID；相对的 PARTUUID 指的就是 `Partition UUID`，即分区的 UUID。顾名思义 `Filesystem UUID` 是储存在文件系统中的，而 `Partition UUID` 是储存在分区表中的。
Linux 中的大多数文件系统都支持 Filesystem UUID，例如 ext2/3/4、XFS。但 FAT 和 NTFS 对 UUID 的支持可能没有那么好。[^4]
如果没有一个可用的 initramfs 或者 initramfs 为空，内核就无法获取`UUID`（也不能获取 `LABEL`，即卷标）[^1]。
实际上，内核并不会处理 `UUID=` 这种标识符，而是会忽略该选项，由 initramfs 中的工具（如 udev）发现 `Filesystem UUID` 对应的设备并切换 rootfs。

```c
/*
 * Convert a name into device number.  We accept the following variants:
 *
 * 1) <hex_major><hex_minor> device number in hexadecimal represents itself
 *         no leading 0x, for example b302.
 * 2) /dev/nfs represents Root_NFS (0xff)
 * 3) /dev/<disk_name> represents the device number of disk
 * 4) /dev/<disk_name><decimal> represents the device number
 *         of partition - device number of disk plus the partition number
 * 5) /dev/<disk_name>p<decimal> - same as the above, that form is
 *    used when disk name of partitioned disk ends on a digit.
 * 6) PARTUUID=00112233-4455-6677-8899-AABBCCDDEEFF representing the
 *    unique id of a partition if the partition table provides it.
 *    The UUID may be either an EFI/GPT UUID, or refer to an MSDOS
 *    partition using the format SSSSSSSS-PP, where SSSSSSSS is a zero-
 *    filled hex representation of the 32-bit "NT disk signature", and PP
 *    is a zero-filled hex representation of the 1-based partition number.
 * 7) PARTUUID=<UUID>/PARTNROFF=<int> to select a partition in relation to
 *    a partition with a known unique id.
 * 8) <major>:<minor> major and minor number of the device separated by
 *    a colon.
 * 9) PARTLABEL=<name> with name being the GPT partition label.
 *    MSDOS partitions do not support labels!
 *
 * If name doesn't have fall into the categories above, we return (0,0).
 * block_class is used to check if something is a disk name. If the disk
 * name contains slashes, the device name has them replaced with
 * bangs.
 */

dev_t name_to_dev_t(const char *name)
```

上面内核源码中的注释说明了内核能够处理的几种设备标识符，其中并没有 `UUID`，大概看了下应该是在 `md_setup_drive` 中调用 `name_to_dev_t` 来解析标识符。
也可以尝试把 `UUID` 更改为一个不存在的值，最终控制台将输出如下内容：

```sh
Begin: Loading essential drivers ... done.
Begin: Running /scripts/init-premount ... done.
Begin: Mounting root file system ... Begin: Running /scripts/local-top ... done.
Begin: Running /scripts/local-premount ... done.
Begin: Waiting for root file system ... Begin: Running /scripts/local-block ... done.
done.
Gave up waiting for root file system device.  Common problems:
 - Boot args (cat /proc/cmdline)
   - Check rootdelay= (did the system wait long enough?)
 - Missing modules (cat /proc/modules; ls /dev)
ALERT!  UUID=50303285-93e1-49a2-92a0-24868766cff1 does not exist.  Dropping to a shell!


BusyBox v1.30.1 (Ubuntu 1:1.30.1-7ubuntu3) built-in shell (ash)
Enter 'help' for a list of built-in commands.

(initramfs)
```

很明显，这些输出内容是由 initramfs 中的程序或脚本输出的，内核本身并未输出错误信息。这也说明了 `UUID=` 标识符并非由内核处理。
内核中如果包含了所有所需的文件系统驱动，那么应该能够自行获得所有文件系统的 UUID，似乎并没有通过用户程序的必要。我也没有在网上发现这一做法的必要性的说法，因此推断该策略的目的可能是实现一定程度的解耦，毕竟正常情况下服务器系统都会加载 initramfs，而更精简的嵌入式领域则少有储存硬件变动的需求。通过用户程序控制该过程也能获得更多的灵活性。
这样看来，结合之前遇到的驱动问题，可以判断由于某些原因 initramfs 没有被正常加载，或者 initramfs 内容有误。

# 解决方案 A：让 grub 使用 PARTUUID 或路径

这里需要说明的是，使用 UUID 作为作为 rootfs 标识符是 grub 的默认行为。每当要从源码安装新内核时，一般做法是执行 `make install`，这背后执行了内核安装、DKMS模块编译、initramfs/initrd 生成、grub.cfg 生成等一系列操作。为了控制生成 grub.cfg 的默认行为，需要修改 /etc/default/grub 配置文件。
这里为了让 grub 在生成 grub.cfg 时使用路径（即 /dev/sdXX），可以在 /etc/default/grub 中添加：

```sh
GRUB_DISABLE_LINUX_UUID=true
```

但如果服务器的硬盘连接顺序发生变化（或者其他可能改变硬件拓扑的情况），rootfs 在系统中的路径就可能发生变化（如从 /dev/sda5 变为 /dev/sdb5），导致系统无法启动。这也是 grub 默认使用 UUID 的原因。
因此我们可以选择使用 `PARTUUID`，从 grub 2.04 版本[^2]开始，支持了一个新的选项 `GRUB_DISABLE_LINUX_PARTUUID`，通过与 `GRUB_DISABLE_LINUX_UUID` 结合可以让 grub 使用 `PARTUUID` 作为标识符[^3]：

```sh
GRUB_DISABLE_LINUX_UUID=true
GRUB_DISABLE_LINUX_PARTUUID=false
```

另外要注意的是，只有 GPT 分区表才有标准的 UUID。MBR 来说并没有一个真正的 UUID，它的 PARTUUID 要短得多，因为它实际上是 PTUUID + 分区编号。因为它们不够长，无法确保唯一性，并且无法保证分区一定存在 PARTUUID。具体解释参见：[partition - Are UUIDs and PTUUIDs important for MBR disks? If so, how do I create them on my own? - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/751946/are-uuids-and-ptuuids-important-for-mbr-disks-if-so-how-do-i-create-them-on-my)

# 分析 initramfs 加载

解决方案 A 只是治标不治本，initramfs 被正常加载才是我们最终的目标。
对比了 5.4.172 和 5.15.15 内核打印的区别，在 5.15.15 中找到两条关键打印：

```sh
 RAMDISK: [mem 0x3900b000-0x3fffdfff]
...
 Trying to unpack rootfs image as initramfs...
```

在内核中搜索了下，第一个打印来自于 `reserve_initrd`：

```c
static void __init reserve_initrd(void)
{
 /* Assume only end is not page aligned */
 u64 ramdisk_image = get_ramdisk_image();
 u64 ramdisk_size  = get_ramdisk_size();
 u64 ramdisk_end   = PAGE_ALIGN(ramdisk_image + ramdisk_size);
 u64 mapped_size;

 if (!boot_params.hdr.type_of_loader ||
     !ramdisk_image || !ramdisk_size)
  return;  /* No initrd provided by bootloader */

 initrd_start = 0;

 mapped_size = memblock_mem_size(max_pfn_mapped);
 if (ramdisk_size >= (mapped_size>>1))
  panic("initrd too large to handle, "
         "disabling initrd (%lld needed, %lld available)\n",
         ramdisk_size, mapped_size>>1);

 printk(KERN_INFO "RAMDISK: [mem %#010llx-%#010llx]\n", ramdisk_image,
   ramdisk_end - 1);
```

准备在内核中加入一些打印调试下，结果发现这次生成的内核出现了 unpack initrd 的打印，但仍然无法使用 UUID 启动。
尝试在自己的 Ubuntu 22.04 LTS 虚拟机（并且软件包均更新到最新版本）上也安装了5.4.172内核，意外发现与服务器上的现象一致。通过串口将错误现场的内核打印输出到文件中，发现以下打印：

```sh
[    1.839668] Unpacking initramfs...
[    1.843910] Initramfs unpacking failed: invalid magic at start of compressed archive
```

出现了明确的错误，这就很好解决了。在网上查了下该错误，是因为目前较新的发行版都使用 zstd 作为 initramfs 压缩算法，而该算法在 5.10 内核中才得到支持。[^5]

# 解决方案 B：配置使用 gzip 压缩 initramfs

对于 Ubuntu 来说，可以修改 `/etc/initramfs-tools/initramfs.conf` 文件，将 `COMPRESS=zstd` 修改为 `COMPRESS=gzip`[^6]。再次执行 `sudo make install` 或者 `sudo update-initramfs -c -k 5.4.172` 即可。
在服务器上修改了下，可以使用 UUID 标识 rootfs 了。但服务器之前并没有 unpack 出错的打印，打印内容也不太一样，根据代码来看是启用了 `CONFIG_BLK_DEV_RAM` 的原因：

```sh
 if (IS_ENABLED(CONFIG_BLK_DEV_RAM))
  printk(KERN_INFO "Trying to unpack rootfs image as initramfs...\n");
 else
  printk(KERN_INFO "Unpacking initramfs...\n");
```

这个选项是给 initrd 使用的[^7]，由于这里仅判断了该选项是否开启，所以和实际加载的是 initrd 还是 initramfs 并没有关系，这里我们使用的是 initramfs。

[^1]: [linux - Why can't I specify my root fs with a UUID? - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/93767/why-cant-i-specify-my-root-fs-with-a-uuid#comment142167_93767)
[^2]: [GRUB/Configuration variables - Gentoo Wiki](https://wiki.gentoo.org/wiki/GRUB/Configuration_variables)
[^3]: [Gentoo Forums - How to make GRUB pass PARTUUID to the kernel](https://forums.gentoo.org/viewtopic-t-1049514-start-0.html)
[^4]: [What UUIDs can do for you - Linux.com](https://www.linux.com/news/what-uuids-can-do-you/)
[^5]: [Initramfs unpacking failed: invalid magic as start of compressed - Support - Manjaro Linux Forum](https://forum.manjaro.org/t/initramfs-unpacking-failed-invalid-magic-as-start-of-compressed/137451/3)
[^6]: [boot - Having trouble with "Initramfs unpacking failed: Decoding failed" solutions - Ask Ubuntu](https://askubuntu.com/questions/1277532/having-trouble-with-initramfs-unpacking-failed-decoding-failed-solutions)
[^7]: [Linux Kernel Driver DataBase: CONFIG\_BLK\_DEV\_RAM: RAM block device support](https://cateee.net/lkddb/web-lkddb/BLK_DEV_RAM.html)
