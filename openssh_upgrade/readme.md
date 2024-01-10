

# Linux OpenSSH-9.5p1最新版升级步骤详细

参考地址：https://blog.csdn.net/wlc_1111/article/details/125228426

前言： 根据 国家信息安全漏洞共享平台 漏洞统计报告，信息安全数据泄露，漏洞入侵时时刻刻都在发生着，我们必须实时监控漏洞发现与威胁感知，及时预警网络安全漏洞的存在，以便及时整改修复漏洞，避免产生更大的安全事故，确保业务和系统的网络安全、稳定，促进社会的和谐发展。

## 一、OpenSSH 介绍

OpenSSH 是 SSH （Secure SHell） 协议的免费开源实现。SSH协议族可以用来进行远程控制， 或在计算机之间传送文件。而实现此功能的传统方式，如telnet(终端仿真协议)、 rcp ftp、 rlogin、rsh都是极为不安全的，并且会使用明文传送密码。OpenSSH提供了服务端后台程序和客户端工具，用来加密远程控制和文件传输过程中的数据，并由此来代替原来的类似服务。

## 二、源码包官网

目前基于最新版本 OpenSSH-9.0p1、OpenSSL1.1.1q 更新 ，测试环境为Centos7.x以上，请结合个人情况参考更新，其他版本请自测！

Zlib官网：http://www.zlib.net/
OpenSSL官网：https://www.openssl.org/
OpenSSH官网：https://www.openssh.com/

注： 手动下载官方源码包时，可能会非常慢，可以去常用的镜像站点（比如 清华镜像站、阿里镜像站 等）去下载

## 三、OpenSSH 安装

### 1.开启 telnet 服务，临时关闭防火墙

由于一般登录方式为ssh，所以需要安装其他登录方式，比如Telnet服务，防止升级失败

```shell
#安装telnet
yum install xinetd telnet-server telnet -y

#备份文件securetty
mv /etc/securetty.bak
#配置telnet登录的终端类型,增加一些pts终端
pts=$'pts/0\npts/1\npts/2\npts/3' && echo "$pts" >> /etc/securetty

#开启服务
systemctl start telnet.socket
service xinetd start

#关闭防火墙（或者开启23端口）
systemctl stop firewalld.service 
systemctl disable firewalld.service 
```

### 2.环境准备

```shell
#安装wget
yum install -y wget

#安装tar
yum install -y tar

#软件包准备
cd /opt   #解压目录
wget --no-check-certificate https://www.zlib.net/zlib-1.2.12.tar.gz
wget --no-check-certificate https://www.openssl.org/source/openssl-1.1.1q.tar.gz
wget --no-check-certificate https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-9.0p1.tar.gz

#解压软件包
tar -zxvf zlib-1.2.12.tar.gz
tar -zxvf openssl-1.1.1q.tar.gz
tar -zxvf openssh-9.5p1.tar.gz

#备份
mv /etc/ssh /etc/ssh.bak
cp /etc/pam.d/sshd.pam /etc/pam.d/sshd.pam.bak
cp /etc/init.d/sshd /etc/init.d/sshd.bak
cp /usr/bin/openssl /usr/bin/openssl.bak

##卸载原有的openssh
rpm -e --nodeps `rpm -qa | grep openssh`

###安装相关依赖包
yum install vim gcc gcc-c++ glibc make autoconf openssl openssl-devel pcre-devel pam-devel zlib-devel tcp_wrappers-devel tcp_wrappers
```

### 3.SSH安装

#### 3.1 Zlib安装

```shell
#进入zlib解压目录
cd /opt/zlib-1.2.12

#编译安装
./configure --prefix=/usr/local/zlib

make && make test && make install

#
ll /usr/local/zlib
ldconfig -v
/sbin/ldconfig
```

#### 3.2 OpenSSL安装

**注：** 

- 1. SSL更新前如果编译目录下有原版本，需删除后编译，防止SSH编译失败！ 
- 2.结尾处截图为初发文时间更新版本，和操作语句新版本不冲突

```shell
#进入openssl解压目录
cd /opt/openssl-1.1.1p

#编译安装
./config shared zlib --prefix=/usr/local/ssl

make clean && make -j 4 && make install 

#更新函数库
echo "/usr/lcoal/ssl/lib" >> /etc/ld.so.conf
ldconfig
ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl
ln -s /usr/local/ssl/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1
ln -s /usr/local/ssl/lib/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1

#检查是否升级成功
openssl version -a
```

#### 3.3 OpenSSH安装

