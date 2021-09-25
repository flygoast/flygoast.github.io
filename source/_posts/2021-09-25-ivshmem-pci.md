---
title: ivshmem PCI设备中断机制驱动示例
date: 2021-09-25 19:00:59
tags:
- virtualization
- ivshmem
- qemu
- pci
categories: Virtualization
description:
---
之前的文章[<<QEMU虚拟机内识别ivshmem设备>>](/2021/09/12/qemu-ivshmem/)介绍了在虚拟机内通过用户态程序访问`ivshmem`设备的共享内存。在虚拟机之间或者宿主机与虚拟机之间通过共享内存进行通信的情形下，共享内存的两端必须依赖轮询方式来实现通知机制。这种方式是`ivshmem`提供的`ivshmem-plain`的使用方式。除此之外，`ivshmem`还提供了`ivshmem-doorbell`的使用方式，它提供了基于中断的通知机制。

`ivshmem-doorbell`提供了两种中断方式，一种是传统的基于`INTx`的中断, 它主要使用`BAR0`的`Interrupt Mask`和`Interrupt Status`两个寄存器；另一种是基于`MSI-X`的中断，它主要使用`BAR0`的`IVPosition`和`Doorbell`两个寄存器。参考共享的设备端叫做`peer`。`IVPosition`寄存器存储该`peer`的数字标识符(0-65535), 称做`peer_id`。该寄存器为只读寄存器。而`Doorbell`寄存器为只写寄存器。`ivshmem-doorbell`支持多个中断向量，写入`Doorbell`寄存器则触发共享该内存的某个`peer`的某个中断。`Doorbell`为`32`位，低`16`位为`peer_id`，而高`16`位为中断向量号(这里是从`0`开始的顺序号，而非`PCI`驱动在Guest虚拟机内部所申请的向量号)。


使用`ivshmem-doorbell`机制需要运行`ivshmem-server`。`ivshmem-server`根据参数创建共享内存，并通过监听本地`UNIX DOMAIN SOCKET`等待共享内存的`peer`来连接。添加了`ivshmem-doorbell`设备的`QEMU`进程会连接该`socket`, 从而获取`ivshmem-server`所分配的一个`peer_id`。`ivshmem-doorbell`支持多个中断向量，`ivshmem-server`会为`ivshmem`虚拟`PCI`设备支持的每个中断向量创建一个`eventfd`，并将共享内存以及为所有客户端中断向量所创建的`eventfd`都通过`SCM_RIGHTS`机制传递给所有客户端进程。这样所有的`peer`便都具备了独立的两两之间的通知通道。之后在虚拟机内通过触发`ivshmem`虚拟`PCI`设备的`DOORBELL`寄存器的写入，虚拟机的`QEMU`进程便会通过`DOORBELL`寄存器中的`peer_id`和中断向量号来找到相应的`eventfd`，从而通知到对端的`QEMU`进程来产生相应的`PCI`中断。


要使用中断机制，用户态程序是无能为力的，需要编写相应的`PCI`驱动来实现。本文通过一个简单的`PCI`驱动示例来说明`ivshmem-doorbell`的`MSI-X`中断机制的使用。

`ivpci.c`代码如下:

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/version.h>
#include <linux/err.h>
#include <linux/fs.h>
#include <linux/mm.h>
#include <linux/pci.h>
#include <linux/pci_regs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/interrupt.h>
#include <linux/ioctl.h>
#include <linux/sched.h>
#include <linux/wait.h>


#define DRV_NAME        "ivpci"
#define DRV_VERSION     "0.1"
#define PFX             "[IVPCI] "
#define DRV_FILE_FMT    DRV_NAME"%d"

#define IVPOSITION_OFF  0x08    /* VM ID */
#define DOORBELL_OFF    0x0c    /* Doorbell */

#define IOCTL_MAGIC         ('f')
#define IOCTL_RING          _IOW(IOCTL_MAGIC, 1, u32)
#define IOCTL_WAIT          _IO(IOCTL_MAGIC, 2)
#define IOCTL_IVPOSITION    _IOR(IOCTL_MAGIC, 3, u32)


static int g_max_devices = 2;
MODULE_PARM_DESC(g_max_devices, "number of devices can be supported");
module_param(g_max_devices, int, 0400);


struct ivpci_private {
    struct pci_dev      *dev;
    struct cdev         cdev;
    int                 minor;

    u8                  revision;
    u32                 ivposition;

    u8 __iomem          *base_addr;
    u8 __iomem          *regs_addr;

    unsigned int        bar0_addr;
    unsigned int        bar0_len;
    unsigned int        bar1_addr;
    unsigned int        bar1_len;
    unsigned int        bar2_addr;
    unsigned int        bar2_len;

    char                (*msix_names)[256];
    struct msix_entry   *msix_entries;
    int                 nvectors;
};



static int event_toggle;
DECLARE_WAIT_QUEUE_HEAD(wait_queue);


/* store major device number shared by all ivshmem devices */
static dev_t g_ivpci_devno;
static int  g_ivpci_major;

