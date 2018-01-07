---
raw: https://hackmd.io/ryY5JyyNM/download
title: Container OS Quickview
zzzNote: do not edit the file directly, edit the raw data instead of it
---
# Container OS 分析比较


容器技术在很多年前已经在内核中实现，随着docker以及集群工具swarm, kubernetes的出现，简化了大规模部署容器应用的难度，现在企业都可以自行给客户部署容器应用并且需求越来越大，传统的操作系统针对部署容器又显得太厚重，container OS正是瞄准大量部署容器而生的轻量级操作系统。 面对container OS的强烈需求，多家系统厂商都趁此机会纷纷的都推出了自己的container OS 版本。大家肯定对他们很感兴趣，让我们来看看这些OS有什么特点和不同。让我们来看看这些OS有什么特点和不同。
## 目前几大container OS 版本

### Core OS
  CoreOS是衍生于chrome OS的轻量级linux操作系统，专注于大规模的部署，主要针对企业用户。并且获得了相当多的贡献者支持。 CoreOS附带了由CoreOS团队开发的一些非常有趣的工具，例如etcd，fleet和flannel。这些工具可以帮助您快速开始使用CoreOS集群。在2014年12月CoreOS团队宣布支持另一种容器引擎rkt, 这个也是从docker脱离的标志，但CoreOS依然能运行docker和rkt容器。CoreoS还和google合作，并创建了Tectonic商业平台，这是让CoreOS能轻松高效地运行在Kubernetes的非常有趣的工具。

### RancherOS
   RancherOS尝试把container OS做的更激进一步，RancherOS中所有的内容都是Docker容器。它运行一个系统容器作为系统进程号为1的进程，然后运行一个容器用来运行所有的用户容器。这看起来很不可思议，但对于一个不需要做其他工作的操作系统来说是很有道理的。更令人感兴趣的是在所有服务是添加到Ranche OS操作系统之上。 当您考虑生产系统需要什么功能时，您通常需要诸如安全和简单的网络，服务发现，负载平衡，监控和调度等功能。 Rancher添加了所有这些功能在RancherOS操作系统之上。 这是一个非常全面的系统。

### Snappy Ubuntu Core

Canonical推出Snappy Ubuntu Core，可搭載在Ubuntu Core的裝置上使用。 Snappy Ubuntu Core OS带有一种新型应用程序管理器（“snappy”），专注于运行应用程序和容器。系统的基础是“Ubuntu Core”，你的应用程序只能在只读镜像中生效运行，并且应用程序可以事务更新。当有新版本更新时，你不需要下载整个应用程序，仅仅下载新版本的更新内容就行。

### Atomic host

Atomic host 是通过centos，fedora和RHEL的上游rpm包构建的轻量级操作系统，

并支持‘原子’的升级和回滚，用户可以选择不同的发行版本作为基础来管理你的机器。OS内置了rpm-ostree的事务更新工具，它总是保留一份旧版本的操作系统用作回退，类似于CoreOS。Atomic host采用selinux技术来保护容器。Atomic host是通过了整合现有可信赖的技术来实现了容器操作系统。


## atomic host的实现原理

通过以上容器OS的对比，atomic host可能更适合我们现有的技术架构，下面我将详细介绍atomic host的实现原理。

Atomic host是一个基于不可变的系统架构而全新设计的操作系统， 使用的是通用的容器操作系统3层架构linux，docker和Kubernetes. Atomic host是一个轻量级的容器操作系统，基本思想是按照FHS(文件系统层次化标准) 将系统设置成不可修改的. 因为对于每一个OS都存储在上游仓库，所以支持大规模部署，应用都运行在容器里面。目前Atomic

Host支持centos版本和fedora版本。

目前Atomic host集成了kubernetes包的安装。但是目标还是实现kubernetes容器化，

将kubernetes应用放在容器中运行，这样在同一的Atomic host上，可以更简单的支持不同版本的应用, 例如openshift v3。 Atomic host 默认也集成了需要kubernetes工具， 例如flannel和etcd。Atomic host系统通过rpm-ostree来管理， rpm-ostree是一个为了实现管理可启动的，不可变的，版本化树型文件系统的开源工具。 Rpm-ostree 是ostree和rpm管理结合的衍生物。Atomic project还支持其他的容器工具包括cockpit，atomicapp等等， 方便部署容器应用。

 

下面我们就来看看Atomic host架构是怎么实现的。

