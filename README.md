

## 一：大致流程

1. 下载ubuntu-base-20.04.1-base-arm64.tar.gz
2. 安装qemu-user-static
3. 挂载ubuntu_rootfs
4. 打包成镜像
5. 烧录进开发板

## 二：具体操作

### 1.下载安装ubuntu18.04的根文件系统（注意需要架构相同：rk3568为arm64）

[ubuntu18.04]: http://cdimage.ubuntu.com/ubuntu-base/releases/18.04/release/

![](.\pic\1.jpg)

### 2 解压根文件系统

#### （1）创建一个文件夹存放根文件系统

```
mkdir ubuntu_rootfs
```

#### （2）解压到文件夹

```
sudo tar -xvf ubuntu-base-18.04.5-base-arm64.tar.gz -C ubuntu_rootfs/
#直接按照下面的也可
mkdir ubuntu_rootfs && cp ./ubuntu-base-18.04.5-base-arm64.tar.gz ./ubuntu_rootfs && cd ubuntu_rootfs && sudo tar -xvf ubuntu-base-18.04.5-base-arm64.tar.gz && rm ubuntu-base-18.04.5-base-arm64.tar.gz
```

### 3 配置根文件系统

#### （1）解压完成后需要配置网络

```
#拷贝本机的resolv.conf文件
sudo cp /etc/resolv.conf  ubuntu_rootfs/etc/resolv.conf	
```

#### （2）更换软件源,编辑根文件系统中的软件源配置文件（内容见附件），注：ARM版本的镜像为ubuntu-ports

```
sudo vim ubuntu_rootfs/etc/apt/sources.list
```

#### （3）配置仿真开发环境,宿主机下的Ubuntu默认不支持ARM架构，通过安装qemu-user-static仿真运行，构建Ubuntu文件系统

```
#1.安装qemu-user-static仿真器
	sudo apt install qemu-user-static
#2.拷贝 qemu-aarch64-static 到 ubuntu_rootfs/usr/bin/ 目录下。
	sudo cp /usr/bin/qemu-aarch64-static ubuntu_rootfs/usr/bin/
```

### 4 完善根文件系统

#### （1）编写挂载脚本 mount\.sh（内容见附件） ,用于挂载根文件系统运行所需要的设备和目录

#### （2）增加权限、挂载

```
#增加脚本执行权限 
	sudo chmod +x mount.sh 
#挂载根文件系统
	sudo ./mount.sh -m ubuntu_rootfs/ 
```

#### （3） 安装必要软件

```
#更新软件
	apt update
	apt upgrade
#安装必要工具
	apt install vim bash-completion net-tools iputils-ping ifupdown ethtool ssh rsync udev htop rsyslog nfs-common language-pack-en-base sudo psmisc 
#配置系统的时区、文字编码 
	apt install locales tzdata 
```

	#时区选择 (输入数字选择)
		Asia/Shanghai dpkg-reconfigure locales 
	#勾选英文和中文环境 (输入数字选择)
	    en_US.UTF-8 UTF-8 
	    zh_CN.UTF-8 UTF-8

```
#安装配置配置桌面环境(可选）
	apt install ubuntu-desktop
```

```
#设置主机名和主机解析
主机名
	echo "RK3588" > /etc/hostname
主机 IP
	echo "127.0.0.1 localhost" >> /etc/hosts
	echo "127.0.0.1 RK3588" >> /etc/hosts
	echo "127.0.0.1 localhost RK3588" >> /etc/hosts

实际测试时会因为接入网络系统无网络状态，开机时会等待很久，卡在网络上连接要5分钟

#修改下面这个文件
	vim /lib/systemd/system/networking.service
#将里面的TimeoutStartSec=5min修改为
	TimeoutStartSec=5sec
```

```
#串口调试设置root登录
	vi /lib/systemd/system/serial-getty\@.service 
#替换ExecStart这行
	ExecStart=-/sbin/agetty --autologin root --noclear %I $TERM
```

```
#禁用系统休眠
#设置禁止休眠 
	systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target 
#查看休眠状态 
	systemctl status sleep.target
```

```
#配置dhcp分配网络
	RK3568有两个网络端口，相应的在配置的时候我们要将两个网口都配置上
#修改网络配置文件
	vim /etc/network/interfaces

#以下是该文件具体内容
	# interfaces(5) file used by ifup(8) and ifdown(8)
	# Include files from /etc/network/interfaces.d:
	auto eth0
	iface eth0 inet dhcp
	auto eth1
	iface eth1 inet dhcp
	source-directory /etc/network/interfaces.d
```