static struct class *g_ivpci_class;

/* number of devices owned by this driver */
static int g_ivpci_count;

static struct ivpci_private *g_ivpci_devs;


static struct pci_device_id ivpci_id_table[] = {
    { 0x1af4, 0x1110, PCI_ANY_ID, PCI_ANY_ID, 0, 0, 0},
    { 0 },
};
MODULE_DEVICE_TABLE(pci, ivpci_id_table);


static struct ivpci_private *ivpci_get_private(void)
{
    int i;

    for (i = 0; i < g_max_devices; i++) {
        if (g_ivpci_devs[i].dev == NULL) {
            return &g_ivpci_devs[i];
        }
    }

    return NULL;
}

static struct ivpci_private *ivpci_find_private(int minor)
{
    int i;
    for (i = 0; i < g_max_devices; i++) {
        if (g_ivpci_devs[i].dev != NULL && g_ivpci_devs[i].minor == minor) {
            return &g_ivpci_devs[i];
        }
    }

    return NULL;
}


static irqreturn_t ivpci_interrupt(int irq, void *dev_id)
{
    struct ivpci_private *ivpci_dev = dev_id;

    if (unlikely(ivpci_dev == NULL)) {
        return IRQ_NONE;
    }

    dev_info(&ivpci_dev->dev->dev, PFX "interrupt: %d\n", irq);

    event_toggle = 1;
    wake_up_interruptible(&wait_queue);

    return IRQ_HANDLED;
}


static int ivpci_request_msix_vectors(struct ivpci_private *ivpci_dev, int n)
{
    int ret, i;

    ret = -EINVAL;

    dev_info(&ivpci_dev->dev->dev, PFX "request msi-x vectors: %d\n", n);

    ivpci_dev->nvectors = n;

    ivpci_dev->msix_entries = kmalloc(n * sizeof(struct msix_entry),
            GFP_KERNEL);
    if (ivpci_dev->msix_entries == NULL) {
        ret = -ENOMEM;
        goto error;
    }

    ivpci_dev->msix_names = kmalloc(n * sizeof(*ivpci_dev->msix_names),
            GFP_KERNEL);
    if (ivpci_dev->msix_names == NULL) {
        ret = -ENOMEM;
        goto free_entries;
    }

    for (i = 0; i < n; i++) {
        ivpci_dev->msix_entries[i].entry = i;
    }

    ret = pci_enable_msix_exact(ivpci_dev->dev, ivpci_dev->msix_entries, n);
    if (ret) {
        dev_err(&ivpci_dev->dev->dev, PFX "unable to enable msix: %d\n", ret);
        goto free_names;
    }

    for (i = 0; i < ivpci_dev->nvectors; i++) {
        snprintf(ivpci_dev->msix_names[i], sizeof(*ivpci_dev->msix_names),
                "%s%d-%d", DRV_NAME, ivpci_dev->minor, i);

        ret = request_irq(ivpci_dev->msix_entries[i].vector,
                ivpci_interrupt, 0, ivpci_dev->msix_names[i], ivpci_dev);

        if (ret) {
            dev_err(&ivpci_dev->dev->dev, PFX "unable to allocate irq for " \
                    "msix entry %d with vector %d\n", i,
                    ivpci_dev->msix_entries[i].vector);
            goto release_irqs;
        }

        dev_info(&ivpci_dev->dev->dev,
                PFX "irq for msix entry: %d, vector: %d\n",
                i, ivpci_dev->msix_entries[i].vector);
    }

    return 0;

release_irqs:
    for ( ; i > 0; i--) {
        free_irq(ivpci_dev->msix_entries[i - 1].vector, ivpci_dev);
    }
    pci_disable_msix(ivpci_dev->dev);

free_names:
    kfree(ivpci_dev->msix_names);

free_entries:
    kfree(ivpci_dev->msix_entries);

error:
    return ret;
}

static void ivpci_free_msix_vectors(struct ivpci_private *ivpci_dev)
{
    int i;

    for (i = ivpci_dev->nvectors; i > 0; i--) {
        free_irq(ivpci_dev->msix_entries[i - 1].vector, ivpci_dev);
    }
    pci_disable_msix(ivpci_dev->dev);

    kfree(ivpci_dev->msix_names);
    kfree(ivpci_dev->msix_entries);
}


static int ivpci_open(struct inode *inode, struct file *filp)
{
    int minor = iminor(inode);
    struct ivpci_private *ivpci_dev;

    ivpci_dev = ivpci_find_private(minor);
    filp->private_data = (void *) ivpci_dev;
    BUG_ON(filp->private_data == NULL);

    dev_info(&ivpci_dev->dev->dev, PFX "open ivpci\n");

    return 0;
}


