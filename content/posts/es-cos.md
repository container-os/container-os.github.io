---
raw: https://hackmd.io/rkJUw7k4f/download
title: Easystack Container OS
zzzNote: do not edit the file directly, edit the raw data instead of it
---
深度剖析Easystack Container linux
===

容器OS介绍：

容 器 技 术 在 很 多 年 前 已 经 在 内 核 中 实 现 ,随 着 docker 以 及 集 群 工 具 swarm, kubernetes 的出现,简化了大规模部署容器应用的难度, 现在企业都可以自行给客户部署容器应用并且需求越来越大, 传统的操作系统针对部署容器又显得太厚重,container OS正是瞄准大量部署容器而生的轻量级操作系统。面对 container OS 的强烈需求,多家系统厂商都趁此机会纷纷的都推出了自己的 container OS 版本。easystack也基于开源atomic host项目研发了自己的容器操作系统Easystack Container linux。

atomic host是通过rpm包构建的轻量级操作系统,并支持‘原子’的升级和回滚,用户也可以选择不同的发行版本作为基础来管理你的机器。 Container OS内置了 rpm-ostree 的事务更新工具,它总是保留一份旧版本的操作系统用作回退,这一点类似于CoreOS。另外atomic host也是采用 selinux 技术来保护容器安全。atomic host是通过了整合现有可信赖的技术来实现了容器操作系统。下面让我们来具体看看怎么实现的吧。

 

容器OS系统升级回退流程分析：

    谈到container linux不得不先看看升级回退功能。用户通过在终端执行rpm-ostree upgrade/rollback来执行升级或回退。为什么能做到快速升级是因为ostree采用了增量更新的方式。说到增量更新还是先来看看ostree如何存储OS版本的。在ostree服务器端，有一个repo仓库目录用来存储多个OS系统版本。 在ostree中数据存储方式是像git一样都采用object对象存储的方式进行存储。每个对象都对应有checksum，对于不同的对象类型采用不同的后缀来表示， XX.File(z) 对应普通文件(压缩)和软链接文件,XX.dirmeta对应目录属性文件, XX.dirtree对应目录文件列表文件, XX.commit对应顶层指向 dirtree 和 dirmeta 的文件(XX=checksum)。

在ostree服务器端构建一个OS系统时，通过

rpmostree_compose_builtin_tree

# 将需要的rpm list安装到临时目录中

|--> install_packages_in_root

   # 为ostree重新制作initramfs

   |--> rpmostree_prepare_rootfs_for_commit

  # 将临时目录提交到object repo

  |--> rpmostree_commit

      # 遍历临时目录进行对象checksum计算

      |--> ostree_repo_write_dfd_to_mtree

         # 递归函数，用来遍历临时目录

         |--> write_dfd_iter_to_mtree_internal

在write_dfd_iter_to_mtree_internal通过递归遍历来填充mtree结构体的值。

获得当前目录属性get_modified_xattrs -> read_xattrs_cb 首先会尝试读取目录的security.capability，user.pax.flags属性并将这些通过GVariant实现数据序列化，如果支持selinux会通过selabel_lookup_raw读取path label，并将label与security.selinux属性添加进序列化数据。然后整个目录的属性metadata就会序列化成格式为：unix::uid@unx:gid@uuix:mode@xattrs的数据并通过计算数据的checksum值做为文件名存储在object repo中. 例如当前目录文件为属性生成的数据checksum为50773817e4519629fb061cb3cfe4ddae0a996c12336d087042481fbeab1a380c，那么就会保存为

./50/773817e4519629fb061cb3cfe4ddae0a996c12336d087042481fbeab1a380c.dirmeta的文件。

对于遍历到文件通过相同的方式读取到xattr值，最后组成序列化数据variant_size @ pad8字节align@unix::uid@uuix:gid@uuix:mode@symlnk_targetpath@xattrs@filestream,最后存储为./aa/788734c6db6dda5f4ae685888b35cc14195c7efa3925a04985c5b5147e2939.filez的文件。

而在file和目录之上就有dirtree类型的对象，用来表示一个目录包含的文件名和子目录名，以及他们对应的checksum值。数据格式化为(filename@checksum)*(subdirname@contentcheksum@metadatachecksum)*序列化数据, contentchecksum 代表subdir的dirtree数据类型的checksum。存储格式为：

./b1/b40ac7cdabc5daf0cf323832d3064d76fdc514ebb4549ad83d956121c23e00.dirtree.

最高一级类型就是commit类型对象，commit类型的序列化数据格式：

