# eagleslab
![image](https://user-images.githubusercontent.com/51991319/128116451-e5bab43d-9da3-489a-ab43-6eb194a1940d.png)
# PXE无人值守安装系统

使用PXE+KickStart可以通过非交互模式完成无人值守安装操作系统。

PXE 客户端从DHCP服务器获取到PXE服务端的具体IP，然后再从PXE配置文件中获取vmlinuz、initrd.img、ks.cfg、系统镜像等文件所在的服务器和位置信息。

# 什么是PXE?

- PXE，全名为Pre-boot Execution Environment，预启动执行环境
- 通过网络接口启动计算机，不依赖本地存储设备（如硬盘）或本地已经安装的操作系统
- 由Intel和Systemsoft公司于1999年9月发布的技术
- Client/Server的工作模式

<img src="12.kickstart/image-20210318163459935.png" alt="image-20210318163459935" style="zoom:80%;" />

# 批量装机软件

- Cobbler

- Kickstart是一种无人值守的安装方式。它的工作原理是在安装过程中记录人工干预填写的各种参数，并生成一个名为**ks.cfg**的文件。如果在自动安装过程中出现要填写参数的情况，安装程序首先会去查找ks.cfg文件，如果找到合适的参数，就采用所找到的参数；如果没有找到合适的参数，便会弹出对话框让安装者手工填写。所以，如果ks.cfg文件涵盖了安装过程中所有需要填写的参数，那么安装者完全可以只告诉安装程序从何处下载ks.cfg文件，然后就去忙自己的事情。等安装完毕，安装程序会根据ks.cfg中的设置重启/关闭系统，并结束安装。

# 实验部署

- 关闭防火墙和selinux

```- 
[root@server1 ~]# systemctl status firewalld.service 
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@server1 ~]# getenforce 
Permissive
```

- 配置DHCP Server：DHCP是一个局域网的网络协议，使用UDP协议工作，主要有两个用途，给内部网络或网络服务供应商自动分配IP地址，给用户或者内部网络管理员作为对所有计算机中央管理的手段。
- 准备网卡

```bash
[root@server1 ~]# nmcli connection modify ens37 ipv4.addresses 192.168.119.200/24 autoconnect yes ipv4.method manual
[root@server1 ~]# ip a | grep global
    inet 192.168.119.200/24 brd 192.168.119.255 scope global ens37

```

- 配置dhcp服务

```bash
[root@server1 ~]# yum -y install dhcp -y
[root@server1 ~]# vim /etc/dhcp/dhcpd.conf 
subnet 192.168.119.0 netmask 255.255.255.0 {
range 192.168.119.100 192.168.119.199;
option subnet-mask 255.255.255.0;
default-lease-time 21600;
max-lease-time 43200;
next-server 192.168.119.200;
filename "/pxelinux.0";
}
[root@server1 ~]# systemctl restart dhcpd
[root@server1 ~]# systemctl enable dhcpd
Created symlink from /etc/systemd/system/multi-user.target.wants/dhcpd.service to /usr/lib/systemd/system/dhcpd.service.

注：
range 192.168.119.100 192.168.119.199; # 可分配的起始IP-结束IP
option subnet-mask 255.255.255.0; # 设定netmask
default-lease-time 21600; # 设置默认的IP租用期限
max-lease-time 43200; # 设置最大的IP租用期限
next-server 192.168.119.200; # 告知客户端TFTP服务器的ip
filename "/pxelinux.0"; # 告知客户端从TFTP根目录下载pxelinux.0文件

# 验证端口号
[root@server1 ~]# ss -uanp | grep 67
UNCONN     0      0            *:67                       *:*                   users:(("dhcpd",pid=4333,fd=7))
```

- 安装tftp

  ```bash
  [root@server1 ~]# yum install tftp-server -y
  [root@server1 ~]# systemctl start tftp.socket 
  [root@server1 ~]# ss -uanp | grep 69
  UNCONN     0      0           :::69                      :::*                   users:(("systemd",pid=1,fd=25))
  ```

- pxe引导配置，syslinux是一个功能强大的引导加载程序，而且兼容各种介质。syslinux是一个小型的Linux操作系统，它的目的是简化首次安装或其他特殊用途的启动盘。首先需要将pxelinux.0配置文件复制到tftp目录下，再将光盘镜像中的一些文件复制到tftp的目录中。

```bash
[root@server1 ~]# yum install syslinux -y
[root@server1 ~]# cd /var/lib/tftpboot/
[root@server1 tftpboot]# cp /usr/share/syslinux/pxelinux.0 .
[root@server1 tftpboot]# mkdir -p /media/cdrom
[root@server1 tftpboot]# mount /dev/cdrom /media/cdrom/
mount: /dev/sr0 写保护，将以只读方式挂载
[root@server1 tftpboot]# cp /media/cdrom/images/pxeboot/{vmlinuz,initrd.img} .
[root@server1 tftpboot]# cp /media/cdrom/isolinux/{vesamenu.c32,boot.msg} .
```

- 配置syslinux服务程序，这个文件是开机时的选项菜单

```bash
[root@server1 tftpboot]# cp /media/cdrom/isolinux/isolinux.cfg pxelinux.cfg/default
[root@server1 tftpboot]# vim pxelinux.cfg/default 
  1 default linux
 64   append initrd=initrd.img inst.stage2=ftp://192.168.119.200 ks=ftp://192.168.119.200/pub/ks.cfg quiet
```

- 配置vsftpd服务程序，光盘镜像时通过ftp协议传输的，因此要用到vsftpd服务程序

```bash
[root@server1 tftpboot]# yum install -y vsftpd
[root@server1 tftpboot]# systemctl restart vsftpd
[root@server1 tftpboot]# systemctl enable vsftpd
Created symlink from /etc/systemd/system/multi-user.target.wants/vsftpd.service to /usr/lib/systemd/system/vsftpd.service.
[root@server1 tftpboot]# cp -r /media/cdrom/* /var/ftp/
```

- 创建Kickstart应答文件，Kickstart应答文件中包含了系统安装过程中需要使用的选项和参数信息，系统可以自动调取这个应答文件的内容，从而彻底实现无人值守安装系统。

```bash
[root@server1 tftpboot]# cp ~/anaconda-ks.cfg /var/ftp/pub/ks.cfg
[root@server1 tftpboot]# chmod +r /var/ftp/pub/ks.cfg 
[root@server1 tftpboot]# vim /var/ftp/pub/ks.cfg 
  5 url --url=ftp://192.168.119.200		# 删除原本的cdrom
 30 clearpart --all --initlabel			# 意思是清空所有磁盘内容并初始化磁盘
```

- 自动部署客户端主机

<img src="12.kickstart/image-20210318161345442.png" alt="image-20210318161345442" style="zoom: 80%;" />

![image-20210318161404558](12.kickstart/image-20210318161404558.png)



![image-20210318161404558](12.kickstart/image-20210318161404558.png)



<img src="12.kickstart/image-20210318161433898.png" alt="image-20210318161433898" style="zoom:80%;" />





<img src="12.kickstart/image-20210318161730515.png" alt="image-20210318161730515" style="zoom:60%;" />



<img src="12.kickstart/image-20210318162227229.png" alt="image-20210318162227229" style="zoom:80%;" />