static int ivpci_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;
    unsigned long len;
    unsigned long off;
    unsigned long start;
    struct ivpci_private *ivpci_dev;

    ivpci_dev = (struct ivpci_private *)filp->private_data;
    BUG_ON(ivpci_dev == NULL);
    BUG_ON(ivpci_dev->base_addr == NULL);

    dev_info(&ivpci_dev->dev->dev, PFX "mmap ivpci bar2\n");

    /* `vma->vm_start` and `vma->vm_end` had aligned to page size */
    WARN_ON(offset_in_page(vma->vm_start));
    WARN_ON(offset_in_page(vma->vm_end));

    off = vma->vm_pgoff << PAGE_SHIFT;
    start = ivpci_dev->bar2_addr;

    /* Align up to page size. */
    len = PAGE_ALIGN((start & ~PAGE_MASK) + ivpci_dev->bar2_len);
    start &= PAGE_MASK;

    dev_info(&ivpci_dev->dev->dev, PFX "mmap vma pgoff: %lu, 0x%0lx - 0x%0lx," \
            " aligned length: %lu\n", vma->vm_pgoff, vma->vm_start,
            vma->vm_end, len);

    if (vma->vm_end - vma->vm_start + off > len) {
        dev_err(&ivpci_dev->dev->dev,
                PFX "mmap overflow the end, %lu - %lu + %lu > %lu",
                vma->vm_end, vma->vm_start, off, len);
        ret = -EINVAL;
        goto error;
    }

    off += start;
    vma->vm_pgoff = off >> PAGE_SHIFT;
    vma->vm_flags |= VM_IO|VM_SHARED|VM_DONTEXPAND|VM_DONTDUMP;

    if (io_remap_pfn_range(vma, vma->vm_start, off >> PAGE_SHIFT,
                vma->vm_end - vma->vm_start, vma->vm_page_prot)) {
        dev_err(&ivpci_dev->dev->dev, PFX "mmap bar2 failed\n");
        ret = -ENXIO;
        goto error;
    }

    ret = 0;

error:
    return ret;
}


static ssize_t ivpci_read(struct file *filp, char *buffer, size_t len,
        loff_t *poffset)
{
    int bytes_read = 0;
    unsigned long offset;
    struct ivpci_private *ivpci_dev;

    ivpci_dev = (struct ivpci_private *)filp->private_data;

    BUG_ON(ivpci_dev == NULL);
    BUG_ON(ivpci_dev->base_addr == NULL);


    offset = *poffset;
    dev_info(&ivpci_dev->dev->dev, PFX "read ivpci bar2, offset: %lu\n",
            offset);

    /* beyoud the end */
    if (len > ivpci_dev->bar2_len - offset) {
        len = ivpci_dev->bar2_len - offset;
    }

    if (len == 0) {
        return 0;
    }

    bytes_read = copy_to_user(buffer, (char *)ivpci_dev->base_addr + offset,
            len);
    if (bytes_read > 0) {
        dev_err(&ivpci_dev->dev->dev,
                PFX "read ivpci bar2, copy_to_user() failed: %d\n", bytes_read);
        return -EFAULT;
    }

    *poffset += len;
    return len;
}


static long ivpci_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct ivpci_private *ivpci_dev;
    u16 ivposition;
    u16 vector;
    u32 value;

    ivpci_dev = (struct ivpci_private *)filp->private_data;

    BUG_ON(ivpci_dev == NULL);
    BUG_ON(ivpci_dev->base_addr == NULL);

    switch (cmd) {
    case IOCTL_RING:
        if (copy_from_user(&value, (u32 *) arg, sizeof(value)) ) {
            dev_err(&ivpci_dev->dev->dev, PFX "copy from user failed");
            return -1;
        }
        vector = value & 0xffff;
        ivposition = value & 0xffff0000 >> 16;
        dev_info(&ivpci_dev->dev->dev,
                PFX "ring doorbell: value: %u(0x%x), vector: %u, peer id: %u\n",
                value, value, vector, ivposition);
        writel(value & 0xffffffff, ivpci_dev->regs_addr + DOORBELL_OFF);
        break;

    case IOCTL_WAIT:
        dev_info(&ivpci_dev->dev->dev, PFX "wait for interrupt\n");
        ret = wait_event_interruptible(wait_queue, (event_toggle == 1));
        if (ret == 0) {
            dev_info(&ivpci_dev->dev->dev, PFX "wakeup\n");
            event_toggle = 0;
        } else if (ret == -ERESTARTSYS) {
            dev_err(&ivpci_dev->dev->dev, PFX "interrupted by signal\n");
            return ret;
        } else {
            dev_err(&ivpci_dev->dev->dev, PFX "unknown failed: %d\n", ret);
            return ret;
        }
        break;

    case IOCTL_IVPOSITION:
        dev_info(&ivpci_dev->dev->dev, PFX "get ivposition: %u\n",
                ivpci_dev->ivposition);
        if (copy_to_user((u32 *) arg, &ivpci_dev->ivposition, sizeof(u32))) {
            dev_err(&ivpci_dev->dev->dev, PFX "copy to user failed");
            return -1;
        }
        break;

    default:
        dev_err(&ivpci_dev->dev->dev, PFX "bad ioctl command: %d\n", cmd);
        return -1;
    }

    return 0;
}


