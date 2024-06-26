# LAB 4 实验报告

## 功能总结

在 `easy-fs::layout::DiskInode` 中添加 `reference_count` 字段用于记录文件硬链接的数量。

在 `easy-fs::vfs::Inode` 中添加 `link` 和 `unlink` 方法。

* 对于 `link` 方法，首先在当前目录（`Inode`）下找到被链接的文件，将其 `reference_count` 加 1，然后在当前目录下增加新文件名的目录项。 
* 对于 `unlink` 方法，首先在当前目录（`Inode`）下找到相应的文件，将其 `reference_count` 减 1，然后在当前目录下删除该文件名的目录项。 若目标文件的 `reference_count` 已为 0，则将该文件删除。

## 问答作业

1. 在我们的easy-fs中，root inode起着什么作用？如果root inode中的内容损坏了，会发生什么？

   root inode 为文件系统的根节点，保存有根目录下所有的目录项。若 root inode 中的内容损坏了，会导致根目录下的其他目录和文件无法被正常索引到。

2. 举出使用 pipe 的一个实际应用的例子。

   使用 `cat` 和 `wc` 统计文件行数：

   ```sh
   cat [FILE] | wc

3. 如果需要在多个进程间互相通信，则需要为每一对进程建立一个管道，非常繁琐，请设计一个更易用的多进程通信机制。

   维护一个全局消息队列。若某时刻进程 A 需要与进程 B 通信，则 A 进程将一个发送方为 A ，接收方为 B 的消息插入队列尾部。当进程 B 需要接收进程 A 的消息时，则从队列头部开始遍历寻找发送方为 A，接收方为 B 的消息，并读取目标长度的内容。

   若某进程发送消息时，消息队列已满，则需要切换至其他进程，直到其他进程读取了消息、释放了消息队列的部分空间后，再切换回原进程执行。若某进程读取消息时，在消息队列中没有找到目标发送方发来的消息或消息长度不足，则需要切换至其他进程，直到目标发送方消息发完后再切换回原进程执行。

    

## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 **以下各位** 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

   > 无

2. 此外，我也参考了 **以下资料** ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

   > 无

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。