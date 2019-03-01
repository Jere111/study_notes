[TOC]

## 一、前期准备

### 1.1 查看系统当前软件版本

```shell
# rpm -q zlib
# openssl version
# ssh -V
```

### 1.2 安装telnet服务并启用

```shell
# yum -y install telnet telnet-server*
```

### 1.3 先关闭防火墙，否则telnet可能无法连接

```sh
# service iptables stop
# chkconfig iptables off
```

**或**

**如果是：centos7**

```sh
#systemctl stop firewalld.service #停止firewall
#systemctl disable firewalld.service #禁止firewall开机启动
#systemctl restart firewalld.service #重启防火墙使配置生效
#systemctl enable firewalld.service #设置防火墙开机启动
```

**启动Telnet服务**

```sh
# systemctl enable telnet.socket 加入开机启动
# systemctl start telnet.socket 启动Telnet服务
# systemctl status telnet.cocket 再次查看服务状态
```

### 1.4 安装编译所需工具包

```sh
# yum -y install gcc pam-devel zlib-devel
```

### 1.5 上传依赖包（已上传远程服务器/home/gzsxgl/openssh）

- openssh-7.9p1.tar.gz  
- openssl-1.0.2q.tar.gz  
- zlib-1.2.11.tar.gz

在/home/gzsxgl/openssh目录下

## 二、正式升级

### 2.1 升级zlib

A、解压zlib_1.2.11源码并编译

```sh
# tar -zxvf zlib-1.2.11.tar.gz
# cd zlib-1.2.11
# ./configure --prefix=/usr
# make
```

B、卸载当前zlib

***注意：此步骤必须在步骤A执行完毕后再执行，否则先卸载zlib后，/lib64/目录下的zlib相关库文件会被删除，步骤A编译zlib会失败。（补救措施：从其他相同系统的服务器上复制/lib64、/usr/lib和/usr/lib64目录下的libcrypto.so.10、libssl.so.10、libz.so.1、libz.so.1.2.3四个文件到相应目录即可。可通过whereis、locate或find命令找到这些文件的位置）***

```sh
# rpm -e --nodeps zlib
```

C、安装之前编译好的zlib

在zlib编译目录执行如下命令：

```sh
# make install
```

D、共享库注册
zlib安装完成后，会在/usr/lib目录中生产zlib相关库文件，需要将这些共享库文件注册到系统中。

```sh
# echo '/usr/lib' >> /etc/ld.so.conf
# ldconfig                    #更新共享库cache
或者采用如下方式也可：
# ln -s  /usr/lib/libz.so.1 libz.so.1.2.11
# ln -s  /usr/lib/libz.so libz.so.1.2.11
# ln -s  /usr/lib/libz.so.1 /lib/libz.so.1
# ldconfig
```

可通过yum list命令验证是否更新成功（更新失败yum不可用），另外redhat和centos的5.*版本不支持高于1.2.3的zlib版本。

```sh
# yum list
```

### 2.2 升级OpenSSL

官方升级文档：http://www.linuxfromscratch.org/blfs/view/cvs/postlfs/openssl.html

A、备份当前openssl

```sh
# find / -name openssl
/usr/lib64/openssl
/usr/bin/openssl
/etc/pki/ca-trust/extracted/openssl
```

```sh
# mv  /usr/lib64/openssl /usr/lib64/openssl.old
# mv  /usr/bin/openssl  /usr/bin/openssl.old
# mv  /etc/pki/ca-trust/extracted/openssl  /etc/pki/ca-trust/extracted/openssl.old
如下两个库文件必须先备份，因系统内部分工具（如yum、wget等）依赖此库，而新版OpenSSL不包含这两个库
# cp  /usr/lib64/libcrypto.so.10  /usr/lib64/libcrypto.so.10.old
# cp  /usr/lib64/libssl.so.10  /usr/lib64/libssl.so.10.old
```

B、卸载当前openssl

```sh
# rpm -qa | grep openssl
```

直接执行此命令：

```sh
# rpm -qa |grep openssl|xargs -i rpm -e --nodeps {}
```

C、解压openssl_1.0.2k源码并编译安装

```sh
# tar -zxvf openssl-1.0.2q.tar.gz
# cd openssl-1.0.2q
# ./config --prefix=/usr --openssldir=/etc/ssl --shared zlib    #必须加上--shared，否则编译时会找不到新安装的openssl的库而报错
# make
# make test                            #必须执行这一步结果为pass才能继续，否则即使安装完成，ssh也无法使用
# make install
# openssl version -a                   #查看是否升级成功
```

