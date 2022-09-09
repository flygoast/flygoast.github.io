---
title: CentOS7配置Console GUI/TUI程序
date: 2022-09-09 17:30:25
tags:
- TUI
categories: MISC
---
以虚拟机镜像或者自带操作系统形式交付的产品往往需要对系统进行各种各样的配置，如IP设置、DNS设置等等。为了提升用户体验，提供更简化的使用方式，往往会在`console`界面提供`GUI/TUI`的交互方式, 如`XenServer`的`console`界面:

{% img /images/2022-09-09/1.png %}

本文介绍在`CentOS7`系统上增加`GUI`界面的方法。

Linux系统的终端输入/输出设备一般称为`TTY`，是`TeleTypewriter`的缩写, 它的历史可以参考这两篇文章:

* https://itsfoss.com/what-is-tty-in-linux/
* https://www.linusakesson.net/programming/tty/index.php

`getty`是管理物理或者虚拟终端(`ttys`)程序的通用名称，表示`get tty`，用于防止非授权访问系统。当`getty`检测到终端连接时，它获取用户输入用户名, 然后执行`login`程序获取用户输入密码来进行登录校验。

从终端登录的一般过程是:

* 系统内核启动完成后，会启动`init`进程，它是一号进程。
* `init`进程会在特定的`tty`启动`getty`进程获取用户名。
* 调用`exec`执行`login`程序处理登录过程
* `login`登录校验成功后，执行相应用户指定的`shell`程序

<!--more-->

整体的流程图如下:
{% img /images/2022-09-09/2.png %}

`CentOS7`上的`init`程序已经从`SysV init`换为`systemd`，安装的`getty`程序为`agetty`。默认情况下，`CentOS7`有`6`个可用的`tty`，可以使用`Ctrl + Alt + F[1-6]`进行切换访问。

根据上述的终端登录流程，想要在终端显示`GUI`程序可以有几种方法:

* 将`GUI`程序作为`getty`程序执行
* 将`GUI`程序替换`login`程序执行，这种方式需要跳过`getty`程序读取用户名的步骤
* 将`GUI`程序作为`shell`程序执行，这种方式需要完成自动登录，跳过密码校验之前的步骤

`CentOS`的`init`程序为`systemd`。它通过`service`模板来定义`getty`服务，从而支持多个`getty`程序启动：
```bash
[root@default ~]# systemctl list-units --type=service |grep tty
getty@tty1.service                 loaded active running Getty on tty1
[root@default ~]# systemctl list-unit-files --type=service |grep '^getty@'
getty@.service                                enabled
```


下面分别来简单实验一下这几种方式。

###### 将`GUI`程序作为`getty`程序执行

我们写一个简单的`GUI`程序来替换`getty`, `getty`程序需要将标准输入/输出指向`/dev/ttyN`(`N`为数字)驱动文件。可以参考`agetty`的实现，它位于包:
```bash
[root@default ~]# rpm -qf /sbin/agetty
util-linux-2.23.2-63.el7.x86_64
```

之前的文章<<TUI库newt和snack简要介绍>>介绍过`Redhat`的`TUI`库`newt`, 这里基于`newt`实现一个简单的登录界面，`tuilogin.c`:
```c
#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>
#include <unistd.h>
#include <newt.h>
#include <assert.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <limits.h>


static void show_login_failed_window(void) {
    newtComponent label, button, form;

    newtCls();
    newtCenteredWindow(60, 6, "NOTE");
    label = newtLabel(10, 2, "Login failed!");

    button = newtCompactButton(30, 4, "Exit");

    form = newtForm(NULL, NULL, 0);
    newtFormAddComponents(form, label, button, NULL);
    newtRunForm(form);

    newtFormDestroy(form);
    newtPopWindow();
}

static void open_tty(char *tty) {
    int fd;
    char buf[PATH_MAX+1];

    snprintf(buf, sizeof(buf), "/dev/%s", tty);

    fd = open(buf, O_RDWR|O_NOCTTY|O_NONBLOCK, 0);
    if (fd < -1) {
       fprintf(stderr, "open %s failed: %d\n", fd);
       exit(-1);
    }

    close(STDIN_FILENO);
    close(STDOUT_FILENO);
    close(STDERR_FILENO);

    assert(dup2(fd, 0) == 0);
    assert(dup2(fd, 1) == 1);
    assert(dup2(fd, 2) == 2);

    close(fd);

    setvbuf(stdout, NULL, _IONBF, 0);
    setenv("TERM", "linux", 1);
}


int main(int argc, char **argv) {
    int cols, rows;
    char *bash_args[1];

    if (argc != 2) {
        fprintf(stderr, "usage: menu <tty>\n");
        exit(-1);
    }

    open_tty(argv[1]);

    newtInit();

    do {
        newtCls();
        newtPushHelpLine(NULL);

        newtGetScreenSize(&cols, &rows);
        newtOpenWindow(1, 1, cols - 2, rows - 4, "LOGIN");

        newtComponent form, btn_ok, label_username, label_password,
                      entry_username, entry_password, result;
        const char *username, *password;

        label_username = newtLabel(10, 3, "Username:");
        entry_username = newtEntry(20, 3, "", 20, &username,
                                   NEWT_FLAG_SCROLL);
        label_password = newtLabel(10, 5, "Password:");
        entry_password = newtEntry(20, 5, "", 20, &password,
                                   NEWT_FLAG_PASSWORD | NEWT_FLAG_SCROLL);

        result = newtLabel(10, 10, "");

        btn_ok = newtButton(20, 7, "  OK  ");

        form = newtForm(NULL, NULL, 0);

        newtFormAddComponents(form, label_username, entry_username,
                              label_password, entry_password,
                              btn_ok, result,
                              NULL);
        struct newtExitStruct exit_status;
        newtFormRun(form, &exit_status);

        if (exit_status.reason == NEWT_EXIT_COMPONENT) {
            if (exit_status.u.co == btn_ok) {
                if ((strcmp(username, "root") == 0)
                    && (strcmp(password, "123456") == 0))
                {
                    /* login success */
                    break;
                } else {
                    show_login_failed_window();
                }
            }
        }

        newtRefresh();
        newtFormDestroy(form);
        newtPopWindow();
        newtPopHelpLine();

    } while (1);

    newtFinished();

    bash_args[0] = "bash";

    execv("/bin/bash", bash_args);

    return 0;
}
```

