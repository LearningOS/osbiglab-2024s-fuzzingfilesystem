# Metis 测试任意文件系统

## 使用 Metis 测试 lwext4

lwext4 是一个 C 语言实现的 ext4 文件系统项目。@elliot 的 lwext4-rust 是在 C 实现的 lwext4 基础上，采用 bindgen 工具封装得到的，并没有引入额外的逻辑。因此，可以直接将原始 C 语言版本的 lwext4 接入 Metis 测试。

在之前对 Metis 的分析中存在一个误区：文件系统必须实现 fuse 才能接入 Metis 测试。这个要求并不是必须的。实际上，只要对 Metis 的初始化过程进行修改，加入自定文件系统的准备过程即可接入测试。

`fs-state/README.md` 中 Testing other file systems 节描述了测试任意文件系统的方法，主要通过修改 `setup.sh` 脚本。经实验发现该方法已**弃用**，只需要向 `setup.sh` 中传入目标文件系统（及文件系统的大小）作为参数即可测试。

Metis 不支持 lwext4，需要对配置和初始化部分进行修改以支持 lwext4。

1. 在 `setup.c` 中，`setup_filesystems` 函数初始化所有待测的文件系统，在这里加入对 lwext4 的特判。

   ```c
   if (strcmp(get_fslist()[i], "lwext4") == 0)
   {
       ret = setup_lwext4(get_devlist()[i], get_basepaths()[i], get_devsize_kb()[i]);
   }
   ```

   在 `setup_lwext4` 中，首先分配磁盘设备，再构建 lwext4 文件系统。这里使用了 `lwext4-mkfs` 工具，其源码在 lwext4 仓库中，可自行编译得到。

   ```c
   static int setup_lwext4(const char *devname, const char *basepath, const size_t size_kb)
   {
       int ret;
       char cmdbuf[PATH_MAX];
       // Expected >= 256 KiB
       ret = check_device(devname, 256);
       if (ret != 0)
       {
           fprintf(stderr, "Cannot %s because %s is bad or not ready.\n",
                   __FUNCTION__, devname);
           return ret;
       }
       // fill the device with zeros
       snprintf(cmdbuf, PATH_MAX,
                "dd if=/dev/zero of=%s bs=%zu count=1",
                devname, size_kb * 1024);
       execute_cmd(cmdbuf);
       // format the device with the specified file system
       snprintf(cmdbuf, PATH_MAX, "lwext4-mkfs --verbose -i %s -e 4", devname);
       execute_cmd(cmdbuf);
   
       return 0;
   }
   ```

2. 在 `init_globals.h` 中，注册 lwext4 以及其使用的设备类型，这里选用 ramdisk。

    ```c
    static const char *fs_all[] = {"btrfs", "ext2", "ext4", "f2fs", 
                                   "jffs2", "ramfs", "tmpfs", "verifs1", 
                                   "verifs2", "xfs", "nilfs2", "jfs",
                                   "nova", "lwext4"};
                                   
    static const char *dev_all[]= {"ram", "ram", "ram", "ram", 
                                    "mtdblock", "", "", "", 
                                    "", "ram", "ram", "ram",
                                    "pmem", "ram"};
    ```

3.  `mount.c` 的 `mountall` 函数用于所有测试文件系统的挂载，在其中加入 lwext4 的特判。

   ```c
   if (is_lwext4(get_fslist()[i])) {
       char cmdbuf[PATH_MAX];
       snprintf(cmdbuf, PATH_MAX, "mount %s %s", get_devlist()[i], get_basepaths()[i]);
       ret = execute_cmd_status(cmdbuf);
       // Chmod of root (default mod of lwext4 root is 777)
       snprintf(cmdbuf, PATH_MAX, "chmod 755 %s", get_basepaths()[i]);
       ret = execute_cmd_status(cmdbuf);
   }
   ```

   这里对于 lwext4 额外执行了一条 `chmod 755`。由于 lwext4 的根目录的 mode 初始为 777，而其他用于对照的文件系统（如 RefFS）为 755，若不加修改会导致对照测试刚开始就出现不一致。因此在此手动将其设置为 755 使得初始状态一致。