D、恢复共享库
​    由于OpenSSL_1.0.2k不提供libcrypto.so.10和libssl.so.10这两个库，而yum、wget等工具又依赖此库，因此需要将先前备份的这两个库进行恢复，其他的可视情况考虑是否恢复。

```sh
# mv  /usr/lib64/libcrypto.so.10.old  /usr/lib64/libcrypto.so.10
# mv  /usr/lib64/libssl.so.10.old  /usr/lib64/libssl.so.10
```

### 2.3 升级OpenSSH

官方升级文档：http://www.linuxfromscratch.org/blfs/view/svn/postlfs/openssh.html

A、备份当前openssh

```sh
# mv /etc/ssh /etc/ssh.old
```

B、卸载当前openssh

```sh
# rpm -qa | grep openssh
```

直接执行此命令：

```sh
# rpm -qa |grep openssh|xargs -i rpm -e --nodeps {}
```

C、openssh安装前环境配置

```sh
# install  -v -m700 -d /var/lib/sshd
# chown  -v root:sys /var/lib/sshd
# groupadd -g 50 sshd
# useradd  -c 'sshd PrivSep' -d /var/lib/sshd -g sshd -s /bin/false -u 50 sshd
```

D、解压openssh_7.4p1源码并编译安装

```sh
# tar -zxvf openssh-7.9p1.tar.gz
# cd openssh-7.9p1
# ./configure --prefix=/usr  --sysconfdir=/etc/ssh  --with-md5-passwords  --with-pam  --with-zlib --with-openssl-includes=/usr --with-privsep-path=/var/lib/sshd

```

注意：若出现：configure: error: PAM headers not found 错误，需要安装pam-devel的rpm包

```sh
# yum install  –y  pam-devel 
```

如果在configure openssh时，如果有参数 --with-pam，会提示：
PAM is enabled. You may need to install a PAM control file for sshd, otherwise password authentication may fail. Example PAM control files can be found in the contrib/subdirectory

就是如果启用PAM，需要有一个控制文件，按照提示的路径找到redhat/sshd.pam，并复制到/etc/pam.d/sshd，在/etc/ssh/sshd_config中打开UsePAM yes。发现连接服务器被拒绝，关掉就可以登录。

```sh
# cp /home/gzsxgl/openssh/openssh-7.9p1/contrib/redhat/sshd.pam /etc/pam.d/sshd.pam
```

直接修改/etc/pam.d/sshd.pam为：

```tex
#%PAM-1.0
auth       required pam_sepermit.so
auth       include      password-auth
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    optional     pam_keyinit.so force revoke
session    include      password-auth
```

在openssh编译目录执行以下命令安装：

```sh
# make
# make install
```

E、openssh安装后环境配置

在openssh编译目录执行如下命令：

```sh
# install -v -m755    contrib/ssh-copy-id /usr/bin
# install -v -m644    contrib/ssh-copy-id.1 /usr/share/man/man1
# install -v -m755 -d /usr/share/doc/openssh-7.9p1
# install -v -m644    INSTALL LICENCE OVERVIEW README* /usr/share/doc/openssh-7.9p1
# ssh -V              #验证是否升级成功
```

F、启用OpenSSH服务

在openssh编译目录执行如下目录：

```sh
# echo 'X11Forwarding yes' >> /etc/ssh/sshd_config
# echo "PermitRootLogin yes" >> /etc/ssh/sshd_config   #允许root用户通过ssh登录
# cp -p contrib/redhat/sshd.init /etc/init.d/sshd
# chmod +x /etc/init.d/sshd
# chkconfig  --add  sshd
# chkconfig  sshd  on
# chkconfig  --list  sshd
```

还原之前的ssh配置信息，可直接删除升级后的配置信息，恢复备份。

```sh
# rm -rf /etc/ssh
# mv /etc/ssh.old /etc/ssh
```

注释/etc/ssh/sshd_config以下配置：

```sh
# vim /etc/ssh/sshd_config
```

#GSSAPIAuthentication no
#GSSAPICleanupCredentials no
#UsePAM yes

注释/etc/ssh/ssh_config以下配置：

```sh
# vim /etc/ssh/ssh_config
```

GSSAPIAuthentication no

GSSAPIAuthentication yes

开放文件权限：

```sh
# chmod 600 sshd_config ssh_host_dsa_key ssh_host_rsa_key ssh_host_ed25519_key ssh_host_ecdsa_key
# chmod 620 moduli
# chmod 644 ssh_config ssh_host_dsa_key.pub ssh_host_rsa_key.pub ssh_host_dsa_key.pub ssh_host_ecdsa_key.pub ssh_host_ed25519_key.pub
```

重启ssh服务：

```sh
# service sshd restart
```

---

打开堡垒机查看终端登录是否正常

查看升级版本

```sh
# ssh -V
```

