# RefFS 分析

RefFS 是专为 Metis 框架设计的用于对照测试的文件系统，为状态保存和恢复的操作进行了特殊优化。

>Ext4 lacks optimizations for model-checking state operations, limiting its suitability. We believe that a reference file system should be lightweight easily testable and extensible, robust, and optimized for SS/R operations in model checking.  
>
>We developed a new file system, called RefFS, specifically designed to function as the reference system.  

## 总体特点

RefFS 具有以下特点：

1. RefFS is a RAM-based FUSE file system.
2. Use libFUSE user-space library together with `/dev/fuse` to bridge user-space implementations and the lower-level fuse kernel module. Metis handles file system operations on RefFS in the same manner as other in-kernel file systems.
3. Provides snapshot APIs to manage the full RefFS file system state via `ioctl`s: `ioctl_SAVE`, `ioctl_RESTORE`, `ioctl_PICKLE`, and `ioctl_LOAD`.   

Snapshot APIs:

> Although RefFS is an in-memory file system lacking persistence, it possesses a concrete state (i.e., snapshot) that includes all information associated with the file system.   
>
> RefFS can capture and restore the in-memory states through its own APIs.

1. **Snapshot Pool.** A hash table that organizes all of RefFS’s snapshots; the **key** is the current position in the search tree. The value associated with each key is a snapshot structure that saves the full file system state including all data and metadata such as the superblock, inode table, file contents, directory structures, etc.  
2. **Save/Restore APIs.** The `ioctl_SAVE` API causes RefFS to take a snapshot of the full RefFS state and add an entry to the snapshot pool. The `ioctl_RESTORE` does the reverse, restoring an existing snapshot from the pool. To call `ioctl_SAVE` and `ioctl_RESTORE`，Metis must pass a **key** to RefFS.
3. **Pickle/Load APIs**. Given a hash key, the `ioctl_PICKLE` API writes the corresponding RefFS state to a disk file. It can also archive the entire snapshot pool to disk. Likewise, the `ioctl_LOAD` API retrieves a snapshot from disk, loading it back into RefFS to reinstate the file system state.   

## 使用 FUSE

RefFS 是基于 Linux FUSE 的文件系统。

RefFS 通过实现 `libfuse` 中的 `fuse_lowlevel_ops` 以支持 FUSE。

