---
layout: post
title: "flashcache简介及模块初始化"
date: 2014-12-17 17:48:16 +0800
comments: true
categories: Flashcache
---
[flashcache](https://github.com/facebook/flashcache)是Facebook开源的用SSD来缓存机械磁盘数据的Linux内核模块。

简单来说，Linux的I/O栈层次结构可以表示为:

```Raw token data
+----------------------------+
| VFS(Virtual File System)   |
+----------------------------+
| Block I/O Layer            |
+----------------------------+
| Devices                    |
+----------------------------+
```
文件系统I/O操作传递给块设备，抽象块设备最终传递给真实的设备驱动，完成I/O操作。Linux在块设备层实现了Device Mapper(DM)的映射机制。DM机制可以将一个或多个块设备再映射成一个虚拟块设备。具体的映射规则由mapping table来实现。到达虚拟块设备的I/O请求，DM机制会根据映射表找到正确的目标设备，将请求放到目标设备的请求队列中，目标设备根据目标类型进行处理。结构图如下:

```Raw token data
          +---------------+
          | Mapped Device |
          +-------+-------+
                  |
                  v
          +---------------+
          | mapping table |
          +--+---------+--+
             |         |
             |         |
             v         v
 +---------------+  +-----------------+
 |Physical Device|  | Physical Device |
 +---------------+  +-----------------+
```
DM机制的映射目标类型可以通过内核模块进行扩展，flashcache就是通过定义一种名称为"flashcache"的映射目标类型来实现SSD做为机械磁盘的缓存。结构示意图如下:

```Raw token data
+---------------+
| Mapped Device |
+-------+-------+
        |
        v
+---------------+
| mapper table  |
+-------+-------+
        |
        | flashcache
        |
    +---+---+
    |       |
    v       v
+-----+  +------+
| SSD |  | DISK |
+-----+  +------+
```
flashcache做为缓存层，提供了三种写入模式：

* writeback
* writethrough
* writearound

三种模式的区别如下图所示:
```
 writethrough  writearound  writeback
      |            |           |
 -----|------------|-----------|------------
      v            |           v       SSD
 -----|------------|------------------------
      v            v                   DISK
 -------------------------------------------
```

* writethrough：进行写操作时，会同时写入SSD和磁盘
* writearound: 进行写操作时，只对磁盘进行写操作
* writeback: 进行写操作时，只写入SSD，flashcache异步地将内容写入磁盘

下面分析flashcache模块的初始化。

flashcache内核模块的入口函数为flashcache_init。
```c
module_init(flashcache_init);
```
flashcache_init首先会对job相关的变量及结构进行初始化:
```c
    r = flashcache_jobs_init();
    if (r)
        return r;
    atomic_set(&nr_cache_jobs, 0);
    atomic_set(&nr_pending_jobs, 0);
```
flashcache_jobs_init会分配job操作所需的内存池，简化逻辑如下:
```c
static int
flashcache_jobs_init(void)
{
    _job_cache = kmem_cache_create("kcached-jobs",
                                   sizeof(struct kcached_job),
                                   __alignof__(struct kcached_job),
                                   0, NULL);
    ...

    _job_pool = mempool_create(MIN_JOBS, mempool_alloc_slab,
                               mempool_free_slab, _job_cache);

    _pending_job_cache = kmem_cache_create("pending-jobs",
                           sizeof(struct pending_job),
                           __alignof__(struct pending_job),
                           0, NULL);

    ...

    _pending_job_pool = mempool_create(MIN_JOBS, mempool_alloc_slab,
                       mempool_free_slab, _pending_job_cache);
    ...

    return 0;
}
```
然后，flashcache_init调用dm_io_client_create创建一个dm_io_client结构，它为块设备I/O请求执行过程提供内存池。
```c
flashcache_io_client = dm_io_client_create();
```
接着，flashcache_init调用dm_kcopyd_client_create创建了一个dm_kcopyd_client结构。kcopyd是一个内核进程，它用于异步地copy一个块设备的区域到一个或多个其他的块设备。dm_kcopyd_client用于向kcopyd提交任务。
```c
flashcache_kcp_client = dm_kcopyd_client_create(NULL);
```
然后，初始化了一个内核工作队列，执行do_work去处理各种job, 本文略过。
```c
INIT_WORK(&_kcached_wq, do_work);
```
接着，注册flashcache的DM机制目标类型, 目标类型中具体回调本文略过。
```c
r = dm_register_target(&flashcache_target);
```
再来看flashcache_init的最后部分:
```c
    flashcache_module_procfs_init();
    flashcache_control = (struct flashcache_control_s *)
        kmalloc(sizeof(struct flashcache_control_s), GFP_KERNEL);
    flashcache_control->synch_flags = 0;
    register_reboot_notifier(&flashcache_notifier);
    return 0;
```
首先，调用flashcache_module_procfs_init创建"/proc/flashcache"目录及文件"/proc/flashcache/flashcache_version"。
flachcache_module_procfs_init简化逻辑如下:
```c
void
flashcache_module_procfs_init(void)
{
#ifdef CONFIG_PROC_FS
    struct proc_dir_entry *entry;

    if (proc_mkdir("flashcache", NULL)) {
        entry = create_proc_entry("flashcache/flashcache_version", 0, NULL);
        if (entry)
            entry->proc_fops =  &flashcache_version_operations;
    }
#endif /* CONFIG_PROC_FS */
}
```
最后，通过register_reboot_notifier注册函数flashcache_notify_reboot。这函数会在机器重启或关闭时被调用。
```c
static struct notifier_block flashcache_notifier = {
    .notifier_call  = flashcache_notify_reboot,
    .next       = NULL,
    .priority   = INT_MAX, /* should be > ssd pri's and disk dev pri's */
};
```
至此，flashcache内核模块初始化完成，可以使用dmsetup指定flashcache目标类型创建虚拟块设备了。
