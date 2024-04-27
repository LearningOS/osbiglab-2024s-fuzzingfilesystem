# 第 9 周

## 本周进展

1. 修改 Metis 测试项目的 shell 脚本，使源代码、二进制文件、测试输出分开，提高结果的可读性。

2. 根据第 8 周的分析，修改 Metis 的代码使得其支持自定义的 FUSE 文件系统，实现了将 lwext4-fuse 接入测试。

   以上两项修改参见[Metis (Forked)](https://github.com/LJxTHUCS/Metis/tree/test-lwext4)。

3. 根据第 8 周的分析，尝试将[纯 Rust 版本的 ext4 文件系统（ext4_rs）](https://github.com/yuoo655/ext4_rs)接入 Metis 测试，为此需要实现 FUSE。首先对 ext4_rs 进行了重构，对其中的各模块进行了拆分，包装为一个 Rust 库。同时也删除了其中的无用代码，并修复了所有 Warning（[仓库链接](https://github.com/LJxTHUCS/ext4_rs)）。

4. 使用 ext4_rs 实现[基于 FUSE 的 ext4 文件系统](https://github.com/LJxTHUCS/ext4_rs_fuse/tr)，尚未完成。

## 下周计划

1. 继续为 ext4_rs 实现 FUSE，进而将其接入 Metis。
2. 继续重构 ext4_rs 的代码，为对接 FUSE，考虑提供不同层级的接口，提高适配性。
3. （可能）为 ext4_rs 实现新功能。