Ostree：ostree其实是一个共享库和一套ostree工具的集合，它实现了类似于git版本管理工具的用来提交和下载一整套可独立运行的文件系统树结构。Ostree 的核心就像git一样，checksum单个文件，然后把内容按寻址对象存储起来。而有不同于git的checkout，

Ostree是通过hardlink来checkout，这样就可以保证所有在不同版本里的相同文件只有一份拷贝，节约了磁盘空间。

 

整个Atomic host的架构分为server 端和client端：

Atomic host Server 端通过rpm-ostree compose命令来组建和提交一个OS，类似用

Git的commit命令，rpm-ostree也是通过一个commit id来识别OS版本。ostree如何实现保存os的文件系统的呢？

Rpm-ostree compose 命令读取一个rpm系统包配置列表，然后通过创建一个临时的chroot环境将所有的包都安装在里面, 这样就形成了一个简单的文件系统，在这个文件系统中，用户可以自定义一下操作，必要添加或修改额外的文件，包括systemd启动服务等。然而这样还不够，为了实现让文件系统能运行，需要让这个OS被加载的时候能被bootloader所识别，所以initramfs需要重新生成，让initramfs在切换到当前OS的文件系统时，能够识别出用户需要进入的是哪一个系统。这里就需要让initramfs能够mount真正的ostree的OS文件系统作为根分区。Atomic host通过grub2 boot参数指定ostree=/ROOT, 告诉initramfs 真正的OS 文件系统在哪个路径，然后在initramfs执行systemd加载mount时，调用ostree提供的hook实现mount 根分区切换。 前面我们讲到，atomic host是不可变的， 那么在initramfs这里就有机会让usr目录mount成readonly的。现在OS的文件系统环境已经创建完毕了，我们需要把他们保存到ostree 的仓库中。Ostree repo支持bare，archive-z2(compressd), bare存储就是说仓库按文件本来的内容存储。Archive-z2存储的是压缩后的文件。

