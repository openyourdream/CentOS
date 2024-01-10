# linux安装配置-JDK配置

以jdk1.8版本为例安装演示。

### 卸载系统自带的jdk

#查看java相关进程
rpm -qa | grep java

#如果存在java进程，卸载进程，实例命令如下
rpm -e --nodeps java-1.8.0-openjdk-headless-1.8.0.65-3.b17.el7.x86_64

#查看jdk配置，如果没有，证明卸载成功
java -version



## 方式一：源码编译安装

### 下载安装包并解压

#下载的安装包信息：

jdk-8u391-linux-x64.tar.gz

#解压安装包到指定目录：

tar -zxvf jdk-8u391-linux-x64.tar.gz

### 配置java环境变量

#修改环境变量配置文件

vi /etc/profile

#在/etc/profile文件末尾添加如下内容

```shell
#java environment
export JAVA_HOME=/usr/local/jdk1.8.0_391
export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export PATH=$PATH:${JAVA_HOME}/bin
```

#使配置文件生效

source /etc/profile

#查看安装版本

java -version



## 方式二：rpm包安装

### #下载的安装包并安装

jdk-8u371-linux-x64.rpm

#安装命令

rpm -ivh jdk-8u371-linux-x64.rpm

#查询安装路径

find / -name java

#路径：/usr/lib/jvm/jdk-1.8-oracle-x64/bin/java

### 配置java环境变量

#修改环境变量配置文件

vi /etc/profile

#在/etc/profile文件末尾添加如下内容

```shell
# java-env
export JAVA_HOME=/usr/lib/jvm/jdk-1.8-oracle-x64
export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export PATH=$PATH:${JAVA_HOME}/bin
```

#使配置文件生效

source /etc/profile

#查看安装版本

java -version
