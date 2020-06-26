---
layout: post
title: spring boot 部署
category: spring boot
tags: 部署
description: spring boot凭借它的自动化部署，大大提高了开发人员的开发效率。在使用它来开发服务器应用程序的时候，它默认包含了tomcat, 不需要额外的下载配置tomcat, 非常方便。其打包的时候也可以选择jar和war两种方式。jar内部包含了tomcat可以直接启动，此外还有boot-jar的打包方式，可以直接在命令行启动。
---

## 部署spring boot应用

应用的部署很多种形式，比如一些公司里面一般会用CI集成工具来自动化部署，当然也可以手动打包，上传到服务器然后启动，这里主要演示的是手动上传的方式。

### 打包

项目的打包使用的spring boot的maven 打包插件。我们使用boot-jar的打包方式，需要添加一个配置参数，如下。

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>

                <configuration>
                    <executable>true</executable>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

之后使用在项目根目录下运行命令`mvn package`或者使用IDE里的maven插件，执行package。之后，在根目录的target的文件夹下，应该会出现的打包了。默认为`{app-name}{version}.jar`.

### 上传

接下来就是上传打包文件到服务器上，一般我们使用的都是linux服务器，一般使用scp命令来出传输文件，scp命令和cp命令差不多, 就是将当前目录下的app.jar文件上传到ip为127.0.0.1的root用户的用户文件夹下。
例如下命令

```bash
scp  ./app.jar root@127.0.0.1:~/app.jar
```

命令运行后会输入密码，密码正确才会开始上传。当然每次都输密码非常的不方便，我们可以使用`-i {identity_file}`指定一个密钥文件(服务器的私钥)，就不用再输入密码,不过这样不安全，最好的做法是使用ssh-copy-id将自己的公钥推送到服务器，之后登陆就可以免密码了。

### 登陆

登陆一般使用ssh来登陆，例如登陆以root登陆一个地址为127.0.0.1的服务器,命令如下

```bash
ssh root@127.0.0.1
```

之后会提示我们输入密码，如果不想输入密码，也可以像scp命令一样指定一个密钥文件, 或者将自己的公钥推送到服务器。

### 运行应用

此时我们可以直接在命令行敲`./app.jar`，运行应用程序。不过这样不利于维护，一般的做法是将应用程序做欸一个linux服务运行。那么如何将我们的应用程序作为一个服务呢。命令如下：

```bash
ln -s -f /root/app.jar /etc/init.d/app
```

现在，我们已经将我们的程序注册为一个linux服务了，当然，有的小伙伴可能会疑惑，linux的服务是由/etc/init.d/目录下的对应的shell脚本控制的，为什么我们将我们的程序文件设置了一个软连接，就可以作为一个服务呢。
我们用vim,打开我们的程序文件看一下。

```bash
#!/bin/bash
#
#    .   ____          _            __ _ _
#   /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
#  ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
#   \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
#    '  |____| .__|_| |_|_| |_\__, | / / / /
#   =========|_|==============|___/=/_/_/_/
#   :: Spring Boot Startup Script ::
#

### BEGIN INIT INFO
# Provides:          app
# Required-Start:    $remote_fs $syslog $network
# Required-Stop:     $remote_fs $syslog $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: app
# Description:       Parent pom providing dependency and plugin management for applications built with Maven
# chkconfig:         2345 99 01
### END INIT INFO

  ......
```

我们的app.jar文件是二进制文件，应该乱码的，怎么会有这些东西。这是因为spring boot为了方便我们部署，在app.jar文件的头部添加了一个linux服务的shell脚本，这样我们就不必自己去写一个shell服务脚本了。这也是我们可以直接在shell里直接`./app.jar`启动应用程序的原因。
好了，注册为服务，接下来就是启动服务了。

```bash
service app start
```

## 小结

当然上述使用的是boot-jar的打包方式，如果你使用的是普通jar的打包方式，那你还是要使用`java -jar ./app.jar`来运行应用程序，同时注册为一个服务也需要自己写一个shell服务脚本，此外你还可以定制脚本内容。