static long ivpci_write(struct file *filp, const char *buffer, size_t len,
        loff_t *poffset)
{
    int bytes_written = 0;
    unsigned long offset;
    struct ivpci_private *ivpci_dev;

    ivpci_dev = (struct ivpci_private *)filp->private_data;

    BUG_ON(ivpci_dev == NULL);
    BUG_ON(ivpci_dev->base_addr == NULL);

    dev_info(&ivpci_dev->dev->dev, PFX "write ivpci bar2\n");

    offset = *poffset;

    if (len > ivpci_dev->bar2_len - offset) {
        len = ivpci_dev->bar2_len - offset;
    }

    if (len == 0) {
        return 0;
    }

    bytes_written = copy_from_user(ivpci_dev->base_addr + offset, buffer, len);
    if (bytes_written > 0) {
        return -EFAULT;
    }

    *poffset += len;
    return len;
}


static loff_t ivpci_lseek(struct file *filp, loff_t offset, int origin)
{
    loff_t retval = -1;
    struct ivpci_private *ivpci_dev;

    ivpci_dev = (struct ivpci_private *)filp->private_data;

    BUG_ON(ivpci_dev == NULL);
    BUG_ON(ivpci_dev->base_addr == NULL);

    dev_info(&ivpci_dev->dev->dev,
            PFX "lseek ivpci bar2, offset: %llu, origin: %d\n", offset, origin);

    switch (origin) {
    case 1:
        offset += filp->f_pos;
        /* fall through */
    case 0:
        retval = offset;
        if (offset > ivpci_dev->bar2_len) {
            offset = ivpci_dev->bar2_len;
        }
        filp->f_pos = offset;
    }
    return retval;
}


static int ivpci_release(struct inode *inode, struct file *filp)
{
    struct ivpci_private *ivpci_dev;
    ivpci_dev = (struct ivpci_private *)filp->private_data;

    BUG_ON(ivpci_dev == NULL);
    BUG_ON(ivpci_dev->base_addr == NULL);

    dev_info(&ivpci_dev->dev->dev, PFX "release ivpci\n");

    return 0;
}


static struct file_operations ivpci_ops = {
    .owner          = THIS_MODULE,
    .open           = ivpci_open,
    .mmap           = ivpci_mmap,
    .unlocked_ioctl = ivpci_ioctl,
    .read           = ivpci_read,
    .write          = ivpci_write,
    .llseek         = ivpci_lseek,
    .release        = ivpci_release,
};


static int ivpci_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    int ret;
    struct ivpci_private *ivpci_dev;
    dev_t devno;

    dev_info(&pdev->dev, PFX "probing for device: %s\n", pci_name(pdev));

    if (g_ivpci_count >= g_max_devices) {
        dev_err(&pdev->dev, PFX "reach the maxinum number of devices, " \
                "please adapt the `g_max_devices` value, reload the driver\n");
        ret = -1;
        goto out;
    }

    ret = pci_enable_device(pdev);
    if (ret < 0) {
        dev_err(&pdev->dev, PFX "unable to enable device: %d\n", ret);
        goto out;
    }

    /* Reserved PCI I/O and memory resources for this device */
    ret = pci_request_regions(pdev, DRV_NAME);
    if (ret < 0) {
        dev_err(&pdev->dev, PFX "unable to reserve resources: %d\n", ret);
        goto disable_device;
    }

    ivpci_dev = ivpci_get_private();
    BUG_ON(ivpci_dev == NULL);

    pci_read_config_byte(pdev, PCI_REVISION_ID, &ivpci_dev->revision);

    dev_info(&pdev->dev, PFX "device %d:%d, revision: %d\n", g_ivpci_major,
            ivpci_dev->minor, ivpci_dev->revision);

    /* Pysical address of BAR0, BAR1, BAR2 */
    ivpci_dev->bar0_addr = pci_resource_start(pdev, 0);
    ivpci_dev->bar0_len = pci_resource_len(pdev, 0);
    ivpci_dev->bar1_addr = pci_resource_start(pdev, 1);
    ivpci_dev->bar1_len = pci_resource_len(pdev, 1);
    ivpci_dev->bar2_addr = pci_resource_start(pdev, 2);
    ivpci_dev->bar2_len = pci_resource_len(pdev, 2);

    dev_info(&pdev->dev, PFX "BAR0: 0x%0x, %d\n", ivpci_dev->bar0_addr,
            ivpci_dev->bar0_len);
    dev_info(&pdev->dev, PFX "BAR1: 0x%0x, %d\n", ivpci_dev->bar1_addr,
            ivpci_dev->bar1_len);
    dev_info(&pdev->dev, PFX "BAR2: 0x%0x, %d\n", ivpci_dev->bar2_addr,
            ivpci_dev->bar2_len);

    ivpci_dev->regs_addr = ioremap(ivpci_dev->bar0_addr, ivpci_dev->bar0_len);
    if (!ivpci_dev->regs_addr) {
        dev_err(&pdev->dev, PFX "unable to ioremap bar0, size: %d\n",
                ivpci_dev->bar0_len);
        goto release_regions;
    }

    ivpci_dev->base_addr = ioremap(ivpci_dev->bar2_addr, ivpci_dev->bar2_len);
    if (!ivpci_dev->base_addr) {
        dev_err(&pdev->dev, PFX "unable to ioremap bar2, size: %d\n",
                ivpci_dev->bar2_len);
        goto iounmap_bar0;
    }
    dev_info(&pdev->dev, PFX "BAR2 map: %p\n", ivpci_dev->base_addr);

    /*
     * Create character device file.
     */
    cdev_init(&ivpci_dev->cdev, &ivpci_ops);
    ivpci_dev->cdev.owner = THIS_MODULE;

    devno = MKDEV(g_ivpci_major, ivpci_dev->minor);
    ret = cdev_add(&ivpci_dev->cdev, devno, 1);
    if (ret < 0) {
        dev_err(&pdev->dev, PFX "unable to add chrdev %d:%d to system: %d\n",
                g_ivpci_major, ivpci_dev->minor, ret);
        goto iounmap_bar2;
    }

    if (device_create(g_ivpci_class, NULL, devno, NULL, DRV_FILE_FMT,
                ivpci_dev->minor) == NULL)
    {
        dev_err(&pdev->dev, PFX "unable to create device file: %d:%d\n",
                g_ivpci_major, ivpci_dev->minor);
        goto delete_chrdev;
    }

    ivpci_dev->dev = pdev;
    pci_set_drvdata(pdev, ivpci_dev);

    if (ivpci_dev->revision == 1) {
        /* Only process the MSI-X interrupt. */
        ivpci_dev->ivposition = ioread32(ivpci_dev->regs_addr + IVPOSITION_OFF);

        dev_info(&pdev->dev, PFX "device ivposition: %u, MSI-X: %s\n",
                ivpci_dev->ivposition,
                (ivpci_dev->ivposition == 0) ? "no": "yes");

        if (ivpci_dev->ivposition != 0) {
            ret = ivpci_request_msix_vectors(ivpci_dev, 4);
            if (ret != 0) {
                goto destroy_device;
            }
        }
    }

    g_ivpci_count++;
    dev_info(&pdev->dev, PFX "device probed: %s\n", pci_name(pdev));
    return 0;

