---
title: Docker环境中systemd环境变量实验
date: 2022-02-22 18:06:49
tags:
- Docker
- Systemd
categories: MISC
description:
---
我们的业务`Docker`镜像是在[centos/systemd](https://hub.docker.com/r/centos/systemd)镜像基础上构建的，业务进程由`systemd`来启动。最近需要对业务逻辑进行改造，需要识别传入的环境变量。看上去是相当简单的改动，但在我们的进程中加入读取环境变量的逻辑却发现读取不到传入的变量内容，最终定位原因是在`systemd`的环境变量的处理。

<!--more-->

下面用简单的示例来说明。

首先基于`centos/systemd`构建实验镜像。创建三个文件:
`Dockerfile`, `envdemo.service`, `envdemo.sh`。


`Dockerfile`内容如下:
```docker
FROM centos/systemd:latest

COPY envdemo.service /usr/lib/systemd/system/

RUN mkdir /app
COPY envdemo.sh /app
RUN chmod a+x /app/envdemo.sh
WORKDIR /app

RUN systemctl enable envdemo.service
```

`envdemo.service`内容如下:
```ini
[Unit]
Description=Environment Variables Demo Service

[Service]
Type=simple
ExecStart=/app/envdemo.sh

[Install]
WantedBy=multi-user.target
```

`envdemo.sh`内容如下:
```bash
#!/bin/bash

exec >>/var/log/envdemo.log 2>&1

while true
do
    date
    echo "ENV_DEMO=${ENV_DEMO}"
    sleep 3
done
```

`systemd`镜像的`CMD`为`/usr/sbin/init`, 它实际链接的就是`systemd`:
```plain
[root@c6b3a7575cf6 app]# ls -l /usr/sbin/init
lrwxrwxrwx. 1 root root 22 Dec  5  2018 /usr/sbin/init -> ../lib/systemd/systemd
```

我们的进程`/app/envdemo.sh`由`systemd`启动运行。尽管`docker`推荐在一个镜像中只运行一个进程，但在一些需要运行多个服务的场景，用`systemd`启动这种方式还是比较优雅的。

构建我们的镜像:
```bash
docker build --rm --no-cache -t envdemo .
```

然后执行容器:
```bash
docker run --name envdemo --rm -d --privileged -e ENV_DEMO="hello" envdemo
```

之后进行容器查看:
```bash
[root@default envdemo]# docker exec -it envdemo bash
[root@fe4ced9ca691 app]# env |grep ENV_DEMO
ENV_DEMO=hello
```

从我们的`bash`进程中可以看到传入的环境变量的值。
查看日志`/var/log/envdemo.log`, 却看到环境变量的值为空:
```bash
[root@bb3f18d5bd77 app]# cat /var/log/envdemo.log
Tue Feb 22 09:16:45 UTC 2022
ENV_DEMO=
Tue Feb 22 09:16:48 UTC 2022
ENV_DEMO=
Tue Feb 22 09:16:51 UTC 2022
ENV_DEMO=
```

调研后发现原来是`systemd`启动进程时会清空所有的环境变量。知道了具体原因，就可以找到许多种不同的解决方案。

因为`systemd`本身是容器的`1`号进程，它会继承父进程的环境变量，因而在容器内可以从`/proc/1/environ`中获取所有的环境变量信息。

修改`envdemo.sh`为:
```bash
#!/bin/bash

exec >>/var/log/envdemo.log 2>&1

ENV_DEMO=`cat /proc/1/environ | tr '\0' '\n' | grep 'ENV_DEMO' | awk -F'=' '{print $2}'`
while true
do
    date
    echo "ENV_DEMO=${ENV_DEMO}"
    sleep 3
done
```

重新`build`镜像并执行之后，可以看到成功获取到环境变量:
```bash
[root@default envdemo]# docker exec -it envdemo bash
[root@c12073fa2585 app]# cat /var/log/envdemo.log
Tue Feb 22 09:26:39 UTC 2022
ENV_DEMO=hello
Tue Feb 22 09:26:42 UTC 2022
ENV_DEMO=hello
Tue Feb 22 09:26:45 UTC 2022
ENV_DEMO=hello
```

除了上边的方法，也可以使用`systemd`本身设置环境变量的方法来解决。`systemd`支持两种设置环境变量的方式:

* 在`service`文件中通过`Environment`指令设置，如:

```ini
Environment=ENV_FOO="foo"
Environment=ENV_BAR="bar"
```

* 也可以在使用`EnvironmentFile`指定一个文件，在文件中设置环境变量:

```ini
EnvironmentFile=/app/env.file
```

将`envdemo.service`文件修改为:
```
[Unit]
Description=Environment Variables Demo Service

[Service]
Type=simple
EnvironmentFile=/app/env.file
ExecStart=/app/envdemo.sh

[Install]
WantedBy=multi-user.target
```

将`envdemo.sh`恢复为最初版本:
```bash
#!/bin/bash

exec >>/var/log/envdemo.log 2>&1

while true
do
    date
    echo "ENV_DEMO=${ENV_DEMO}"
    sleep 3
done
```

重新`build`镜像。
创建环境变量文件`env.file`:
```plain
ENV_DEMO="env from file"
```

启动容器时将`env.file`映射到容器:
```bash
docker run --name envdemo --rm -d --privileged -v `pwd`/env.file:/app/env.file envdemo
```

进入容器查看, 可以看到`env.file`中设置的环境变量:
```bash
[root@default envdemo]# docker exec -it envdemo bash
[root@a55340c4b7d9 app]# cat /var/log/envdemo.log
Tue Feb 22 09:50:46 UTC 2022
ENV_DEMO=env from file
Tue Feb 22 09:50:49 UTC 2022
ENV_DEMO=env from file
Tue Feb 22 09:50:52 UTC 2022
ENV_DEMO=env from file
```

使用这种方式需要准备不同的文件并映射到不同的容器实例。我们实际部署使用`docker-compose`, 更倾向使用只需要在`docker-compose.yml`中设置不同的环境变量，而不用准备不同环境变量文件的方式。

能够解决问题的方法多种多样，也可以通过`Dockerfile`中`CMD`指定处理环境变量的程序来覆盖原有的`CMD`, 在这程序最后再`exec`调用`/usr/sbin/init`, 比如:
```
#!/bin/bash

exec >>/var/log/another.log 2>&1
date
echo $ENV_DEMO

exec /usr/sbin/init
```

参考:

* https://stackoverflow.com/questions/52571577/issues-in-accessing-docker-environment-variables-in-systemd-service-files
* https://wiki.archlinux.org/title/Systemd/User#Environment_variables
* https://dev.to/darnahsan/how-to-make-systemd-have-access-to-environment-variables-583b
* https://www.flatcar.org/docs/latest/setup/systemd/environment-variables/
* https://www.bmc.com/blogs/docker-cmd-vs-entrypoint/
* https://www.cloudbees.com/blog/understanding-dockers-cmd-and-entrypoint-instructions
* https://www.ctl.io/developers/blog/post/dockerfile-entrypoint-vs-cmd/
