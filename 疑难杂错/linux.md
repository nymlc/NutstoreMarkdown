# 1. 安装`nodejs`

下载对应的版本，解压

```shell
wget https://nodejs.org/dist/v10.15.3/node-v10.15.3-linux-x64.tar.xz
tar -xvf node-v10.15.3-linux-x64.tar.xz
```

进入解压目录中，查看该包是否正常

```shell
	./bin/node -v
```

配置软连接，使得全局可用

```shell
## /usr/local/node/bin/node是解压包的路径
ln -s /usr/local/node/bin/node /usr/bin/node
ln -s /usr/local/node/bin/npm /usr/bin/npm
```

安装完`cnpm`之后可能需要配置软连接

```shell
ln -s /home/nymlc/Downloads/node-v10.17.0-linux-x64/bin/cnpm  /usr/local/bin/cnpm
```

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1628067171556.png" alt="屏幕截图 2021-08-04 164548" style="zoom:75%;" />

> `usr/bin` 是系统自带的应用
>
> `usr/local/bin` 是自己安装的应用和自己写的全局脚本

# 2. VirtualBox安装增强功能

有时候会报如下错误

![virtualbox centos7 安装增强功能时报错【未能加载虚拟光盘】非图形界面下的解决方案_梦启未来-CSDN博客](https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1628067600569.png)

这个只需要到虚拟机内的文件管理内弹出虚拟光驱`VBoxsGuestAdditions.iso`，然后重新安装即可

但是可能还是无效，那么就可以运行如下图的应用，若是闪退的话可以在命令行内用`sudo`运行

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1628067882230.png" alt="屏幕截图 2021-08-04 164548" style="zoom:75%;" />