```shell
#进入openssh解压目录
cd /opt/openssh-9.5p1

#编译安装
./configure --prefix=/usr/local/ssh --sysconfdir=/etc/ssh --with-ssl-dir=/usr/local/ssl --with-zlib=/usr/local/zlib 

make -j 4 && make install 

#查看目录版本
/usr/local/ssh/bin/ssh -V

#复制新ssh文件
cp -rf /opt/openssh-9.5p1/contrib/redhat/sshd.init /etc/init.d/sshd
cp -rf /opt/openssh-9.5p1/contrib/redhat/sshd.pam /etc/pam.d/sshd.pam
cp -rf /opt/openssh-9.5p1/sshd_config /etc/ssh/sshd_config

cp -rf /usr/local/ssh/sbin/sshd /usr/sbin/sshd
cp -rf /usr/local/ssh/bin/ssh /usr/bin/ssh
cp -rf /usr/local/ssh/bin/ssh-keygen /usr/bin/ssh-keygen

#开启sshd
chmod u+x /etc/init.d/sshd;
chkconfig --add sshd      ##自启动
chkconfig --list |grep sshd
chkconfig sshd on

#允许root登录
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
echo 'Subsystem sftp /usr/local/ssh/libexec/sftp-server' >> /etc/ssh/sshd_config

#重启sshd服务
/etc/init.d/sshd restart
/etc/init.d/sshd status

#查看升级后ssh版本
ssh -V
```

#### 4.关闭 telnet 服务

```shell
#关闭telnet服务
systemctl stop xinetd.service
systemctl stop telnet.socket

#卸载telnet服务
yum remove xinetd telnet-server telnet -y

#开启防火墙
systemctl start firewalld.service 
systemctl enable firewalld.service 
```

#### 5.SSH启动失败问题

5.1. Permissions 0737 for ‘/etc/ssh/ssh_host_rsa_key’ are too open”问题
 解决方法：更改报错文件权限

```sh
chmod 600 /etc/ssh/ssh_host_rsa_key
chmod 600 /etc/ssh/ssh_host_dsa_key
chmod 600 /etc/ssh/ssh_host_ecdsa_key
chmod 600 /etc/ssh/ssh_host_ed25519_key
```

### 四、OpenSSH-9.5p1 升级脚本*

注：考虑系统内容环境因素，脚本请根据个人情况，进行升级，固有些差异的地方需要根据实际情况修改！！