destroy_device:
    devno = MKDEV(g_ivpci_major, ivpci_dev->minor);
    device_destroy(g_ivpci_class, devno);
    ivpci_dev->dev = NULL;

delete_chrdev:
    cdev_del(&ivpci_dev->cdev);

iounmap_bar2:
    iounmap(ivpci_dev->base_addr);

iounmap_bar0:
    iounmap(ivpci_dev->regs_addr);

release_regions:
    pci_release_regions(pdev);

disable_device:
    pci_disable_device(pdev);

out:
    pci_set_drvdata(pdev, NULL);
    return ret;
}


static void ivpci_remove(struct pci_dev *pdev)
{
    int devno;
    struct ivpci_private *ivpci_dev;

    dev_info(&pdev->dev, PFX "removing ivshmem device: %s\n", pci_name(pdev));

    ivpci_dev = pci_get_drvdata(pdev);
    BUG_ON(ivpci_dev == NULL);

    ivpci_free_msix_vectors(ivpci_dev);

    ivpci_dev->dev = NULL;

    devno = MKDEV(g_ivpci_major, ivpci_dev->minor);
    device_destroy(g_ivpci_class, devno);

    cdev_del(&ivpci_dev->cdev);

    iounmap(ivpci_dev->base_addr);
    iounmap(ivpci_dev->regs_addr);

    pci_release_regions(pdev);
    pci_disable_device(pdev);
    pci_set_drvdata(pdev, NULL);
}


static struct pci_driver ivpci_driver = {
    .name       = DRV_NAME,
    .id_table   = ivpci_id_table,
    .probe      = ivpci_probe,
    .remove     = ivpci_remove,
};


static int __init ivpci_init(void)
{
    int ret, i, minor;

    pr_info(PFX "*********************************************************\n");
    pr_info(PFX "module loading\n");

    ret = alloc_chrdev_region(&g_ivpci_devno, 0, g_max_devices, DRV_NAME);
    if (ret < 0) {
        pr_err(PFX "unable to allocate major number: %d\n", ret);
        goto out;
    }

    g_ivpci_devs = kzalloc(sizeof(struct ivpci_private) * g_max_devices,
            GFP_KERNEL);
    if (g_ivpci_devs == NULL) {
        goto unregister_chrdev;
    }

    minor = MINOR(g_ivpci_devno);
    for (i = 0; i < g_max_devices; i++) {
        g_ivpci_devs[i].minor = minor++;
    }

    g_ivpci_class = class_create(THIS_MODULE, DRV_NAME);
    if (g_ivpci_class == NULL) {
        pr_err(PFX "unable to create the struct class\n");
        goto free_devs;
    }

    g_ivpci_major = MAJOR(g_ivpci_devno);
    pr_info(PFX "major: %d, minor: %d\n", g_ivpci_major, MINOR(g_ivpci_devno));

    ret = pci_register_driver(&ivpci_driver);
    if (ret < 0) {
        pr_err(PFX "unable to register driver: %d\n", ret);
        goto destroy_class;
    }

    pr_info(PFX "module loaded\n");
    return 0;

destroy_class:
    class_destroy(g_ivpci_class);

free_devs:
    kfree(g_ivpci_devs);

unregister_chrdev:
    unregister_chrdev_region(g_ivpci_devno, g_max_devices);

out:
    return -1;
}

