# 第 5 周汇报

## 本周进展

### 在 ArceOS 上运行 lwext4-rust 文件系统

@elliot10 将 c 实现的 ext4 文件系统 `lwext4` 包装成了 [rust 版本](https://github.com/elliott10/lwext4_rust)并接入 `ArceOS（StarryOS）` 运行。

1. 为了编译 `StarryOS`，在本地编译构建了 `x86_64-linux-musl` 的工具链。
2. 编译 `StarryOS` 时遇到 `rust-bindgen` 库报错，查阅文档后得知该库需要依赖 `clang`。使用 `apt` 安装了 `clang` 工具链解决了此问题 。
3. 构建文件系统时需要用到 `mkfs.ext4` 工具，使用 `apt` 安装了 `dosfstools` 以获取 `mkfs.ext4`。
4. 在本地成功编译运行了使用 `fat32` 文件系统的 `StarryOS`，但使用 `ext4` 文件系统的版本在构建时出现链接错误。向 @elliot10 询问解决方案，尝试了多种方案后未果，计划在下周找 @elliot10 当面处理。

### 阅读 [Metis](https://www.usenix.org/conference/fast24/presentation/liu-yifei  ) 论文

* **Input**: 文件系统元操作、系统调用。
* **State Explorer**: DFS，文件系统状态为节点，文件系统操作为边。
  * Abstract State saved in hash table for tree search, Concrete State saved in stack for state restoration.
  * Abstraction Function 总结主要状态：文件路径、数据、目录结构、重要的文件元数据。
  * Tracking full file system states  （in-disk & in-memory）：unmount & remount
* **RefFS**: new file system designed to function as reference file system
  * RAM-based FUSE file system  
  * Snapshot pool, Load/Store API

##下周计划

1. 找 @elliot10 解决运行 `lwext4-rust` 文件系统遇到的链接错误。
2. 按@elliot10 在[RefFS-build](https://github.com/elliott10/lwext4_rust/blob/main/doc/RefFS/RefFS-build.md) 文档所写的方法复现 Metis 测试框架。
3. 尝试使用 Metis 检查 `lwext4-rust`。