```shell
#!/bin/bash
#
#########################################################
# Function :openssh-9.5p1 update                        #
# Platform :Centos7.X                                   #
# Version  :2.0                                         #
# Date     :2024-1-10                                  #     
#########################################################

clear
export LANG="en_US.UTF-8"

#版本号
zlib_version="zlib-1.2.12"
openssl_version="openssl-1.1.1q"
openssh_version="openssh-9.5p1"

#安装包地址
file="/opt"

#默认编译路径
default="/usr/local" 
date_time=`date +%Y-%m-%d—%H:%M`

#安装目录
file_install="$file/openssh_install"
file_backup="$file/openssh_backup"
file_log="$file/openssh_log"

#源码包链接
zlib_download="https://www.zlib.net/$zlib_version.tar.gz"
openssl_download="https://www.openssl.org/source/$openssl_version.tar.gz"
openssh_download="https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/$openssh_version.tar.gz"


Install_make()
{
# Check if user is root
	if [ $(id -u) != "0" ]; then
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e " 当前用户为普通用户，必须使用root用户运行，脚本退出中......" "\033[31m Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	echo ""
	sleep 4
	exit
	fi

#判断是否安装wget
echo -e "\033[33m 正在安装Wget...... \033[0m"
sleep 2
echo ""
	if ! type wget >/dev/null 2>&1; then
		yum install -y wget
	else
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e " wget已经安装了：" "\033[32m Please continue\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	echo ""
	fi

#判断是否安装tar
echo -e "\033[33m 正在安装TAR...... \033[0m"
sleep 2
echo ""
	if ! type tar >/dev/null 2>&1; then
		yum install -y tar
	else
	echo ""
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e " tar已经安装了：" "\033[32m Please continue\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	fi
	echo ""

#安装相关依赖包
echo -e "\033[33m 正在安装依赖包...... \033[0m"
sleep 3
echo ""
	yum install gcc gcc-c++ glibc make autoconf openssl openssl-devel pcre-devel pam-devel zlib-devel tcp_wrappers-devel tcp_wrappers
	if [ $? -eq 0 ];then
	echo ""
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
   		echo -e " 安装软件依赖包成功 " "\033[32m Success\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	else
   	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
   		echo -e " 解压源码包失败，脚本退出中......" "\033[31m Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	sleep 4
	exit
	fi
	echo ""
}


Install_backup()
{
#创建文件（可修改）
mkdir -p $file_install
mkdir -p $file_backup
mkdir -p $file_log
mkdir -p $file_backup/zlib
mkdir -p $file_backup/ssl
mkdir -p $file_backup/ssh
mkdir -p $file_log/zlib
mkdir -p $file_log/ssl
mkdir -p $file_log/ssh


#备份文件（可修改）
cp -rf /usr/bin/openssl  $file_backup/ssl/openssl_$date_time.bak > /dev/null
cp -rf /etc/init.d/sshd  $file_backup/ssh/sshd_$date_time.bak > /dev/null
cp -rf /etc/ssh  $file_backup/ssh/ssh_$date_time.bak > /dev/null
cp -rf /usr/lib/systemd/system/sshd.service  $file_backup/ssh/sshd_$date_time.service.bak > /dev/null
cp -rf /etc/pam.d/sshd.pam  $file_backup/ssh/sshd_$date_time.pam.bak > /dev/null
}

Remove_openssh()
{
##并卸载原有的openssh（可修改）
rpm -e --nodeps `rpm -qa | grep openssh`
}

Install_tar()
{
#下载的源码包，检查是否解压（可修改）
#	if [ -e $file/$zlib_version.tar.gz ] && [ -e $file/$openssl_version.tar.gz ] && [ -e /$file/$openssh_version.tar.gz ];then
#		echo -e " 下载软件源码包已存在  " "\033[32m  Please continue\033[0m"
#	else
#		echo -e "\033[33m 未发现本地源码包，链接检查获取中........... \033[0m "
#	echo ""
#	cd $file
#	wget --no-check-certificate  $zlib_download
#	wget --no-check-certificate  $openssl_download
#	wget --no-check-certificate  $openssh_download
#	echo ""
#	fi
#zlib
echo -e "\033[33m 正在下载Zlib软件包...... \033[0m"
sleep 3
echo ""
	if [ -e $file/$zlib_version.tar.gz ] ;then
		echo -e " 下载软件源码包已存在  " "\033[32m  Please continue\033[0m"
	else
		echo -e "\033[33m 未发现zlib本地源码包，链接检查获取中........... \033[0m "
	sleep 1
	echo ""
	cd $file
	wget --no-check-certificate  $zlib_download
	echo ""
	fi
#openssl
echo -e "\033[33m 正在下载Openssl软件包...... \033[0m"
sleep 3
echo ""
	if  [ -e $file/$openssl_version.tar.gz ]  ;then
		echo -e " 下载软件源码包已存在  " "\033[32m  Please continue\033[0m"
	else
		echo -e "\033[33m 未发现openssl本地源码包，链接检查获取中........... \033[0m "
	echo ""
	sleep 1
	cd $file
	wget --no-check-certificate  $openssl_download
	echo ""
	fi
#openssh
echo -e "\033[33m 正在下载Openssh软件包...... \033[0m"
sleep 3
echo ""
	if [ -e /$file/$openssh_version.tar.gz ];then
		echo -e " 下载软件源码包已存在  " "\033[32m  Please continue\033[0m"
	else
		echo -e "\033[33m 未发现openssh本地源码包，链接检查获取中........... \033[0m "
	echo ""
	sleep 1
	cd $file
	wget --no-check-certificate  $openssh_download
	fi
}

echo ""
echo ""
#安装zlib
Install_zlib(){
echo -e "\033[33m 1.1-正在解压Zlib软件包...... \033[0m"
sleep 3
echo ""
    cd $file && mkdir -p $file_install && tar -xzf zlib*.tar.gz -C $file_install > /dev/null
    if [ -d $file_install/$zilb_version ];then
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
              		echo -e "  zilb解压源码包成功" "\033[32m Success\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	echo ""
        	else
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
              		echo -e "  zilb解压源码包失败，脚本退出中......" "\033[31m Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
    echo ""
    sleep 4
    exit
    fi
echo -e "\033[33m 1.2-正在编译安装Zlib服务.............. \033[0m"
sleep 3
echo ""
    cd $file_install/zlib*
	./configure --prefix=$default/$zlib_version > $file_log/zlib/zlib_configure_$date_time.txt  #> /dev/null 2>&1
	if [ $? -eq 0 ];then
	echo -e "\033[33m make... \033[0m"
		make > /dev/null 2>&1
	echo $?
	echo -e "\033[33m make test... \033[0m"
		make test > /dev/null 2>&1
	echo $?
	echo -e "\033[33m make install... \033[0m"
		make install > /dev/null 2>&1
	echo $?
	else
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e "  编译安装压缩库失败，脚本退出中..." "\033[31m Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	echo ""
	sleep 4
	exit
	fi

	if [ -e $default/$zlib_version/lib/libz.so ];then
	sed -i '/zlib/'d /etc/ld.so.conf
	echo "$default/$zlib_version/lib" >> /etc/ld.so.conf
	echo "$default/$zlib_version/lib" >> /etc/ld.so.conf.d/zlib.conf
	ldconfig -v > $file_log/zlib/zlib_ldconfig_$date_time.txt > /dev/null 2>&1
	/sbin/ldconfig
	fi
}

echo ""
echo ""
Install_openssl(){
echo -e "\033[33m 2.1-正在解压Openssl...... \033[0m"
sleep 3
echo ""
    cd $file  &&  tar -xvzf openssl*.tar.gz -C $file_install > /dev/null
	if [ -d $file_install/$openssl_version ];then
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
              		echo -e "  OpenSSL解压源码包成功" "\033[32m Success\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
        	else
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
              		echo -e "  OpenSSL解压源码包失败，脚本退出中......" "\033[31m Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
    echo ""
    sleep 4
    exit
    fi
	echo ""
echo -e "\033[33m 2.2-正在编译安装Openssl服务...... \033[0m"
sleep 3
echo ""
	cd $file_install/$openssl_version
        ./config shared zlib --prefix=$default/$openssl_version >  $file_log/ssl/ssl_config_$date_time.txt  #> /dev/null 2>&1
	if [ $? -eq 0 ];then
	echo -e "\033[33m make clean... \033[0m"
		make clean > /dev/null 2>&1
	echo $?
	echo -e "\033[33m make -j 4... \033[0m"
		make -j 4 > /dev/null 2>&1
	echo $?
	echo -e "\033[33m make install... \033[0m"
		make install > /dev/null 2>&1
	echo $?
	else
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e "  编译安装OpenSSL失败，脚本退出中..." "\033[31m Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	echo ""
	sleep 4
	exit
	fi

	mv /usr/bin/openssl /usr/bin/openssl_$date_time.bak    #先备份
	if [ -e $default/$openssl_version/bin/openssl ];then
	sed -i '/openssl/'d /etc/ld.so.conf
	echo "$default/$openssl_version/lib" >> /etc/ld.so.conf
	ln -s $default/$openssl_version/bin/openssl /usr/bin/openssl
	ln -s $default/$openssl_version/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1 
	ln -s $default/$openssl_version/lib/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1 
	ldconfig -v > $file_log/ssl/ssl_ldconfig_$date_time.txt > /dev/null 2>&1
	/sbin/ldconfig
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e " 编译安装OpenSSL " "\033[32m Success\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	echo ""
echo -e "\033[33m 2.3-正在输出 OpenSSL 版本状态.............. \033[0m"
sleep 3
echo ""
	echo -e "\033[32m====================== OpenSSL veriosn =====================  \033[0m"
	echo ""
		openssl version -a
	echo ""
	echo -e "\033[32m=======================================================  \033[0m"
	sleep 2
	else
	echo ""
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e " OpenSSL软连接失败，脚本退出中..." "\033[31m  Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	fi
}
echo ""
echo ""
Install_openssh(){
echo -e "\033[33m 3.1-正在解压OpenSSH...... \033[0m"
sleep 3
echo ""
	cd $file && tar -xvzf openssh*.tar.gz -C $file_install > /dev/null
	if [ -d $file_install/$openssh_version ];then
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
         echo -e "  OpenSSh解压源码包成功" "\033[32m Success\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
        	else
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
         echo -e "  OpenSSh解压源码包失败，脚本退出中......" "\033[31m Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
    echo ""
    sleep 4
    exit
    fi
	echo ""
echo -e "\033[33m 3.2-正在编译安装OpenSSH服务...... \033[0m"
sleep 3
echo ""
	mv /etc/ssh /etc/ssh_$date_time.bak     #先备份
	cd $file_install/$openssh_version
	./configure --prefix=$default/$openssh_version --sysconfdir=/etc/ssh --with-ssl-dir=$default/$openssl_version --with-zlib=$default/$zlib_version >  $file_log/ssh/ssh_configure_$date_time.txt   #> /dev/null 2>&1
	if [ $? -eq 0 ];then
	echo -e "\033[33m make -j 4... \033[0m"
		make -j 4 > /dev/null 2>&1
	echo $?
	echo -e "\033[33m make install... \033[0m"
		make install > /dev/null 2>&1
	echo $?
	else
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e " 编译安装OpenSSH失败，脚本退出中......" "\033[31m Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	echo ""
	sleep 4
	exit
	fi
	
	echo ""
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e " 编译安装OpenSSH " "\033[32m Success\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	echo ""
	sleep 2
	echo -e "\033[32m==================== OpenSSH—file veriosn =================== \033[0m"
	echo ""
		/usr/local/$openssh_version/bin/ssh -V
	echo ""
	echo -e "\033[32m======================================================= \033[0m"
	sleep 3
	echo ""

echo -e "\033[33m 3.3-正在迁移OpenSSH配置文件...... \033[0m"
sleep 3
echo ""
#迁移sshd
	if [ -f  "/etc/init.d/sshd" ];then
		mv /etc/init.d/sshd /etc/init.d/sshd_$date_time.bak
	else
		echo -e " /etc/init.d/sshd不存在 " "\033[31m Not backed up(可忽略)\033[0m"
	fi
	cp -rf $file_install/$openssh_version/contrib/redhat/sshd.init /etc/init.d/sshd;

	chmod u+x /etc/init.d/sshd;
	chkconfig --add sshd      ##自启动
	chkconfig --list |grep sshd;
	chkconfig sshd on
#备份启动脚本,不一定有
	if [ -f  "/usr/lib/systemd/system/sshd.service" ];then
		mv /usr/lib/systemd/system/sshd.service /usr/lib/systemd/system/sshd.service_bak
	else
		echo -e " sshd.service不存在" "\033[31m Not backed up(可忽略)\033[0m"
	fi
#备份复制sshd.pam文件
	if [ -f "/etc/pam.d/sshd.pam" ];then
		mv /etc/pam.d/sshd.pam /etc/pam.d/sshd.pam_$date_time.bak 
	else
        echo -e " sshd.pam不存在" "\033[31m Not backed up(可忽略)\033[0m"
	fi
	cp -rf $file_install/$openssh_version/contrib/redhat/sshd.pam /etc/pam.d/sshd.pam
#迁移ssh_config	
	cp -rf $file_install/$openssh_version/sshd_config /etc/ssh/sshd_config
	sed -i 's/Subsystem/#Subsystem/g' /etc/ssh/sshd_config
	echo "Subsystem sftp $default/$openssh_version/libexec/sftp-server" >> /etc/ssh/sshd_config
	cp -rf $default/$openssh_version/sbin/sshd /usr/sbin/sshd
	cp -rf /$default/$openssh_version/bin/ssh /usr/bin/ssh
	cp -rf $default/$openssh_version/bin/ssh-keygen /usr/bin/ssh-keygen
	sed -i 's/#PasswordAuthentication\ yes/PasswordAuthentication\ yes/g' /etc/ssh/sshd_config
	#grep -v "[[:space:]]*#" /etc/ssh/sshd_config  |grep "PubkeyAuthentication yes"
	echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config

#重启sshd
	service sshd start > /dev/null 2>&1
	if [ $? -eq 0 ];then
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e " 启动OpenSSH服务成功" "\033[32m Success\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	echo ""
	sleep 2
	#删除源码包（可修改）
	rm -rf $file/*$zlib_version.tar.gz
	rm -rf $file/*$openssl_version.tar.gz
	rm -rf $file/*$openssh_version.tar.gz
	#rm -rf $file_install
echo -e "\033[33m 3.4-正在输出 OpenSSH 版本...... \033[0m"
sleep 3
echo ""
	echo -e "\033[32m==================== OpenSSH veriosn =================== \033[0m"
	echo ""
		ssh -V
	echo ""
	echo -e "\033[32m======================================================== \033[0m"
	else
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
		echo -e " 启动OpenSSH服务失败，脚本退出中......" "\033[31m Error\033[0m"
	echo -e "\033[33m--------------------------------------------------------------- \033[0m"
	sleep 4
	exit
	fi
	echo ""
}

End_install()
{

##sshd状态
	echo ""
	echo -e "\033[33m 输出sshd服务状态： \033[33m"
	sleep 2
	echo ""
	systemctl status sshd.service
	echo ""
	echo ""
	echo ""
	sleep 1
	
echo -e "\033[33m==================== OpenSSH file =================== \033[0m"
echo ""
	echo -e " Openssh升级安装目录请前往:  "
	cd  $file_install && pwd
	cd ~
	echo ""
	echo -e " Openssh升级备份目录请前往:  " 
	cd  $file_backup && pwd
	cd ~
	echo ""
	echo -e " Openssh升级日志目录请前往:  "
	cd  $file_log && pwd
	cd ~
	echo ""
echo -e "\033[33m======================================================= \033[0m"
}


Install_make
Install_backup
Remove_openssh
Install_tar
Install_zlib
Install_openssl
Install_openssh
End_install
```