编译:
```bash
gcc -o tuilogin tuilogin.c -lnewt
```

程序功能为，当用户名输入`root`，密码输入`123456`时，执行`bash`进程。

将编译后的`tuilogin`放到`/sbin/`目录。

然后使用`systemd`的`drop-in files`机制来修改`tty1`的启动逻辑:

```bash
systemctl edit getty@tty1
```
输入内容:
```plain
[Service]
ExecStart=
ExecStart=-/sbin/tuilogin %I
```
这会创建文件`/etc/systemd/system/getty@tty1.service.d/override.conf`

重启系统:
{% img /images/2022-09-09/3.png %}

可以看到`TUI`程序在机器启动时执行，当输入`root`和`123456`之后，进入`bash`界面。

###### 将`GUI`程序替换`login`程序执行

因为`agetty`已经处理了`/dev/ttyN`文件的处理，我们去掉上述代码中`tty`的处理逻辑:

```c
#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>
#include <newt.h>

void show_login_failed_window(void) {
    newtComponent label, button, form;

    newtCls();
    newtCenteredWindow(60, 6, "NOTE");
    label = newtLabel(10, 2, "Login failed!");

    button = newtCompactButton(30, 4, "Exit");

    form = newtForm(NULL, NULL, 0);
    newtFormAddComponents(form, label, button, NULL);
    newtRunForm(form);

    newtFormDestroy(form);
    newtPopWindow();
}

int main(void) {
    int cols, rows;
    char *bash_args[1];

    newtInit();

    do {
        newtCls();
        newtPushHelpLine(NULL);

        newtGetScreenSize(&cols, &rows);
        newtOpenWindow(1, 1, cols - 2, rows - 4, "LOGIN NOTTY");

        newtComponent form, btn_ok, label_username, label_password,
                      entry_username, entry_password, result;
        const char *username, *password;

        label_username = newtLabel(10, 3, "Username:");
        entry_username = newtEntry(20, 3, "", 20, &username,
                                   NEWT_FLAG_SCROLL);
        label_password = newtLabel(10, 5, "Password:");
        entry_password = newtEntry(20, 5, "", 20, &password,
                                   NEWT_FLAG_PASSWORD | NEWT_FLAG_SCROLL);

        result = newtLabel(10, 10, "");

        btn_ok = newtButton(20, 7, "  OK  ");

        form = newtForm(NULL, NULL, 0);

        newtFormAddComponents(form, label_username, entry_username,
                              label_password, entry_password,
                              btn_ok, result,
                              NULL);
        struct newtExitStruct exit_status;
        newtFormRun(form, &exit_status);

        if (exit_status.reason == NEWT_EXIT_COMPONENT) {
            if (exit_status.u.co == btn_ok) {
                if ((strcmp(username, "root") == 0)
                    && (strcmp(password, "123456") == 0))
                {
                    /* login success */
                    break;
                } else {
                    show_login_failed_window();
                }
            }
        }

        newtRefresh();
        newtFormDestroy(form);
        newtPopWindow();
        newtPopHelpLine();

    } while (1);

    newtFinished();

    bash_args[0] = "bash";

    execv("/bin/bash", bash_args);

    return 0;
}
```

