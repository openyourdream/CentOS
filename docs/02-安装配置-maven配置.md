# linux安装配置-maven配置

## maven安装

### 下载安装

#下载的安装包信息

apache-maven-3.8.2-bin.tar.gz

#解压安装包

tar -zxvf apache-maven-3.8.2-bin.tar.gz

#移动指定目录

 mv apache-maven-3.8.2 /usr/local/apache-maven-3.8.2

### 配置环境变量

#修改环境变量配置文件

vi /etc/profile

#在/etc/profile文件末尾添加如下内容

```shell
#maven environment
export MAVEN_HOME=/usr/local/apache-maven-3.8.2
export PATH=$PATH:$MAVEN_HOME/bin
```

#使配置文件生效

source /etc/profile

#验证成功

mvn -v
