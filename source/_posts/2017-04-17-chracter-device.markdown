---
layout: post
title: "Linux字符设备驱动开发"
date: 2017-04-17 00:04:50 +0800
comments: true
categories: Kernel
---
字符设备是Linux内核中按字节顺序读写的设备。字符设备需要与`/dev`目录下的设备文件关联，应用程序通过访问设备文件与字符设备完成双向数据通信。结构如图所示:

{% img /images/2017-04-17/1.png %}

内核使用设备号来标识设备，设备号分为主设备号和次设备号。主设备号用于标识特定的驱动程序，表示设备类别，次设备号被驱动程序用于识别操作的具体设备。

<!--more-->

查看linux/types.h文件:

```c
typedef __u32 __kernel_dev_t;
…
typedef __kernel_dev_t      dev_t;
```

再查看linux/kdev_t.h文件:

```c
#define MINORBITS   20
#define MINORMASK   ((1U << MINORBITS) - 1)

#define MAJOR(dev)  ((unsigned int) ((dev) >> MINORBITS))
#define MINOR(dev)  ((unsigned int) ((dev) & MINORMASK))
#define MKDEV(ma,mi)    (((ma) << MINORBITS) | (mi))
```

可知，内核中使用32位无符号整数表示设备号，其中低20位表示次设备号、高12位表示主设备号。

设备号可以由开发者静态指定，也可以由内核动态生成。

静态指定设备号比较简单，但设备号可能已经被其他设备使用，这会导致设备号冲突。可以使用`MKDEV`宏构造设备号，之后调用`register_chrdev_region`将设备号注册到内核中。

```c
int register_chrdev_region(dev_t from, unsigned count, const char *name);
```

该API向内核注册从`from`开始的`count`个设备号。主设备号不变，次设备号递增。

由内核动态分配设备号可以避免设备号冲突:

```c
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);
```

该API请求内核分配`count`个设备号，且次设备号从`baseminor`开始。

设备号确定后，需要将设备号与字符设备关联。内核中使用cdev结构表示字符设备。字符设备需要与相应的设备文件进行关联。设备文件的操作由`file_operations`结构指定。应用程序访问设备文件时，相应系统调用会调用`file_operations`结构中的回调函数。

关联设备和设备文件操作由`cdev_init`完成:

```c
void cdev_init(struct cdev *p, const struct file_operations *fops);
```

接着需要调用`cdev_add`将设备和设备号关联到内核中。注意，`cdev_add`会立即激活设备。

```c
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
```

更古老的内核代码中使用`register_chrdev`来完成上述的所有工作:

* 分配设备号
* 关联设备与文件操作
* 关联设备与设备号

```c
int register_chrdev(unsigned int major, const char *name, const struct file_operations *fops);
```

实现设备文件操作的具体回调函数后，基本的字符设备驱动雏形就完成了。

我们的示例代码如下:

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/string.h>
#include <linux/uaccess.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("flygoast");
MODULE_DESCRIPTION("A Simple Character Device driver module");

static struct cdev cdev;
static dev_t  devno;

static char tmp[128] = "hello from kernel!";

static int
hello_open(struct inode *inodep, struct file *filep)
{
    printk(KERN_INFO "open\n");
    return 0;
}

static int
hello_release(struct inode *inodep, struct file *filep)
{
    printk(KERN_INFO "release\n");
    return 0;
}

static ssize_t
hello_read(struct file *filep, char __user *buf, size_t count, loff_t *offset)
{
    size_t  avail;

    printk(KERN_INFO "read\n");

    avail = sizeof(tmp) - *offset;

    if (count <= avail) {
        if (copy_to_user(buf, tmp + *offset, count) != 0) {
            return -EFAULT;
        }

        *offset += count;
        return count;

    } else {
        if (copy_to_user(buf, tmp + *offset, avail) != 0) {
            return -EFAULT;
        }

        *offset += avail;
        return avail;
    }
}

static ssize_t
hello_write(struct file *filep, const char __user *buf, size_t count,
            loff_t *offset)
{
    size_t  avail;

    printk(KERN_INFO "write\n");

    avail = sizeof(tmp) - *offset;

    memset(tmp + *offset, 0, avail);

    if (count > avail) {
        if (copy_from_user(tmp + *offset, buf, avail) != 0) {
            return -EFAULT;
        }
        *offset += avail;
        return avail;

    } else {
        if (copy_from_user(tmp + *offset, buf, count) != 0) {
            return -EFAULT;
        }
        *offset += count;
        return count;
    }
}