`agetty`程序支持通过命令行指令被调用的`login`程序:
```plain
       -l, --login-program login_program
           Invoke the specified login_program instead of /bin/login.
           This allows the use of a non-standard login program. Such a
           program could, for example, ask for a dial-up password or use
           a different password file. See --login-options.
```

```plain
       -n, --skip-login
           Do not prompt the user for a login name. This can be used in
           connection with the --login-program option to invoke a
           non-standard login process such as a BBS system. Note that
           with the --skip-login option, agetty gets no input from the
           user who logs in and therefore will not be able to figure out
           parity, character size, and newline processing of the
           connection. It defaults to space parity, 7 bit characters,
           and ASCII CR (13) end-of-line character. Beware that the
           program that agetty starts (usually /bin/login) is run as
           root.
```


再次修改`getty@tty1`的`drop-in file`:
```bash
systemctl edit getty@tty1
```
内容为:
```plain
[Service]
ExecStart=
ExecStart=-/usr/sbin/agetty --login-program=/sbin/tuilogin_notty -n %I $TERM
```

重启系统, 可以看到新的`TUI`程序正常执行:
{% img /images/2022-09-09/4.png %}


###### 将`GUI`程序作为`shell`程序执行

这种方式需要完成自动登录，跳过密码校验之前的步骤，可以使用`agetty`的`-a`选项:
```plain
       -a, --autologin username
           Automatically log in the specified user without asking for a
           username or password. Using this option causes an -f username
           option and argument to be added to the /bin/login command
           line. See --login-options, which can be used to modify this
           option’s behavior.

           Note that --autologin may affect the way in which getty
           initializes the serial line, because on auto-login agetty
           does not read from the line and it has no opportunity
           optimize the line setting.
```
再次修改`getty@tty1`的`drop-in file`:

```bash
systemctl edit getty@tty1
```
内容为:
```bash
[Service]
ExecStart=
ExecStart=-/usr/sbin/agetty --autologin root --noclear %I $TERM
```

这次我们使用`dialog`程序来创建一个`TUI`程序用来和之前的窗口区分。
安装`dialog`:
```bash
yum install -y dialog
```

创建`/sbin/tuilogin_shell`文件:
```bash
#!/bin/bash
while true
do
    USERNAME=$(dialog --backtitle "Welcome" --title "login" --colors --no-shadow --stdout --nocancel --inputbox "Username:" 10 60)
    if [ x"$USERNAME" == x"" ]; then
        continue
    fi

    PASSWORD=$(dialog --backtitle "Welcome" --title "login" --colors --no-shadow --stdout --insecure --passwordbox "Password:" 10 60)

    if [ x"$PASSWORD" == x"" ]; then
        continue
    fi

    if [ x"$USERNAME" == x"root" ] && [ x"$PASSWORD" == x"123456" ]; then
        clear
        exec /bin/bash
    else
        dialog --backtitle "Welcome" --title "login" --colors --no-shadow --msgbox "Invalid username or password!" 10 60
    fi
done
```

修改`/etc/passwd`中`root`帐号的`shell`部分为`/sbin/tuilogin_shell`:
```plain
root:x:0:0:root:/root:/sbin/tuilogin_shell
```

重启系统, 可以看到执行成功:
{% img /images/2022-09-09/5.png %}

参考:

* https://itsfoss.com/what-is-tty-in-linux/
* https://www.linusakesson.net/programming/tty/index.php
* https://en.wikipedia.org/wiki/Getty_(Unix)
* https://linux.die.net/sag/login-via-terminal.html
* https://vedslinux.blogspot.com/2014/06/login-process-in-linux.html
* https://www.howtogeek.com/428174/what-is-a-tty-on-linux-and-how-to-use-the-tty-command/
* https://everyday.codes/linux/services-in-systemd-in-depth-tutorial/
* https://www.linux.com/training-tutorials/understanding-and-using-systemd/
* https://www.digitalocean.com/community/tutorials/systemd-essentials-working-with-services-units-and-the-journal
* https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units
* https://www.digitalocean.com/community/tutorials/how-to-use-journalctl-to-view-and-manipulate-systemd-logs
* https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files
* https://fedoramagazine.org/what-is-an-init-system/
* https://fedoramagazine.org/systemd-template-unit-files/
* https://www.thegeekdiary.com/centos-rhel-7-how-to-disable-all-tty-consoles-and-enable-only-1/
* https://man7.org/linux/man-pages/man8/agetty.8.html
* https://www.humbug.in/2010/utility-to-send-commands-or-data-to-other-terminals-ttypts/
