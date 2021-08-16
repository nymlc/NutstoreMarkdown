安装参见[官网](http://fmpp.sourceforge.net/index.html)

主要是环境变量设置，就是在[/Users/nymlc/.bash_profile](/Users/nymlc/.bash_profile)文件设置如下，主要是`FMPP_HOME`

```shell
JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_261.jdk/Contents/Home"
export JAVA_HOME

FMPP_HOME="/usr/lib/fmpp"
export FMPP_HOME


PATH="$JAVA_HOME/bin:$FMPP_HOME/bin:$PATH:."
CLASSPATH="$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:."


export PATH
export CLASSPATH
```

编辑完毕记得得`source ~/.bash_profile`，以生效

`fmpp`使用参见下例

```shell
fmpp "template/contact/user/userList.ftl" -o "dist/contact/user/userList.html" -D "tdd(../mock/sync/extra.tdd)" -S template -s
```