static loff_t
hello_llseek(struct file *filep, loff_t off, int whence)
{
    loff_t  newpos;

    switch (whence) {
    case 0: /* SEEK_SET */
        newpos = off;
        break;
    case 1: /* SEEK_CUR */
        newpos = filep->f_pos + off;
        break;
    case 2: /* SEEK_END */
        newpos = sizeof(tmp) + off;
        break;
    default:
        return -EINVAL;
    }

    if (newpos < 0) {
        return -EINVAL;
    }

    filep->f_pos = newpos;
    return newpos;
}

static const struct file_operations  fops = {
    .owner = THIS_MODULE,
    .open = hello_open,
    .release = hello_release,
    .read = hello_read,
    .llseek = hello_llseek,
    .write = hello_write,
};

static int __init hello_init(void) {
    int    ret;

    printk(KERN_INFO "Load hello\n");

    devno = MKDEV(111, 0);
    ret = register_chrdev_region(devno, 1, "hello");

    if (ret < 0) {
        return ret;
    }

    cdev_init(&cdev, &fops);
    cdev.owner = THIS_MODULE;

    cdev_add(&cdev, devno, 1);

    return 0;
}

static void __exit hello_cleanup(void) {
    printk(KERN_INFO "cleanup hello\n");
    unregister_chrdev_region(devno, 1);
    cdev_del(&cdev);
}

module_init(hello_init);
module_exit(hello_cleanup);
```

Makefile文件内容:

```makefile
obj-m += hello.o
all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

应用程序代码app.c如下:

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <fcntl.h>

char buf[128];

int main()
{
    int fd, m, n;
    fd = open("/dev/hello", O_RDWR);
    if (fd < 0) {
        fprintf(stderr, "open file \"/dev/hello\" failed\n");
        exit(-1);
    }

    printf("read: ");
    while ((m = read(fd, buf, 1)) > 0 && buf[0] != '\0') {
        printf("%c", buf[0]);
    }
    printf("\n");

    llseek(fd, 0, 0);

    n = write(fd, "hello from user1!", 17);
    printf("write length: %d\n", n);
    n = write(fd, " ", 1);
    printf("write length: %d\n", n);
    n = write(fd, "hello from user2!", 17);
    printf("write length: %d\n", n);

    close(fd);
    return 0;
}
```
编译生成内核模块并加载:
```bash
insmod ./hello.ko
```
查看设备:
```plain
[root@localhost hello]# cat /proc/devices |grep hello
111 hello
```
确定设备驱动已经加载, 设备已经存在。

这时设备文件/dev/hello并不存在，我们需要使用`mknod`来创建:
```bash
mknod /dev/hello c 111 0
```
编译应用程序:
```bash
gcc app.c -o app
```
执行两次应用程序:
```plain
[root@localhost hello]# ./app
read: hello from kernel!
write length: 17
write length: 1
write length: 17
[root@localhost hello]# ./app
read: hello from user1! hello from user2!
write length: 17
write length: 1
write length: 17
```
第一次程序从设备文件读取到的是驱动程序BUFFER内的初始值，第二次读到的则是第一次应用程序执行时写入的字符串。

由于像上述例子中加载驱动后再手动创建设备文件比较烦琐，Linux内核又提供了`udev`的API可以在加载驱动时自动创建设备文件，并在卸载时自动删除设备文件。修改后的`init`和`cleanup`函数为:
```c
static int __init hello_init(void) {
    int    ret;

    printk(KERN_INFO "Load hello\n");

    devno = MKDEV(111, 0);
    ret = register_chrdev_region(devno, 1, "hello");

    if (ret < 0) {
        return ret;
    }

    cdev_init(&cdev, &fops);
    cdev.owner = THIS_MODULE;

    cdev_add(&cdev, devno, 1);

    class = class_create(THIS_MODULE, "hello");
    if (IS_ERR(class)) {
        unregister_chrdev_region(devno, 1);
        return PTR_ERR(class);
    }

    device = device_create(class, NULL, devno, NULL, "hello");
    if (IS_ERR(device)) {
        class_destroy(class);
        unregister_chrdev_region(devno, 1);
        return PTR_ERR(device);
    }
    return 0;
}

static void __exit hello_cleanup(void) {
    printk(KERN_INFO "cleanup hello\n");

    device_destroy(class, devno);
    class_unregister(class);
    class_destroy(class);

    unregister_chrdev_region(devno, 1);
    cdev_del(&cdev);
}
```

资源清除和卸载相关的API在文中没有详细介绍，可以参考`Linux Cross Reference`。
