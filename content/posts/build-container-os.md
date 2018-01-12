---
raw: https://hackmd.io/AwIwHApgrAzATAFgLQQIYTkhBGVwmoJwDsSAbAMZnFjZghljAxA=/download
title: building Container OS
zzzNote: do not edit the file directly, edit the raw data instead of it
---
**build easystack container linux**
===
Escore-build是一个编译包和创建contianer linux的自主研发的工具集。
注意：如果在kvm虚机中安装，需要配置虚机内存大于2G，否则可能最后制作image时因为内存不足而失败。
# 1. 安装Escore-build工具

安装Escore-build时需要重新配置yum 源，所以在此之前，有一些工具包可以先安装上：
	yum install -y createrepo		用于创建yum源本地仓库

（1）由于我们的escore系统中存在已经安装的三个包(libsemanage,
cyrus-sasl-lib,policycoreutils)与rpm-ostree-toolbox所需的依赖包冲突(仅仅是三个包的后缀名有所不同)，所以需要先将系统中的包卸载，重新安装上所需包。
先到如下地址分别下载这三个rpm包到本地(可以用wget命令直接下载)：
	http://mirror.easystack.io/ESCL/vault.es/7.3.1611/updates/x86_64/Packages/libsemanage-2.5-5.1.el7.centos.es.x86_64.rpm
	http://mirror.easystack.io/ESCL/vault.es/7.3.1611/updates/x86_64/Packages/policycoreutils-2.5-11.el7.centos.es.x86_64.rpm
	http://mirror.easystack.io/ESCL/vault.es/7.3.1611/os/x86_64/Packages/cyrus-sasl-lib-2.1.26-20.el7.centos.es.x86_64.rpm
分别卸载escore系统中原来的三个包：
	rpm -e  libsemanage –nodeps
	rpm -e cyrus-sasl-lib –nodeps
	rpm -e policycoreutils –nodeps
最后安装下载下来的三个包：
	rpm -ivh libsemanage-2.5-5.1.el7.centos.es.x86_64.rpm
	rpm -ivh cyrus-sasl-lib-2.1.26-20.el7.centos.es.x86_64.rpm
	rpm -ivh policycoreutils-2.5-11.el7.centos.es.x86_64.rpm

（2）安装Escore-build之前，需要先配置yum源：
	vi /etc/yum.repos.d/CentOS-Base.repo 
修改为如下内容：
[base]
name=CentOS-$releasever - Base
baseurl=http://escore:escore@mirror.easystack.io/ESCL/vault.es/7.3.1611/os/x86_64/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[updates]
name=CentOS-$releasever - Updates
baseurl=http://escore:escore@mirror.easystack.io/ESCL/vault.es/7.3.1611/updates/x86_64/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[extras]
name=CentOS-$releasever - Extras
baseurl=http://escore:escore@mirror.easystack.io/ESCL/vault.es/7.3.1611/extras/x86_64/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[atomic]
name=CentOS-$releasever - Atomic
baseurl=http://escore:escore@mirror.easystack.io/ESCL/vault.es/7.3.1611/atomic/x86_64/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[easystack]
name=CentOS-$releasever - Easystack
baseurl=http://escore:escore@mirror.easystack.io/ESCL/vault.es/7.3.1611/easystack/x86_64/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[outatomic]
name=CentOS-$releasever - Outatomic
baseurl=http://mirror.easystack.io/ESCL/7.3.1611/atomic/x86_64/
gpgcheck=0

保存后，更新yum的cache：
	yum clean all	清除cache
	yum makecache		重新加载cache，提高搜索效率

（3）最后安装上escore-build：
	yum install -y escore-build
escore-build文件一般会在/opt/ostree/目录下

（4）最后制作image时会需要一个基础包net-tools，这里先安装上：
	yum install -y net-tools