metadata@parentchecksum@subject@body@time@contentschecksum@metadatachecksum，

存储格式为：./92/919c27f64e4365010d7e97bfdd87dcaff23f8cf892f12a73855860e90a0356.commit。

所有对象采用类似树型的结构组织起来的。 到这里我们基本知道了服务器端是如何存储OS文件目录内容了,这样每个OS部署版本会对应一个commit checksum值。 结构关系见图3。

                              图3.

 

 

在用户执行upgrade时，根据配置的remote repo URL地址，通过libsoup库执行读取URL/config文件，获取仓库存储OS方式mode=archive-z2(压缩)，bare(未压缩). 通过读取URL/summary文件得到最新仓库commit提交checksum值，通过在本地repo/sysroot/ostree/repo目录下查找对应commit类型的文件，如果没有找到就下载.commit文件并开始解析commit内容并反序列化得到contentchecksum和metadtachecksum，继续迭代寻找对应的dirtree类型，接而是dirmeta类型，然后是file（z）类型。如果在本地repo中没有找到就通过libsoup通过解析的checksum数据在URL/objects 目录下寻找并下载，经过反复的循环查找。最后都会将服务器端缺的object下载到本地repo中。这就是一种增量更新。因为很多时候不同的OS版本有相同的目录内容和文件内容。比如新版本可能只修改了部分/etc配置文件或更新了某些rpm包，这样就有可能相同的目录文件在不同的OS版本共用。通过这种方式一方面减小了系统的大小，另一方面让更新下载更快。例如，在server端有新的commit2可以用来做upgrade，而在client端当前的commit1已经包含了FILE1，DIRTREE2，那么在client端做upgrade时只需要更新COMMIT2,DIRMETA3,FILE2文件就可以了。详见图4。

                                 图4.

Client repo对象更新后，然后就需要解析提交的commit repo将原始的文件目录结构反编译回来，通过函数checkout_deployment_tree 在本地创建/ostree/deploy/OS-NAME/deploy/${commit-checksum}.${deployserial}目录，并根据服务器存储逆向方法将序列化数据解析到对应的目录文件或目录。在这里因为ostree/deploy目录与系统目录是在同一个分区或LV中，所以只需要做hardlink指向就可以了。如果boot目录是单独分区，还需要拷贝vmlinuz到boot分区并更新/boot/loader/entries启动信息，

最后调用grub2生成grub2.cfg。

而对于rollback就比较简单了，因为在client端已经保存了旧的所有objects，只需要更新/boot的loaders信息重新生成grub2让第1个entries指向旧的启动COMMIT就可以了。

另外需要注意的地方是，在保存OS版本系统内容时，原始系统etc目录会被存储在usr/etc目录下，并拷贝一份出来作为当前etc目录，这样就能保证etc目录是可修改的。并且在版本升级时可以比较当前修改的etc目录与usr/etc目录计算当前OS版本修改的配置，并且将用户修改内容直接应用到新的OS版本中。

 

系统启动流程分析：

Container linux启动与常规的发布操作系统内核启动有些不一样， 我们先看看常规的操作系统启动流程。 操作系统上电后，会首先通过加载存储在磁盘MBR中的bootloader(grub2)程序，然后grub2程序读取boot分区中的grub2.cfg配置文件找到启动项，grub2程序就可以根据配置文件中指定的linux内核文件路径进行内核加载。到这里linux内核开始接管系统。来看看内核是如何加载initramfs的。

   内核源码分析start_kernel -> kernel_init -> kernel_init_freeable -> do_basic_setup -> do_initcalls, 调用模块的初始化，initramfs模块通过rootfs_initcall(populate_rootfs) 注册。在populate_rootfs中会根据是否是builtin的initramfs，initrd还是initramfs，进行分别处理。首先会解压builtin的initramfs内容到rootfs：unpack_to_rootfs(__initramfs_start, __initramfs_size)，__initramfs_start和__initramfs_end是内核builtin initramfs映射到内存的开始和结束地址。然后会根据是否有外部initrd/initramfs文件进行判断。对于initrd或initramfs外部文件，会在前面arch_init -> relocate_initrd根据bootloader加载initrd/initramfs到内存ramdisk_image地址的内容进行重新布局维护然后更新到地址initrd_start。然后在populate_rootfs 中进行解压，先按照initramfs方式进行解压，如果无法解压就会尝试initrd方法处理，先直接将内存中的内容写到/initrd.img文件。如果前面是initramfs并解压成功就会直接执行其中的/init文件。如果是initrd文件，在接下来prepare_namespace -> initrd_load 中， 会创建/dev/ram0设备，并且将前面存在/initrd.image文件通过rd_load_image加载到ram0中。如果/dev/ram0被指定为系统真正的根文件系统，直接执行/sbin/init文件。 如果/dev/ram0不是根文件系统，将会执行/linuxrc文件，根据linuxrc脚本信息umount initrd并mount真正的root根文件系统，最后执行/sbin/init文件。如果在builtin initramfs中没有提供/init(默认kernel没有提供)和在bootloader中没有提供外部initrd和initramfs文件的话，会在prepare_namespace -> mount_root最后直接mount root=指定的分区进行挂载, 执行/init脚本。流程可参考图1.

                                 图1

 