> Ref: [libfuse `fuse_lowlevel_ops`](https://libfuse.github.io/doxygen/structfuse__lowlevel__ops.html)。
>
> Rust 中与 `fuse_lowlevel_ops` 对应是 `trait fuse::Filesystem`，参见 [crate fuse](https://docs.rs/fuse/latest/fuse/trait.Filesystem.html) 。

实现了 `fuse_lowlevel_ops` 即可支持 FUSE。首先创建了一个 `fuse_lowlevel_ops` 对象。

```cpp
/**
 All the supported filesystem operations mapped to object-methods.
 */
struct fuse_lowlevel_ops FuseRamFs::FuseOps = {};
```

`fuse_lowlevel_ops` 包含各个文件系统操作的函数指针。在 `class FuseRamFs` 中定义并实现了这些函数，并在 `FuseRamFs` 的构造函数中让 `FuseOps` 中的函数指针指向这些函数。

`FuseRamFs`的所有成员均为 static 成员，其作用主要是对文件系统相关的变量和方法进行包装，以及利用访问控制关键字。

```cpp
FuseRamFs::FuseRamFs(fsblkcnt_t blocks, fsfilcnt_t inodes) {
    FuseOps.init = FuseRamFs::FuseInit;
    FuseOps.destroy = FuseRamFs::FuseDestroy;
    FuseOps.lookup = FuseRamFs::FuseLookup;
    FuseOps.forget = FuseRamFs::FuseForget;
    FuseOps.getattr = FuseRamFs::FuseGetAttr;
    FuseOps.setattr = FuseRamFs::FuseSetAttr;
    FuseOps.readlink = FuseRamFs::FuseReadLink;
    FuseOps.mknod = FuseRamFs::FuseMknod;
    FuseOps.mkdir = FuseRamFs::FuseMkdir;
    FuseOps.unlink = FuseRamFs::FuseUnlink;
    FuseOps.rmdir = FuseRamFs::FuseRmdir;
    FuseOps.symlink = FuseRamFs::FuseSymlink;
    FuseOps.rename = FuseRamFs::FuseRename;
    FuseOps.link = FuseRamFs::FuseLink;
    FuseOps.open = FuseRamFs::FuseOpen;
    FuseOps.read = FuseRamFs::FuseRead;
    FuseOps.write = FuseRamFs::FuseWrite;
    FuseOps.flush = FuseRamFs::FuseFlush;
    FuseOps.release = FuseRamFs::FuseRelease;
    FuseOps.fsync = FuseRamFs::FuseFsync;
    FuseOps.opendir = FuseRamFs::FuseOpenDir;
    FuseOps.readdir = FuseRamFs::FuseReadDir;
    FuseOps.releasedir = FuseRamFs::FuseReleaseDir;
    FuseOps.fsyncdir = FuseRamFs::FuseFsyncDir;
    FuseOps.statfs = FuseRamFs::FuseStatfs;
    FuseOps.setxattr = FuseRamFs::FuseSetXAttr;
    FuseOps.getxattr = FuseRamFs::FuseGetXAttr;
    FuseOps.listxattr = FuseRamFs::FuseListXAttr;
    FuseOps.removexattr = FuseRamFs::FuseRemoveXAttr;
    FuseOps.access = FuseRamFs::FuseAccess;
    FuseOps.create = FuseRamFs::FuseCreate;
    FuseOps.getlk = FuseRamFs::FuseGetLock;
    FuseOps.ioctl = FuseRamFs::FuseIoctl;
	...
}
```

当实现了 `fuse_lowlevel_ops` 中的函数后，即可支持 FUSE。可参考 libfuse 提供的[例子](https://libfuse.github.io/doxygen/hello__ll_8c.html)创建一个 `fuse_session`从而启动文件系统，代码如下：

```cpp
int main(int argc, const char * argv[]) {
	...
	// The core code for our filesystem.
    size_t nblocks = options.capacity / Inode::BufBlockSize;
    FuseRamFs core(nblocks, options.inodes);
    
    if (options.subtype) {
        mountpoint = options.mountpoint;
        if (mountpoint == nullptr) {
            cerr << "USAGE: fuse-cpp-ramfs MOUNTPOINT" << endl;
        } else if ((ch = fuse_mount(mountpoint, &args)) != nullptr) {
            struct fuse_session *se;
            // The FUSE options come from our core code.
            se = fuse_lowlevel_new(&args, &(FuseRamFs::FuseOps),
                                   sizeof(FuseRamFs::FuseOps), nullptr);
            if (se != nullptr) {
                fuse_daemonize(options.deamonize == 0);
                if (fuse_set_signal_handlers(se) != -1) {
                    fuse_session_add_chan(se, ch);
                    err = fuse_session_loop(se);
                    fuse_remove_signal_handlers(se);
                    fuse_session_remove_chan(ch);
                }
                fuse_session_destroy(se);
            }
            fuse_unmount(mountpoint, ch);
            delete[] mountpoint;
        }
    }
    fuse_opt_free_args(&args);
    return err ? 1 : 0;
}
```

启动过程有以下主要步骤：

1. 实例化（初始化） `FuseRamFs`，其中同时完成了对 `FuseOps` 的初始化。
2. 使用 `fuse_mount` 创建一个 mountpoint，返回一个 `fuse_chan`。`fuse_chan`是一个 communication channel，提供收发消息的 hook 函数。
3. 使用 `fuse_lowlevel_new` 创建一个 `fuse_session`。`fuse_session` 提供 FUSE 文件系统处理请求和退出时需要的 hook 函数。这里在创建`fuse_session`时传入了`FuseOps`， `FuseOps`中的函数将作为这些 hook 函数。
4. 使用 `fuse_daemonize` 设置文件系统在后台运行。
5. 使用 `fuse_set_signal_handlers` 设置信号处理函数，在收到 HUP，TERM 和 INT 信号时退出 session 。
6. 使用 `fuse_session_add_chan` 为 FUSE session 分配一个 channel 。
7. 使用 `fuse_session_loop` 进入单线程的、阻塞的事件循环。
8. 当文件系统从 `fuse_session_loop` 中退出后，使用 `fuse_session_destroy` 回收资源。
9. 使用 `fuse_unmount` 卸载文件系统。

## 状态保存和恢复 API

RefFS 并不采用磁盘或（文件、内存）模拟的磁盘保存数据，所有数据均保存在 `class FuseRamFs` 的三个 `static` 成员（也就相当于全局变量）中：

```cpp
static std::vector<Inode *> Inodes;
static std::queue<fuse_ino_t> DeletedInodes;
static struct statvfs m_stbuf;
```

这样的三元组被定义为 RefFS 的状态：

```cpp
typedef std::tuple<std::vector<Inode *>, std::queue<fuse_ino_t>, struct statvfs> verifs2_state;
```

因此 RefFS 的 Save/Restore API 即是对这个三元组的状态进行保存和恢复。RefFS 维护了一个全局的 unordered_map 用于保存之前的所有状态：

```cpp
std::unordered_map<uint64_t, verifs2_state> state_pool;	// hash_key, state
```

状态保存 `ioctl_SAVE` 由 `checkpoint` 函数处理，需要传入一个 hash_key，RefFS 会向 `state_pool` 中保存 hash_key 以及自身的状态三元组。

状态恢复 `ioctl_RESTORE` 由 `restore` 函数处理，需要传入一个 hash_key，RefFS 会从 `state_pool` 中根据 key 获取状态三元组，并覆盖自身当前状态。