现在 Metis 即可支持测试 lwext4，在 `mcfs_scripts` 目录下添加将 lwext4 和 RefFS 对照测试的脚本 `lwext4_verifs2.sh`。

```sh
LWEXT4_SZKB=256
VERIFS2_SZKB=0 

cd ..
sudo ./stop.sh

sudo ./setup.sh -f lwext4:$LWEXT4_SZKB:verifs2:$VERIFS2_SZKB
```

运行该脚本，可以看到 Metis 开始对这两个文件系统进行对照测试。似乎找到了一个不一致的状态，输出如下：

```
[   1.952787890] fileutil.c:274:compare_equality_absfs: [seqid=560, fs=lwext4]: Directory structure:
/, mode=<dir 755>, size=2048 (Ignored), nlink=4, uid=0, gid=0
/d-00, mode=<dir 755>, size=1024 (Ignored), nlink=3, uid=0, gid=0
/d-00/d-00, mode=<dir 755>, size=1024 (Ignored), nlink=2, uid=0, gid=0
/d-00/f-00, mode=<file 644>, size=0, nlink=1, uid=0, gid=0
/d-00/f-01, mode=<file 600>, size=0, nlink=1, uid=0, gid=0
/d-00/f-02, mode=<file 644>, size=1, nlink=1, uid=0, gid=0
/d-01, mode=<dir 755>, size=1024 (Ignored), nlink=2, uid=0, gid=0
/d-01/f-00, mode=<file 640>, size=0, nlink=1, uid=0, gid=0
/d-01/f-01, mode=<file 644>, size=0, nlink=1, uid=0, gid=0
/d-01/f-02, mode=<file 600>, size=1, nlink=1, uid=0, gid=0
/f-00, mode=<file 600>, size=1, nlink=1, uid=0, gid=0
/f-01, mode=<file 755>, size=0, nlink=1, uid=0, gid=0
/f-02, mode=<file 644>, size=0, nlink=1, uid=0, gid=0
hash=55908c75a3f8c165117ef87bc4ceed9f
[   1.952844762] fileutil.c:274:compare_equality_absfs: [seqid=560, fs=verifs2]: Directory structure:
/, mode=<dir 755>, size=551 (Ignored), nlink=4, uid=0, gid=0
/d-00, mode=<dir 755>, size=475 (Ignored), nlink=3, uid=0, gid=0
/d-00/d-00, mode=<dir 755>, size=171 (Ignored), nlink=2, uid=0, gid=0
/d-00/f-00, mode=<file 644>, size=0, nlink=1, uid=0, gid=0
/d-00/f-01, mode=<file 600>, size=0, nlink=1, uid=0, gid=0
/d-00/f-02, mode=<file 644>, size=0, nlink=1, uid=0, gid=0
/d-01, mode=<dir 755>, size=399 (Ignored), nlink=2, uid=0, gid=0
/d-01/f-00, mode=<file 640>, size=0, nlink=1, uid=0, gid=0
/d-01/f-01, mode=<file 644>, size=0, nlink=1, uid=0, gid=0
/d-01/f-02, mode=<file 600>, size=1, nlink=1, uid=0, gid=0
/f-00, mode=<file 600>, size=1, nlink=1, uid=0, gid=0
/f-01, mode=<file 755>, size=0, nlink=1, uid=0, gid=0
/f-02, mode=<file 644>, size=0, nlink=1, uid=0, gid=0
hash=d9170899f737bfb11be52006fddbb3bc
```

`/d-00/f-02, mode=<file 644>, size=1, nlink=1, uid=0, gid=0` 和 `/d-00/f-02, mode=<file 644>, size=0, nlink=1, uid=0, gid=0` 二者 size 不同，未知是 bug 还是 false discrepancy，仍需要进一步研究。

若想仿照 lwext4 接入 Metis 测试，可能需要文件系统满足：

1. 能够使用自定义的构建工具（如mkfs）在空的磁盘设备上（如 /dev/ram）构建文件系统。
2. 能够支持 mount 和 unmount 操作。