```
#修改系统重启默认等待时间
#重启开发板的时候，会遇到进程没有结束系统在等待，默认的等待时间很长，导致重启时间变慢
#我们可以通过修改默认等待时间解决这个问题
	vim /etc/systemd/system.conf
#找到如下几行，并将其注释解除掉，然后修改 DefaultTimeoutStopSec=1s
	#DefaultTimeoutStartSec=90s
	#DefaultTimeoutStopSec=90s
	#DefaultTRestartSec=100ms
```

```
#扩充emmc分区
Ubuntu根文件系统打包成镜像并烧录到开发板以后，发现所占分区大小和镜像文件的大小是一样的，而实际上原先由buildroot构建的根文件系统在这个分区上是5.9G，所以下面这个图实际需求这里是不符合的

参考到其他博主的资料构建Ubuntu根文件系统扩充分区结合实际发现，需要充分利用到emmc的空间，在第一次运行时就要扩充分区大小，RK3568默认是对/dev/mmcblk0p6分区进行扩充。创建一个脚本和服务来扩充分区
如果不知道实际的文件系统设备的话可以用下面的方式进行查询
	mount | grep “^/dev”
	lsblk
扩充分区最主要的就是需要用到resize2fs 这个命令工具，以下是通过脚本和服务实现的过程

首先创建一个脚本vim etc/init.d/firstboot.sh，记得创建完成后要添加可执行权限chmod +x etc/init.d/firstboot.sh

#以下是firstboot.sh的内容
	#!/bin/bash -e
	# first boot configure
	# resize filesystem mmcblk0p6
	if [ ! -e "/usr/local/first_boot_flag" ] ;
	then
	  echo "Resizing /dev/mmcblk0p6..."
	  resize2fs /dev/mmcblk0p6
	  touch /usr/local/first_boot_flag
	fi

#其次是创建一个服务去实现脚本
	vim lib/systemd/system/firstboot.service
#然后再启动该服务
	systemctl enable firstboot.service
#以下是firstboot.service的内容
	#start
	[Unit]
	Description=Setup rockchip platform environment
	Before=lightdm.service
	After=resize-helper.service
	[Service]
	Type=simple
	ExecStart=/etc/init.d/firstboot.sh
	[Install]
	WantedBy=multi-user.target
	#end
```

#### （4）卸载

```
#退出根文件系统 
	exit 
#卸载根文件系统
	sudo ./mount.sh -u ubuntu_rootfs/
```

#### （5）打包根文件系统镜像

```
#首先创建一个空镜像文件，大小根据根据根文件系统的软件安装大小(是否安装图形化界面图像化界面)
#图形化界面一般要6g大小,非图形化界面2-4g左右都可
	dd if=/dev/zero of=ubuntu_rootfs.img bs=1M count=4096
#将该文件格式化为ext4文件系统
	mkfs.ext4 ubuntu_rootfs.img
#将镜像文件挂载到一个空文件中，并将ubuntu_roofs中的文件拷贝到该空文件中
	mkdir ubuntu_base_rootfs
	chmod 777 ubuntu_base_rootfs
	sudo mount ubuntu_rootfs.img ubuntu_base_rootfs
	sudo cp -rfp ubuntu_rootfs/* ubuntu_base_rootfs/
#复制完以后用e2fsck修复及检测镜像文件系统，resize2fs减小镜像文件的大小
	sudo umount ubuntu_base_rootfs/
	e2fsck -p -f ubuntu_rootfs.img
	resize2fs -M ubuntu_rootfs.img
```

### 5烧录根文件系统镜像到开发板中（镜像也有在附件中）

用瑞芯微提供的瑞芯微开发工具RKDevTool.exe去烧录文件系统即可

有俩种方式烧录一种是整个用编译sdk的方式打包成一个镜像后用升级固件的方式烧录（用自己编写的rootfs去替换生成的，不过sdk太大了，编译所需的16G我的小电脑也跑不动）。

另一种方法是单独烧录，在下载镜像中仅勾选rootfs，然后进入loader或则maskrom模式（我采用的是loader模式烧录，先按住V+，然后按住reset（这俩按钮要按久点），另一个模式也可以，就是V+键换成update键，其他一样）后点击设备分区表，右侧空白出来列表后再执行即可（否则俩个模式下直接按下执行都会出错）

![](.\pic\2.jpg)

### 6问题小结

#### （1）参考文章中提到的问题

