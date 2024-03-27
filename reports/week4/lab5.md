# LAB 5 实验报告

## 功能总结

在 `os::sync` 中新增 `detect` 模块，定义 `struct DeadLockDetect`，维护银行家算法的 `available`、`allocation` 和 `need` 三张表，并实现以下方法：

* `add_resouce`：增加一种资源类型。
* `add_thread`：增加一个需要资源的线程。
* `request`：线程声明对某项资源的需求，在其中调用 `check_safety` 进行安全性检查。
* `allocate`：为某线程分配资源。
* `recycle` ：从某线程回收资源。
* `check_safety`：使用银行家算法检查线程的需求是否能被满足。

以  semaphore 的死锁检测为例：

* 当新建一个 semaphore 时，调用 `add_resource` 增加资源。
* 当调用 `semaphore_down` 时，调用 `request`，此时会使用银行家算法进行检查。若状态安全，则调用 `allocate` 分配资源，否则返回 `-0xdead` 错误。
* 当调用 `semaphore_up` 时，调用 `recycle` 回收资源。

mutex 的死锁检测与之类似。

## 问答作业

1. 在我们的多线程实现中，当主线程 (即 0 号线程) 退出时，视为整个进程退出， 此时需要结束该进程管理的所有线程并回收其资源。 

   * 需要回收的资源有哪些？ 

     * 属于各线程的资源：线程号、用户栈、Trap 上下文。

     * 属于进程的资源：子进程、地址空间、文件描述符。

   * 其他线程的 `TaskControlBlock` 可能在哪些位置被引用，分别是否需要回收，为什么？

     * 全局任务管理器 `TASK_MANAGER` 中的等待队列 `ready_queue`。保存了 `TaskControlBlock` 的强引用，需要将这些线程从 `ready_queue` 中删除，从而正确减少这些线程的引用计数。

     * 所属进程 `ProcessControlBlock` 的线程列表 `tasks`。保存了 `TaskControlBlock` 的强引用，进程退出时需清空 `tasks` 列表。
     * 所属进程 `ProcessControlBlock` 的互斥锁 `MutexBlocking`、信号量 `Semaphore` 和 条件变量 `CondVar` 的等待列表 `wait_queue`。可能保存了 `TaskControlBlock` 的强引用。进程退出时会清空所有的互斥资源，其中保存的 `TaskControlBlock` 的强引用同时会被释放。

2. 对比以下两种 `Mutex.unlock` 的实现，二者有什么区别？这些区别可能会导致什么问题？

   ```rust
   impl Mutex for Mutex1 {
       fn unlock(&self) {
           let mut mutex_inner = self.inner.exclusive_access();
           assert!(mutex_inner.locked);
           mutex_inner.locked = false;
           if let Some(waking_task) = mutex_inner.wait_queue.pop_front() {
               add_task(waking_task);
           }
       }
   }
   
   impl Mutex for Mutex2 {
       fn unlock(&self) {
           let mut mutex_inner = self.inner.exclusive_access();
           assert!(mutex_inner.locked);
           if let Some(waking_task) = mutex_inner.wait_queue.pop_front() {
               add_task(waking_task);
           } else {
               mutex_inner.locked = false;
           }
       }
   }
   ```

   第一种无论等待队列中是否有线程，都会将 `mutex_inner.locked` 置为 `false`；第二种仅在等待队列中没有线程时才将 `mutex_inner.locked` 置为 `false`。

   对于第一种实现，若等待队列中有线程，则该线程会进入临界区执行，且此时 `mutex_inner.locked` 为 `false`。若该线程在临界区的执行过程中发生时钟中断，操作系统切换到其他线程执行时，由于 `mutex_inner.locked` 为 `false` ，其他线程使用 `Mutex::lock` 获取锁时会成功，从而也能进入临界区执行。这就导致临界区中存在两个线程同时执行，破坏了互斥条件。

## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 **以下各位** 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

   > 无

2. 此外，我也参考了 **以下资料** ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

   > 1. [操作系统之银行家算法_银行家算法求安全序列例题-CSDN博客](https://blog.csdn.net/weixin_44001222/article/details/112762387)
   >
   > 2. [zh.wikipedia.org/wiki/银行家算法](https://zh.wikipedia.org/wiki/银行家算法)

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。