容器OS系统启动流程分析：

    以上是常规系统用户态初始化流程，但是 container linux 需要考虑到真正的root并不是直接存储在磁盘的某个分区或 LVM 逻辑卷。因为为了支持多个OS版本的管理，ostree采用类似于仓库的方式来存储OS所有信息，包括OS版本系统。这样container linux的存储方式就要求在启动的时候能够识别出真正的rootfs。

我们来看看ostree是如何存储和解析启动版本信息，假如系统boot目录是单独分区， 在boot目录中会有2个目录，ostree和loader.X， loader.X中的X代表简单的序号，一般是0和1，用来指向boot启动的版本序号。比如更新新的系统版本后，需要更新grub2信息就创建一个loader.X， 部署新版本信息会被写到其中entries目录下保存启动项信息：

options no_timer_check console=tty1 console=ttyS0,115200n8 crashkernel=auto rd.lvm.lv=atomicos/root root=/dev/mapper/atomicos-root ostree=/ostree/boot.1/escore-lite-host/86ebea26b5a15a7d0ce676c5a6cb5949a356f262660cf7aaf92614d6f4cef852/0

title EasyStack Container Linux 7.20171120.11 (Core) 7.20171120.11 (ostree)

linux /ostree/escore-lite-host-86ebea26b5a15a7d0ce676c5a6cb5949a356f262660cf7aaf92614d6f4cef852/vmlinuz-4.4.83-lite_1.el7.centos.es.x86_64

version 2

这些都是Bootloader需要用到的信息，在这里没有initrd外部文件，是因为easystack container linux采用的是builtin initramfs方式，这样能减小启动时间。然后会调用注册的grub2 callback(exec ostree admin instutil grub2-generate)。 它会解析loader.X/entries目录下的配置文件信息生成grub2.cfg 文件保存在loader.X下，并指向/boot/grub2/grub.cfg。这样启动后bootloader在开机时就能列出所有的启动OS部署版本。

在/boot/ostree目录会存储内核信息(和initramfs信息), 这也是在安装新的系统版本时被拷贝到ostree/os_name-checksum/目录下，这里的checksum是计算vmlinuz和 initramfs得出来的。这是为了适用如果多个OS版本采用的内核文件版本相同，这样只需要保存一份内核副本版本。

真正的root ostree把它隐藏在ostree=参数下。在这里我们看到目录是/ostree/boot.1/escore-lite-host/86ebea26b5a15a7d0ce676c5a6cb5949a356f262660cf7aaf92614d6f4cef852/0, boot.1这里是软连接，最后指向的是/ostree/deploy/escore-lite-host/deploy/86ebea26b5a15a7d0ce676c5a6cb5949a356f262660cf7aaf92614d6f4cef852/0, 这这里escore-lite-host是OS的名字，86ebea为OS部署版本checksum值，0为部署序号。在initramfs的阶段就需要将这个目录作为root挂载。

再来看看执行挂载步骤：首先通过mount /proc分区并读取/proc/cmdline，得到ostree= 参数。通过resolve_deploy_path获取绝对路径并chdir到目录中，先将../../var mount到当前的var目录（这里解释一下，var目录对于相同OS name的版本是共用的，因为var目录存储的都是可变数据，并不需要在ostree repo中管理）。然后将usr目录作为readonly的方式remount. 最后switch_root将当前ostree deploy目录作为root并执行/sbin/init并将源系统根目录切换到/sysroot下。这里涉及到的文件列表件图2.

                                图2.

到这里container linux启动流程已经分析完毕。

 

作者简介：

陈帆，多年内核虚拟化开发经验，涉及qemu，libvirt，kvm，xen，openstack-nova，gpu等，2016年加入easystack就职于系统工程团队，解决内核以及虚拟化相关问题，并主要担当easystack container linux OS调研设计开发工作。

 

 

 

 

 

 