## 使用 FUSE 接入 Metis

除上述接入 Metis 的方法外，还可以参照 RefFS，采用更简单的 FUSE 方法接入 Metis。RefFS 与 Metis 简单协同，得益于以下特点：

1. RefFS 是 all-in-memory 的文件系统，其所有状态均在内存中，因此不需要使用 mount 和 unmount 即可跟踪其全部状态。

2. RefFS 是基于 FUSE 的文件系统，不需要 mkfs 工具进行构建，可以直接作为用户进程运行。 

相对特殊的是，RefFS 特别实现了 ioctl_SAVE 和 ioctl_RESTORE 两个 API 以自行维护历史状态，因此不需要 SPIN 跟踪状态。不过，我们的 FUSE 文件系统也可以不实现状态保存和恢复的特殊 API，仍利用 SPIN 来维护状态。可以将某块内存空间作为于文件系统的模拟磁盘，同时作为 concrete state 交给 SPIN 进行跟踪，这样仍可利用 SPIN 来实现文件系统全部状态的保存和恢复。

因此，对于一个自定义文件系统，一个相对简单易行 Metis 测试方案如下：

1. 对文件系统的底层接口进行包装，实现 FUSE 的（部分）接口。
2. 若文件系统自行保存和恢复状态，可以仿照 RefFS 设计特殊的文件系统调用（ioctl），在文件系统收到相应调用时自行进行状态保存或恢复。若采用此方法，需要修改 Metis 中状态切换前后的 hook 函数（[参考](../week7/Metis-Spin.md)）以进行相应的文件系统调用。
3. 若利用 SPIN 进行文件系统状态的维护，则可以考虑用一块内存空间来模拟文件系统的磁盘。若采用一块内存空间模拟磁盘，可能需要双线程的交互（FUSE 和 Metis 测试系统），需要使用锁来保证对文件系统模拟磁盘（同时也是 SPIN 跟踪的状态）的安全读写。 

## Rust FUSE 尝试

根据上一节的思路，我尝试了为 lwext4-rust 实现 FUSE 接口。一开始使用的 Rust FUSE 库为 [fuse](https://docs.rs/fuse/latest/fuse/)，后发现该库疏于维护，换为 [fuser](https://docs.rs/fuser/latest/fuser/)。

为支持 FUSE，需要实现 fuser 库中的 `trait Filesystem` 。[C libfuse 库](https://libfuse.github.io/doxygen/)中提供了两套 API，分别为上层的 `fuse.h` （主要以路径为操作对象）和底层的 `fuse_lowlevel.h`（主要以 inode 为操作对象）；而 Rust  fuser 库中只有 `trait Filesystem` 对应于 `fuse_lowlevel.h` 中的 `struct fuse_lowlevel_ops`，而没有上层接口。

因此，我在为 lwext4-rust 实现 FUSE 时，只能用到 `trait Filesystem` 这套底层接口，因此需要触及大量的针对 `inode` 的操作，从而必须深入 lwext4 的实现细节，在实现了 `init`、`getattr` 和 `lookup` 后放弃。

虽然 `trait Filesystem` 并未实现完成，但 FUSE 进程可以正常启动，并且已实现的 `getattr` 接口可以正常运行。如对于 FUSE 挂载的根节点执行 stat，可以得到以下输出：

```
  File: mnt
  Size: 8192            Blocks: 16         IO Block: 4096   directory
Device: 47h/71d Inode: 1           Links: 3
Access: (0777/drwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 1970-01-01 08:00:00.000000000 +0800
Modify: 1970-01-01 08:00:00.000000000 +0800
Change: 1970-01-01 08:00:00.000000000 +0800
 Birth: -
```

同时在 FUSE 进程的 log 输出中可以看到其调用了 `getattr`。

```
2024-04-20T17:35:11.961Z DEBUG [fuser::request] FUSE(  4) ino 0x0000000000000001 GETATTR
```

虽然为 lwext4-rust 实现 FUSE 不顺利，但若对于熟悉实现的自定义文件系统，实现 FUSE，并进而接入 Metis 进行测试应该是可行的。