这里来看看具体如何存储的，ostree 会记录目录、文件的metadata(uid，gid和mode）包括文件额外属性，然后会将这些数据序列化后作为metadata保存到仓库的objects目录，这里通过把目录的属性序列化后，计算序列化后数据做checksum然后通过checksum数据将checksum数据分成前2位为仓库objects的第一级目录，checksum剪掉前面2位字符后面为文件名存储metadta，记录成类似ec/bf5a05f24a066524f22ed8dbf4224fab09293c10149968d5e17398ed829764.dirmeta 这样的数据。 对文件的存储将文件的metadata(uid,gid,mode,链接文件的目标文件)和额外属性属性序列化后和文件内容stream后计算序列化checksum，如果存储设置成压缩模式，在这里的文件内容stream就采用压缩处理。 所以看来通过checksum对目录和文件做唯一化处理。其实这里objects并不只能存储metadata和文件，objects对应了file, dirmeta，dirtree，commit,commitmeta. 每个目录类型会对应成dirtree的类型objects文件，记录目录里有哪些子目录和文件。 最后commit类型就是存储整个

文件系统的commit的checksum对象。

File 对应普通文件，软链接文件，dirmeta对应目录属性文件，dirtree 对应目录文件列表文件， commit对应顶层指向dirtree和dirmeta的文件。Commitmeta对应删除的commit对象文件。通过这些对象文件，ostree就可以反序列化列出有多少commit，commit有什么内容，内容里有什么文件等。最后通过调用ostree summary来更新os

server的总体概况信息, 会记录server最新的repo 分支更新时间，最新更新commit-id等等。

最后通过httpd服务来暴露OS 对象服务的全部内容， 对外呈现的是http://ip:port/地址url。

 

Atomic host Client 端通过rpm-ostree upgrade/rollback 来更新一个OS或者回退到先前的版本。来看看upgrade如果更新一个OS文件系统到client。

由于server 是通过http协议提供服务，client端就可以通过libcurl或者libsoup来实现下载, 默认rpm-ostree是采用libsoup的库.

首先通过获取url/config 获得os server是什么类型的存储方式, 可通过工具curl 获得，

比如获得mode是archive-v2, 通过获取url/summary获取summary数据。解析summary数据后获得分支最新的commit-id, 这样就有了分支和commit-id，客户端创建一个线程去做查询的动作，首先查询client本地repo是否有对应的commit的objects,

如果没有，就会通过将commit-id做checksum，然后按server端一样通过checksum计算存储路径转换成类似objects/XX/XXXXXX.commit, 通过libsoup库进行下载，然后进行迭代读commit的metadata数据，然后是dirtree的数据，直到最后全部下载到所有的repo数据，这里注意如果读取到file数据是压缩过的， 会进行解压存储，除非用户指定了client端做镜像。到这里基本client本地repo更新完成了，这样如果client本地已经有相同的文件，那么client也只会保存一份。节省系统了空间。

仓库更新后，下面我们就需要checkout这份OS了，checkout 开始解析server存储的commit的metadata，找到对应的commit包含的内容checksum和元数据checksum，

通过内容checkusm可以解析到commit下的目录名和文件名以及子目录等。最后通过获取的真实文件名和目录名，做硬链接到object数据。这样客户端的OS就基本完成了。最后需要做的就是发布新的OS，在发布OS的之前，需要做的就是拷贝一份etc目录出来，默认OS的etc保存在/usr/etc 目录下。然后更加当前client的etc目录以及当前client

的/usr/etc与新OS的/usr/etc目录做合并动作， 合并规则主要是比较client 的/usr/etc与client etc 文件，然后更新到新OS的etc 目录。最新新OS的启动信息。包括kernel的参数，主要是告诉guest在启动的时候能够remount到上面发布的根路径。硬链接或者拷贝vmlinuz到boot的ostree目录下，因为有可能boot挂载的是不同的分区。然后更新/etc/os-release 将OS 的release 更新成ostree的，最后就是组建grub2的entries，

保存文件到/boot/loader/entries/下，然后执行grub2-mkconfig 命令，ostree先会注册grub2的callback到/etc/grub.d/下，那么在执行grub2-mkconfig 的时候就会读取/boot/loader/entries/下boot配置文件，最后更新成新的grub2.cfg. 到这里整个client就创建好了。

 

Server 端 repo 示意图:

 圆圈对应每个object 文件， 假设1,4,7 代表commit类型， {1，2，3} 代表OS version1, {4,5,6} 代表OS version 2. {7,2,6} 代表OS version 3.

 







 

 

 

 

 

那么在client 从OS version 1 upgrade 到version 3 时只需要下载object 7和6就可以了。






 

 

给client端释放了大量的存储空间。

 

而对于client端来说， 由于OS大部分数据是readonly，如果client端需要添加第3方包怎么办， rpm-ostree提供了安装或卸载第3方包的功能，通过rpm-ostree install/uninstall包名来实现。由于是根据当前OS版本做更新，所有在client端就会先checkout一份OS base版本到临时目录， 然后通过libdnf库读取本地/etc/yum.repos.d/的repo数据下载包和依赖的包，然后将包逐个更新到新的cache repo中， 也称为pkgcache-repo， 这个repo会在默认存储在os repo路径下的extensions/rpmostree/pkgcache目录， 然后根据rpmdb计算每个包的添加还是删除，最后将包在pkgcache-repo的commit-id对应的tree更新到系统repo文件目录中，这里也是采用hardlink技术，然后根据新的OS目录，组建新的commit数据将整个OS更新到client的repo中，生成新系统的commit。

 

 例如： {1，2，3} 代表OS version1,  安装包包括{4，5}, 安装的时候会先更新到 pkgcache repo中， 然后rpm-ostree 会将{1,2,3,4,5} 组成新的commit 到OS version 2.

 

 

 

 






 

 

 

 

我们往往需要考虑到容器安全的问题，容器安全无非分两类：保护host不受影响和容器相互之间不受影响。在atomic host中，采用的是selinux来与docker结合来保护容器的以上2个方面的安全。我们来看看selinux如果是如何保护容器的。

 SELinux是一个Linux内核功能，允许对应用程序权限进行细粒度的限制。 每个进程都有一个关联的上下文，通过规则定义上下文之间的交互。 这可以严格限制哪些进程可以对另外的进程进行操作，以及可以访问哪些资源。每个文件，目录包括系统对象都有标签，通过配置标签规则达到控制的目的。Docker 提供2种类型的selinux存取控制保护机制：enforcement类型 和 multi-category security(MCS)隔离。

Enforcement类型是基于进程的规则，受限的容器进程默认svirt_lxc_net_t, 这种类型允许读取和执行/ usr下的所有文件类型以及/ etc下的大多数类型。 允许svirt_lxc_net_t使用网络，但不允许读取/ var，/ home，/ root，/ mnt下的内容。svirt_lxc_net_t被允许只写入标记为svirt_sandbox_file_t和docker_var_lib_t的文件。 容器中的所有文件都默认标记为svirt_sandbox_file_t。 docker卷的挂载时会使用标记docker _var_lib_t。MCS隔离有时被称为svirt, 为每个容器的SELinux标签的级别字段分配一个唯一的值。默认情况下，每个容器都分配了相当于启动容器的docker进程的PID的MCS Level。在OpenShift中，这可以被覆盖以基于UID生成MCS级别。此字段也可以用于多级安全（MLS）环境中，在这种环境中，需要将字段设置为TopSecret或Secret。标准的目标策略包括规定进程的MCS标签必须控制目标的MCS标签。目标通常是一个文件。 MCS标签通常看起来像s0：c1，c2这样的标签将控制s0，s0：c1，s0：c2，s0：c1，c2文件。但是，它不会控制s0：c1，c3。所有MCS标签都需要使用两个类别。这保证了默认情况下没有两个容器可以具有相同的MCS标签。具有s0的文件（系统上的大多数文件）不会被MCS阻止，对由类型强制执行的文件的访问仍将被强制执行。

例如有container A，B， A资源a1，B资源b1， A进程context设置为svirt_lxc_net_t :s0:c1,c2， a1的context设置为svirt_sandbox_file_t s0:c1,c2，B进程context设置为svirt_lxc_net_t:s0:c3,c4， b1的context设置为svirt_sandbox_file_t:s0:c3,c4. 这样A进程无法访问到b1, 就算是root权限也不行。

 

 

 

 

 

 

 

 

 

 



 


 

 

 

3.easystack通过对atomic ostree的理解，考虑到ostree可以裁剪的更小，一些userspace的工具可以采用busybox 工具集来提供，这样就减小了软件包的数量。为了解决依赖问题，我们可以让busybox提供对应包的依赖。这样一部分工具包就可以删除了。


 

 

 

 

 

 

 

 

 

 

4.easystack 通过对kernel的裁剪，让内核更利于容器OS，并删除了initramfs的加载，这样容器OS在启动的时候就会更快，比官方了OS启动时间减少了30%～40%。通过以上的ostree的加载流程，我们只需要一个简单的initramfs文件系统，让kernel能找到真正ostree发布的根目录就可以。这里通过遍历grub2 配置文件的ostree信息就知道根分区在什么目录了。执行根分区挂载后，执行进程为1的程序，执行systemd启动就行。考虑到ostree的initramfs是ostree生成的，为了防止ostree会自动生成initramfs，在服务器端compose OS时添加generate_initramfs配置项，如果为否，就不让ostree生成initramfs。

 

 

5. 考虑到在client端， OS为存储两次commit用来做rollback，这样如果两次 commit差别很大，包括kenrel版本更新，或者branch被切换， client 的仓库就会显得非常庞大.  easystack 系统团队根据atomic系统的immutable架构开发了yak工具, 通过对接ostree接口实现了本地只保留一份当前OS数据，然后通过yak保存rollback的元数据文件，当需要rollback的时候根据rollback保存的源数据，记录包括基本的commit-id，请求需要安装的rpm等，然后通过

上面提到的下载方法下载到rollback的repo数据，最后更新到一个新的rollback的OS。

实现真正的rollback，因为在OS rollback时，基本的不可变目录还是固定了，所以我们在本地保存变化的部分，我们就能安全的实现rollback。

 

例如： 默认atomic 提供2个os version用来做rollback，yak只保留当前OS系统版本，需要回退时通过ostree接口以及保存的metadata数据通过源端接口直接合并成rollback的OS。等回退到旧的OS后，就可以删除当前的OS来节约空间。





 


 

 

 

Yak:                   






 

 

 

合并metadta和repo数据到rollback OS






 


 

 

 

6. 为了方便制作ostree container OS， easystack研发了制作工具escore-build，可以很方便的制作contianer OS. 可以通过git下载或rpm下载方式。安装好后，默认会安装在/opt/escore-build，进入目录可以调用make help查看有些什么方法提供。在atomic子目录有一个es-default.json, 用户可以修改里面的内容，包括需要制作OS所需要的包名，需要替换的文件，需要systemd启动的服务等。具体配置可参见https://rpm-ostree.readthedocs.io/en/latest/manual/treefile/，关于配置项的说明。

修改了json文件，通过执行make atomic_compose 默认就会在/srv/ostree/目录下建立es-atomic-host 的仓库，当然这些名字可以在项目目录下copy envrc.example envrc文件修改。如果一切顺利的话， container OS的服务端就制作好了。然后通过执行make atomic_image 就会制作一个包含最新container OS的kvm 虚机镜像。这个镜像可以直接在kvm平台通过clould-init 注册用户密码登录， cloud-init默认用户名escore。