static void __exit ivpci_exit(void)
{
    pci_unregister_driver(&ivpci_driver);

    class_destroy(g_ivpci_class);

    kfree(g_ivpci_devs);

    unregister_chrdev_region(g_ivpci_devno, g_max_devices);

    pr_info(PFX "module unloaded\n");
    pr_info(PFX "*********************************************************\n");
}


/************************************************
 * Just for eliminating the compiling warnings.
 ************************************************/
#define my_module_init(initfn)					\
	static inline initcall_t __inittest(void)		\
	{ return initfn; }					\
	int init_module(void) __cold __attribute__((alias(#initfn)));

#define my_module_exit(exitfn)					\
	static inline exitcall_t __exittest(void)		\
	{ return exitfn; }					\
	void cleanup_module(void) __cold __attribute__((alias(#exitfn)));

my_module_init(ivpci_init);
my_module_exit(ivpci_exit);

MODULE_AUTHOR("flygoast@126.com");
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Demo PCI driver for ivshmem device");
MODULE_VERSION(DRV_VERSION);
```

简单解释一下代码逻辑。驱动被加载时，会调用`ivpci_init`。我们希望所有的`ivshmem`设备使用相同的设备`major`号，因而在这里调用`alloc_chrdev_region`申请足够的`major`和`minor`号，接着创建出足够的`ivshmem`设备所需的数据结构`struct ivpci_private`，并将`minor`号存储在这些结构中。接着，创建设备文件`class`, 然后注册`ivshmem`设备的驱动结构。

当`ivshmem`设备被检测到，`probe`函数`ivpci_probe`会被调用。我们调用`pci_enable_device`启用该设备，并将`PCI`设备的3个`BAR`空间映射到内存空间，方便后续访问。然后创建对应的设备文件，给用户态程序提供访问接口。

`QEMU-2.6.0`之后，`ivshmem`虚拟`PCI`设备配置空间的`revision`字段为`1`，这样才支持`MSI-X`中断机制，所以我们只处理`revision`为`1`的`ivshmem`设备。

接下来，函数`ivpci_request_msix_vectors`调用`pci_enable_msix`分配中断向量，然后依次调用`request_irq`注册我们的中断处理程序:`ivpci_interrupt`。 


示例程序里简单使用`waitqueue`来实现等待和通知。当用户态程序以`IOCTL_WAIT`命令调用`ioctl`时，就让该进程在`waitqueue`上进行等待。当中断触发时，我们的中断处理程序会调用`wake_up_interruptible`唤醒`waitqueue`上所等待的进程。

设备文件的文件操作的`mmap`实现可以把设备`BAR2`空间映射到用户空间，与之前的文章[<<QEMU虚拟机内识别ivshmem设备>>](/2021/09/12/qemu-ivshmem/)中介绍的用户态程序直接`mmap` `sysfs`下的`resource2`文件作用是一致的。


示例的用户态代码，`ioctl.c`如下:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/types.h>
#include <assert.h>


#define IOCTL_MAGIC         ('f')
#define IOCTL_RING          _IOW(IOCTL_MAGIC, 1, __u32)
#define IOCTL_WAIT          _IO(IOCTL_MAGIC, 2)
#define IOCTL_IVPOSITION    _IOR(IOCTL_MAGIC, 3, __u32)

static void usage(void)
{
    printf("Usage: \n"  \
           "  ioctl <devfile> ivposition\n"    \
           "  ioctl <devfile> wait\n"          \
           "  ioctl <devfile> ring <peer_id> <vector_id>\n");
}


int main(int argc, char **argv)
{
    int fd, arg, ret, vector, peer;

    if (argc < 3) {
        usage();
        return -1;
    }

    fd = open(argv[1], O_RDWR);
    assert(fd != -1);

    ret = ioctl(fd, IOCTL_IVPOSITION, &peer);
    printf("IVPOSITION: %d\n", peer);

    if (strcmp(argv[2], "ivposition") == 0) {
        return 0;

    } if (strcmp(argv[2], "wait") == 0) {
        if (argc != 3) {
            usage();
            return -1;
        }
        printf("wait:\n");
        ret = ioctl(fd, IOCTL_WAIT, &arg);

    } else if (strcmp(argv[2], "ring") == 0) {
        if (argc != 5) {
            usage();
            return -1;
        }
        printf("ring:\n");
        peer = atoi(argv[3]);
        vector = atoi(argv[4]);

        arg = ((peer & 0xffff) << 16) | (vector & 0xffff);
        printf("arg: 0x%x\n", arg);

        ret = ioctl(fd, IOCTL_RING, &arg);

    } else {
        printf("Invalid command: %s\n", argv[2]);
        usage();
        return -1;
    }

    printf("ioctl finished, returned value: %d\n", ret);

    close(fd);

    return 0;
}
```

`Makefile`如下:
```c
CONFIG_MODULE_SIG=n

obj-m += ivpci.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
	gcc -g -O0 ioctl.c -o ioctl
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
	rm -rf ioctl
```

编译执行:
```bash
make
```

接下来，在宿主机上运行`ivshmem-server`:
```bash
sudo -u qemu /tmp/ivshmem-server -l 4M -M fg-doorbell -n 8 -F -v
```

由于`ivshmem-server`分配的第一个`peer_id`为`0`, 而当`ivshmem`设备的`ivposition`为`0`时，无法判断是不支持`MSI-X`中断还是分配的`peer_id`为`0`。所以我们可以先调用`ivshmem-client`程序来先连接`ivshmem-server`来占据`0`号`peer_id`:

```plain
[root@controller qemu-2.6.0]# ./ivshmem-client
dump: dump peers (including us)
int <peer> <vector>: notify one vector on a peer
int <peer> all: notify all vectors of a peer
int all: notify all vectors of all peers (excepting us)
cmd> listen on server socket 3
dump
our_id = 0
  vector 0 is enabled (fd=5)
  vector 1 is enabled (fd=6)
  vector 2 is enabled (fd=7)
  vector 3 is enabled (fd=8)
  vector 4 is enabled (fd=9)
  vector 5 is enabled (fd=10)
  vector 6 is enabled (fd=11)
  vector 7 is enabled (fd=12)
cmd>
```

接下来启动两台虚拟机，分别给两台虚拟机各动态添加一个`ivshmem-plain`和一个`ivshmem-doorbell`设备:
```bash
virsh qemu-monitor-command --hmp 4 "object_add  memory-backend-file,size=8M,share,mem-path=/dev/shm/shm2,id=shm2"
virsh qemu-monitor-command --hmp 4 "device_add  ivshmem-plain,memdev=shm2,bus=pci.0,addr=0x1f,master=on"
virsh qemu-monitor-command --hmp 4 "chardev-add socket,path=/tmp/ivshmem_socket,id=fg-doorbell"
virsh qemu-monitor-command --hmp 4 "device_add ivshmem-doorbell,chardev=fg-doorbell,vectors=8"
```

在两台虚拟机中都装载驱动模块:
```bash
insmod ./ivpci.ko
```

`dmesg`查看加载信息, 如下:
```
[ 7204.791580] [IVPCI] *********************************************************
[ 7204.791583] [IVPCI] module loading
[ 7204.791592] [IVPCI] major: 244, minor: 0
[ 7204.791613] ivpci 0000:00:1f.0: [IVPCI] probing for device: 0000:00:1f.0
[ 7204.791677] ivpci 0000:00:1f.0: [IVPCI] device 244:0, revision: 1
[ 7204.791680] ivpci 0000:00:1f.0: [IVPCI] BAR0: 0xfebd3000, 256
[ 7204.791682] ivpci 0000:00:1f.0: [IVPCI] BAR1: 0x0, 0
[ 7204.791684] ivpci 0000:00:1f.0: [IVPCI] BAR2: 0xfe000000, 8388608
[ 7204.791696] ivpci 0000:00:1f.0: [IVPCI] BAR2 map: ffffade482000000
[ 7204.792366] ivpci 0000:00:1f.0: [IVPCI] device ivposition: 0, MSI-X: no
[ 7204.792369] ivpci 0000:00:1f.0: [IVPCI] device probed: 0000:00:1f.0
[ 7204.792379] ivpci 0000:00:06.0: [IVPCI] probing for device: 0000:00:06.0
[ 7204.792414] ivpci 0000:00:06.0: [IVPCI] device 244:1, revision: 1
[ 7204.792416] ivpci 0000:00:06.0: [IVPCI] BAR0: 0x80401000, 256
[ 7204.792418] ivpci 0000:00:06.0: [IVPCI] BAR1: 0x80400000, 4096
[ 7204.792420] ivpci 0000:00:06.0: [IVPCI] BAR2: 0x80000000, 4194304
[ 7204.792429] ivpci 0000:00:06.0: [IVPCI] BAR2 map: ffffade481800000
[ 7204.796436] ivpci 0000:00:06.0: [IVPCI] device ivposition: 1, MSI-X: yes
[ 7204.796440] ivpci 0000:00:06.0: [IVPCI] request msi-x vectors: 4
[ 7204.796526] ivpci 0000:00:06.0: irq 29 for MSI/MSI-X
[ 7204.796579] ivpci 0000:00:06.0: irq 30 for MSI/MSI-X
[ 7204.796602] ivpci 0000:00:06.0: irq 31 for MSI/MSI-X
[ 7204.796624] ivpci 0000:00:06.0: irq 32 for MSI/MSI-X
[ 7204.797082] ivpci 0000:00:06.0: [IVPCI] irq for msix entry: 0, vector: 29
[ 7204.797123] ivpci 0000:00:06.0: [IVPCI] irq for msix entry: 1, vector: 30
[ 7204.797162] ivpci 0000:00:06.0: [IVPCI] irq for msix entry: 2, vector: 31
[ 7204.797201] ivpci 0000:00:06.0: [IVPCI] irq for msix entry: 3, vector: 32
[ 7204.797203] ivpci 0000:00:06.0: [IVPCI] device probed: 0000:00:06.0
[ 7204.797221] [IVPCI] module loaded
```

可以看到两个`ivshmem`设备都被检测到，驱动只对`0000:00:06.0`设备进行了中断处理。

通过`lspci`查看该设备, 可以看到`MSI-X`的`Capabilities`结构显示支持`MSI-X`机制。

```
[root@host-172-16-0-29 ~]# lspci -s 00:06.00 -v
00:06.0 RAM memory: Red Hat, Inc. Inter-VM shared memory (rev 01)
	Subsystem: Red Hat, Inc. QEMU Virtual Machine
	Physical Slot: 6
	Flags: fast devsel
	Memory at 80401000 (32-bit, non-prefetchable) [size=256]
	Memory at 80400000 (32-bit, non-prefetchable) [size=4K]
	Memory at 80000000 (64-bit, prefetchable) [size=4M]
	Capabilities: [40] MSI-X: Enable+ Count=8 Masked-
	Kernel driver in use: ivpci
	Kernel modules: virtio_pci
```


查看系统的中断信息，可以看到`29-32`的中断由我们的驱动来处理:

```
[root@host-172-16-0-29 ~]# cat /proc/interrupts
...
 29:          1   PCI-MSI-edge      ivpci-0
 30:          9   PCI-MSI-edge      ivpci-1
 31:          0   PCI-MSI-edge      ivpci-2
 32:          0   PCI-MSI-edge      ivpci-3
...
```

在其中一台虚拟机上执行用户态程序, 进程被阻塞，`peer_id`为`1`:
```plain
[root@host-172-16-0-29 ~]# ./ioctl /dev/ivpci1 wait
IVPOSITION: 1
wait:
```

可以看到此时进程进入睡眠状态:

```plain
[root@host-172-16-0-29 ~]# ps aux |grep ioctl
root      3911  0.0  0.0   4212   352 pts/0    S+   17:06   0:00 ./ioctl /dev/ivpci1 wait
root      3932  0.0  0.0 112708   972 pts/1    S+   17:07   0:00 grep --color=auto ioctl
```


在另一台机器上执行用户态程序触发对端的`1`号中断:

```plain
[root@host-172-16-0-30 ~]# ./ioctl /dev/ivpci1 ring 1 1
IVPOSITION: 2
ring:
arg: 0x10001
ioctl finished
```

此时，`peer_id`为`1`的虚拟机上的用户态进程将从睡眠状态返回:

```
[root@host-172-16-0-29 ~]# ./ioctl /dev/ivpci1 wait
IVPOSITION: 1
wait:
ioctl finished, returned value: 0
```

使用中断机制，可以避免各`peer`通过轮询进行消息通知，性能更优。但`ivshmem-server`的实现以及`ivshmem`机制的服务端与客户端之间的协议设计本身的生产可用性还是有所不足。比如上边提到的第一个`peer_id`为`0`。而`peer_id`的分配机制本身所携带的信息过少，`ivshmem-server`没有足够的信息来区分哪个设备的`peer_id`各是多少。在实际生产应用中，需要独立实现`注册中心`来收集各设备`peer_id`。这样还需要`注册中心`与虚拟机内的程序、宿主机上的辅助程序都要具备网络连通性。这本身会对云平台的网络结构带来比较大的依赖。在生产环境中使用中断机制建议对`ivshmem-server`进行较大完善。

本文对`ivshmem-server`和中断机制的原理实现的介绍较为粗略，后面再专文继续详细分析`ivshmem`相关的具体实现。

参考文档:
* https://betontalpfa.medium.com/oh-no-i-need-to-write-a-pci-driver-2b389720a9d0
* https://embetronicx.com/linux-device-driver-tutorials/
* https://olegkutkov.me/2021/01/07/writing-a-pci-device-driver-for-linux/
* http://www.zarb.org/~trem/kernel/pci/pci-driver.c
* https://blog.csdn.net/haitaoliang/article/details/22753423
* https://blog.cloudflare.com/know-your-scm_rights/
* https://github.com/cmacdonell/ivshmem-code
* https://github.com/qemu/qemu/blob/master/docs/specs/ivshmem-spec.txt
* https://www.kernel.org/doc/html/latest/PCI/msi-howto.html
* https://www.kernel.org/doc/html/latest/PCI/sysfs-pci.html
