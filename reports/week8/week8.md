# 第 8 周

## 本周进展

1. 阅读了 RefFS 的代码，理解了其 FUSE 的使用以及状态保存和恢复机制，分析了其与 Metis 协同工作的机制。分析结果参见 [RefFS Report](./RefFS.md)。
2. 将 lwext4 接入 Metis 测试，总结了使用 Metis 测试一般文件系统的流程，分析了将自定义文件系统接入 Metis 的基本方法。分析结果参见[ Metis 自定义测试](../Metis-custom-test.md)。
3. 尝试为 lwext4-rust 实现 Rust FUSE 接口。 

## 下周计划

1. 修改 Metis 测试的代码，将测试输出文件与代码文件分开，使测试结果更易读。
2. 继续使用 Metis 测试 lwext4，分析其中的 bug 或 false discrepancy。
3. 继续纯 Rust 版本的 ext4 文件系统的工作，并根据 Metis 测试框架的分析结果，尝试将该文件系统接入测试。 