1. 折腾的过程中碰到了很多问题，其中最主要的问题便是获取ip的问题，一直只能获取到inet6但是无法获取到inet，排查了很多方向，试过/etc/network/interfaces文件、netplan的配置文件，netmanager网络管理都行不通，再经过排查发现是dhcp分配有问题，但是安装dhcp服务器后对/etc/dhcp/dhcpd.conf和dhcp的监听端口设置进行配置修改也无法成功配置，遂心态崩溃，后面想着是不是驱动内核层面的问题，试着追溯回源，想起自己在烧根文件系统之前烧的boot.img是我之前项目修改过dts的内核模块，非sdk原初厂编译的boot.img，重新烧录后网络问题遂解决
2. 板子跑起来Ubuntu根文件系统后，apt安装软件的时候出现了设备空间不足的情况，发现不到1G，很奇怪觉得板子明明空间那么大，为什么现在就挂载的文件系统那么小，百思不得其解，后面参考了一篇文章才发现原因所在，可从本文中的扩充分区那一节看到具体原因以及解决方案

#### （2）自己遇到的问题

1. 有许多碰到的问题已经是在原来的问题上改好，最后才有的的这篇文章
2. 烧写ubuntu后，虽无法像正常一样识别到设备，但是进入那俩个烧录模式仍然正常，也就是说不影响后续修改和烧录
2. 用按键进入烧录实在麻烦，还不如先按住按键之后上电来的快
3. 若apt update 时fail to fetch ，404，请检查架构amd64还是arm64是否相同，[RK3399开发板ubuntu20.04报错 binary-arm64/Packages 404 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/589519525)
4. [debconf: unable to initialize frontend: Dialog的解决方法 - Hugo.Linの生活志 (399s.com)](https://399s.com/3089.html)
6.  [ubuntu 安装ssh 出现 Unsafe symlinks encountered in /var/log/journal, refusing. 错误_installed systemd-timesyncd package post-installat-CSDN博客](https://blog.csdn.net/weixin_44112313/article/details/119042268)
    [Unsafe symlinks · Issue #4922 · rsyslog/rsyslog (github.com)](https://github.com/rsyslog/rsyslog/issues/4922)

### 7.重要的参考文章

1. [【正点原子】手把手教你学RK3568 AI开发板环境搭建与系统编译 - 原子哥，专注电子技术教学 (yuanzige.com)](https://www.yuanzige.com/course/detail/80589?section_id=91584)
2. [RK356X/RK3588构建Ubuntu20.04根文件系统_rk3588 移植ubuntu-CSDN博客](https://blog.csdn.net/qq_38312843/article/details/133985193?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-1-133985193-blog-131682463.235^v40^pc_relevant_3m_sort_dl_base1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-1-133985193-blog-131682463.235^v40^pc_relevant_3m_sort_dl_base1&utm_relevant_index=2)
3. **[构建Ubuntu20.04根文件系统并移植到RK3568_rk3568 ubuntu-CSDN博客](https://blog.csdn.net/weixin_46025014/article/details/131682463)**
4. [RK35xx定制 Ubuntu18 根文件系统_rk3568 制作ubuntu20.04文件系统-CSDN博客](https://blog.csdn.net/lhc180/article/details/128477654?ops_request_misc=%7B%22request%5Fid%22%3A%22170573355616800213010989%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=170573355616800213010989&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-4-128477654-null-null.142^v99^pc_search_result_base4&utm_term=rk3568 ubuntu&spm=1018.2226.3001.4187)
5. [RSB-4810 RK3568平台三种烧录镜像方法说明_瑞芯微创建升级磁盘丁具v1.59-CSDN博客](https://blog.csdn.net/LXT_21/article/details/128676304)
6. [基于ubuntu-base构建根文件系统并移植到RK3568开发板_ubuntu-base-20.04.3-base-arm.tar.gz-CSDN博客](https://blog.csdn.net/JIangzc2019/article/details/121509816)
7. [移植带桌面ubuntu18.04到RK3568开发板_在rk3568板子上安装 ubuntu18.04-CSDN博客](https://blog.csdn.net/yufeng1108/article/details/125411845)
8. [RK3568开发笔记（四）：在虚拟机上使用SDK编译制作uboot、kernel和buildroot镜像_rksdk-CSDN博客](https://hpzwl.blog.csdn.net/article/details/125844240)
9. [RK3568开发笔记（五）：在虚拟机上使用SDK编译制作uboot、kernel和ubuntu镜像_瑞芯微rk3568 sdk开发之kernel编译-CSDN博客](https://hpzwl.blog.csdn.net/article/details/127783966)
10. [RK3568开发笔记（六）：开发板烧写ubuntu固件（支持mipi屏镜像+支持hdmi屏镜像）_rk3568开发笔记(六):开发板烧写ubuntu固件-CSDN博客](https://blog.csdn.net/qq21497936/article/details/132686096)
