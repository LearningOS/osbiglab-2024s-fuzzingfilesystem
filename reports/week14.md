# 第 14 周

## 本周进展

1. 开发：

   * 为 ext4 实现了一套 low-level 的[接口](https://github.com/LJxTHUCS/ext4_rs/blob/main/src/ext4/low_level.rs)，主要操作对象为 `Inode`，包括：`lookup`、`create`、`read`、`write`、`link`、`unlink`、`mkdir`、`list`、`rmdir` 等。接口设计参考了 [FUSE 的 `fuse_lowlevel_ops`](https://libfuse.github.io/doxygen/structfuse__lowlevel__ops.html)。
   * 在 low-level 接口的基础上，实现了若干基于路径索引的 high-level 的[接口](https://github.com/LJxTHUCS/ext4_rs/blob/main/src/ext4/high_level.rs)。接口设计参考了 [FUSE 的 `fuse_operations`](https://libfuse.github.io/doxygen/structfuse__operations.html)。
   * 利用 low-level 接口，可以方便地[实现 FUSE](https://github.com/LJxTHUCS/ext4_rs/blob/main/ext4_fuse/src/fuse_fs.rs) 。使用 Rust FUSE 库 fuser，为 ext4 实现了 `trait fuser::Filesystem` 中的部分函数，能够挂载入 Linux 文件系统进行操作。 
2. 重构：

   * 重构了索引节点 `Inode` 中 `mode` 的处理。使用 `bitflags` 定义了 `InodeMode`。 

## 下周计划

1. 修复文件读写时 EOF 处理的问题。
2. 继续实现 FUSE，最终目标是使得 ext4 能接入 model-check。实现更多 low-level 的接口，进而利用这些接口实现 FUSE 中的更多函数。
3. 将 ext4 接入操作系统（Starry）。