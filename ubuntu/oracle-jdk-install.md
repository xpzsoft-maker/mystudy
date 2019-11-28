# 1. 准备JDK
前往oracle官网，下载linux版本[jdk](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)，如jdk-8u221-linux-x64.tar.gz
# 2. 解压到用户安装目录
打开终端，输入命令：

```shell
    mkdir /usr/lib/java
    tar -zxvf ~/download/jdk-8u221-linux-x64.tar.gz -C /usr/lib/java
```

# 3. 配置环境变量

```shell
    sudo vim ~/.bashrc
```

```shell
    #set oracle jdk environment
    export JAVA_HOME=/usr/lib/java/jdk1.8.0_221  ## 这里要注意目录要换成自己解压的jdk 目录
    export JRE_HOME=${JAVA_HOME}/jre  
    export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib  
    export PATH=${JAVA_HOME}/bin:$PATH
```

```shell
    source ~/.bashrc ##使得配置生效
```

# 4. 注册java命令引用

```shell
    sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk1.8.0_181/bin/java 300  
    sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk1.8.0_181/bin/javac 300  
    sudo update-alternatives --install /usr/bin/jar jar /usr/lib/jvm/jdk1.8.0_181/bin/jar 300   
    sudo update-alternatives --install /usr/bin/javah javah /usr/lib/jvm/jdk1.8.0_181/bin/javah 300   
    sudo update-alternatives --install /usr/bin/javap javap /usr/lib/jvm/jdk1.8.0_181/bin/javap 300 
```
1. update-alertnatives --install link name path priority
- link 功能相同软件的公共链接目录
- name link名称
- path 实际执行目录
- priority 优先级

例如可以装多个jdk，然后通过`sudo alternatives --list java`查看名称为java的链接的实际目录，
用`sudo alternatives --config java`设置名称为java的链接的实际目录

# 5. 测试

```shell
    java
    java -version
    javac
```

