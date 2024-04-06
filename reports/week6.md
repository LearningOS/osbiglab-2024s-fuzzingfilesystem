# 第 6 周

## 本周进展

1. 与 @elliot10 解决了链接问题，可以在 ArceOS 上运行 rust 版本的 ext4。
2. 运行了 Metis 和 RefFS，可以对提供的文件系统进行测试。
3. 阅读了 Metis 的部分项目代码，学习其使用的 Spin 模型验证工具的教程。

## 下周计划

1. 继续学习 Spin 模型验证工具，理解 Metis 项目中使用 Spin 的细节。
2. 进一步实验 Metis，核心理解其使用 SPIN 进行状态搜索和状态维护的部分。
3. 与工程师商议使用 Metis 测试目前的 ext4 实现。



##SPIN Model Checker

[Spin: A widely used open-source software verification tool](https://spinroot.com/spin/whatispin.html)

Spin 使用 Promela (Process or Protocol Meta Language) 建模语言对系统进行建模。

1. `parameters.pml` 用于给出输入参数，由 `parameters.py` 生成，满足一定的分布概率。SPIN 自动选择测试命令进行测试。
2. `mcfs-main.pml` 给出了 Metis 的主要测试循环，SPIN 不断自动选择测例对目标文件系统进行测试。
3. 状态扩展的 DFS 算法以及状态的保存由 SPIN 完成（具体机制尚未理解）。