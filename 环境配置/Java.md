# Mac

官网下载安装好安装包（类似`jdk-8u181-macosx-x64.dmg`）之后可以如图所示找见安装路径

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/image-20200917171026849.png" alt="image-20200917171026849" style="zoom:50%;" />

环境配置也简单，其实就是在根目录下新建一个`.bash_profile`文件

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/image-20200917171152146.png" alt="image-20200917171152146.png" style="zoom:50%;" />

内容如下，**注意得用双引号，不然会报错**

```shell
JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_261.jdk/Contents/Home"
export JAVA_HOME
PATH="$JAVA_HOME/bin:$PATH:."
CLASSPATH="$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:."
export PATH
export CLASSPATH
```

最后输入`source .bash_profile`生效该配置文件

<img src="/Users/nymlc/Library/Application Support/typora-user-images/image-20200917171343287.png" alt="image-20200917171343287" style="zoom:50%;" />