# 2. Container OS 的源创建
Container OS 主要是由一系列的rpm包组成，因此默认我们提供repo源，包含
Escore-os，escore-update, escore-easystack,escore-extras ..., 参见escore-build的atomic目录下的.repo文件。这些repo也是我们escloud-linux 用的源。
但是我们可能需要添加额外的源，比如k8s更新。
（1）需要先建立本地yum源仓库：
	这里在根目录下mkdir k8s_update，
	copy 所有rpm包到该目录下，
	最后createrepo k8s_update，
成功会产生repodata目录，该目录下有repomd.xml文件自动创建索引信息。
每次更新k8s_update目录下的rpm包后，需要createrepo –update k8s_update.

（2）设置好yum源本地仓库后再到escore-build/atomic 目录下添加
K8s-update.repo文件：
[k8s-update]
name=ESCore-7 - k8s-update
baseurl=ftp://127.0.0.1/k8s_update/
enabled=1
gpgcheck=0
注意：请保证repo文件名与repo名相同，便于观察。

（3）最后需要在atomic/es-default.json文件中修改repos 项，添加k8s-update. 如：
 "repos": ["escore-os", "escore-updates", "escore-easystack",
	      "escore-extras", "escore-atomic", "k8s-update"],

# 3.创建contianer linux 仓库
Container linux的仓库类似与git仓库，但他是用来存储OS的。通过以上repo的修改，我们就需要创建我们自己的commit。默认我们的repo 路径在/srv/ostree, repo 名叫es-atomic-host。Refs默认是 es-atomic-host/7/x86_64/standard， 这里refs相当于git的branch(分支)。
有时我们需要创建一个test的branch，方便测试。我们可以修改 escore-build 目录下的envrc 文件（从envrc.example拷贝）。比如修改为：
OSTREE_REPO_REF = es-atomic-host/7/x86_64/yuzhun_test
（1）envrc文件相关配置含义：
	OSTREE_REPO指定OSTree仓库的根目录，默认在/srv/ostree/目录下；
	OSTREE_REPO_NAME 定义仓库名，默认为es-atomic-host；
	OSTREE_REPO_REF 指定分支目录，默认 /es-atomic-host/7/x86_64/standard；
	OSTREE_SERV_HOST 指定主机IP，默认为192.168.122.1，如果是在虚机中则需要更改IP为当前虚机的实际IP地址；
	OSTREE_SERV_PORT 指定端口；
当需要对以上配置修改时就需要将相关地方的注释符#取消，不做任何修改时使用默认配置，也就不需要copy envrc.example。
（2）如果是在虚机中，需要先配置虚机的CPU模式为host-passthrough模式，该模式直接将物理CPU暴露给虚拟机使用，在虚拟机上完全可以看到的就是物理CPU的型号。
在主机上虚机配置文件一般在/etc/libvirt/qemu/**.xml， 找到该虚机配置信息
	<cpu mode='custom' match='exact'>	
		<model fallback='allow'>IvyBridge</model>
	</cpu>
部分，将该部分修改为<cpu mode='host-passthrough'/>
再执行virsh define **.xml	定义域使文件生效；
最后虚机中执行：
	modprobe -r kvm_intel	
	modprobe kvm_intel nested=1	
卸载掉kvm_intel重新装，否则在虚机中跑会非常耗时。
（3）然后执行make atomic_compose 将rpm 打包成OS并提交。等执行完后，会发现commit成功的。如果需要覆盖原来的commit可以使用：
	make atomic_compose ARGS=--force-nocache。

# 4.创建image
有了以上的commit后，就可以通过这个commit来生成image。默认会按最新的commit生成image。
一般需要先重启(开启)docker：
	systemctl restart docker
关闭防火墙：systemctl stop firewalld.service
接着执行make atomic_prepare_image_dir （仅第一次时需要）
最后make atomic_image 就可以了。
最终可以看到生成的qcow2镜像文件所在目录类似如下：
Created: /srv/ostree/es-atomic-host/images/disk_171127-01/images/es-atomic-host-7.qcow2
