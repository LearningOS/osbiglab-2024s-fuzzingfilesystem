# LAB 3 实验报告

## 功能总结

### 进程创建

仿照 `TaskControlBlock::fork` 实现 `TaskControlBlock::spawn` 方法。

* 由目标程序的 elf 文件构建地址空间。
* 分配 PID 和内核栈。
* 初始化子进程的 `TaskControlBlock`。
* 初始化子进程的 `TrapContext`，与新建进程方法相同。
* 将子进程插入父进程的孩子列表中。

### Stride 调度算法

* 在 `TaskControlBlock` 中添加用于 Stride 算法的 `stride` 和 `priority` 字段。
* 仿照原先 `TaskManager` 实现采用 Stride 算法的 `StrideTaskManager`。在 `StrideTaskManager` 的 `fetch` 方法中寻找 `stride` 最小的进程进行调度，然后更新该进程的 `stride`，加上 `BIG_STRIDE / priority`。

## 问答作业

stride 算法原理非常简单，但是有一个比较大的问题。例如两个 pass = 10 的进程，使用 8bit 无符号整形储存 stride， p1.stride = 255, p2.stride = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。

- 实际情况是轮到 p1 执行吗？为什么？

不是。250 + 10 发生溢出，得到的 p2.stride = 4 < p1.stride，故仍是 p2 执行。

我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明，**在不考虑溢出的情况下** ，在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 STRIDE_MAX – STRIDE_MIN <= BigStride / 2 。

- 为什么？尝试简单说明（不要求严格证明）。

  设进程  $p_1, \cdots, p_n$ 的 stride 分别为  $s_1, \cdots, s_n$ ，某时刻轮到  $p_i$ 执行，则有 $s_i \le s_1,\cdots, s_n$

  此时 STRIDE_MAX – STRIDE_MIN 为
  $$
  D =\max(s_1,\cdots, s_n) - \min(s_1, \cdots, s_n) = \max(s_1,\cdots, s_n) - s_i
  $$
  在 $p_i$ 执行一个时间片后，其 stride $s_i^\prime = s_i + \text{BigStride} / \text{Priority}_i$

  此时，STRIDE_MIN 为
  $$
  \min(s_1, \cdots, s_{i-1}, s_i^\prime, s_{i+1}, \cdots, s_n) \ge \min(s_1, \cdots, s_n) = s_i
  $$
  STRIDE_MAX 为
  $$
  \max(s_1, \cdots, s_{i-1}, s_i^\prime, s_{i+1}, \cdots, s_n) = \max(\max(s_1,\cdots, s_n), s_i^\prime)
  $$
  则 STRIDE_MAX – STRIDE_MIN 为
  $$
  D^\prime = \max(\max(s_1,\cdots, s_n), s_i^\prime) -\min(s_1, \cdots, s_{i-1}, s_i^\prime, s_{i+1}, \cdots, s_n) \le \max(\max(s_1,\cdots, s_n), s_i^\prime) -s_i
  $$
  于是有
  $$
  D^\prime \le 
  \begin{cases}
  	\max(s_1,\cdots, s_n) - s_i = D, ~ \max(s_1,\cdots, s_n) > s_i^\prime\\
  	s_i^\prime - s_i = \text{BigStride} / \text{Priority}_i \le  \text{BigStride} / 2, ~ \max(s_1,\cdots, s_n) \le s_i^\prime \\
  \end{cases}
  $$
  初始时 $D = 0$，又 $D^\prime \le D$ 或 $D^\prime \le \text{BigStride} / \text{Priority}_i$，故 $\max(D) = \text{BigStride} / 2$，当
  $$
  \min(s_1, \cdots, s_{i-1}, s_i^\prime, s_{i+1}, \cdots, s_n) = \min(s_1, \cdots, s_n) =s_i \\
  s_i^\prime \ge \max(s_1,\cdots, s_n)
  $$
  时，可取到最大值。

- 已知以上结论，**考虑溢出的情况下**，可以为 Stride 设计特别的比较器，让 BinaryHeap<Stride> 的 pop 方法能返回真正最小的 Stride。补全下列代码中的 `partial_cmp` 函数，假设两个 Stride 永远不会相等。

  ```rust
  use core::cmp::Ordering;
  
  struct Stride(u64);
  
  impl PartialOrd for Stride {
      fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
          if self.0 < other.0 {
              if other.0 - self.0 > BIG_STRIDE / 2 {
                  Some(Ordering::Greater)
              } else {
                  Some(Ordering::Less)
              }
          } else if self.0 == other.0 {
              Some(Ordering::Equal)
          } else {
              if self.0 - other.0 > BIG_STRIDE / 2 {
                  Some(Ordering::Less)
              } else {
                  Some(Ordering::Greater)
              }
          }
      }
  }
  
  impl PartialEq for Stride {
      fn eq(&self, other: &Self) -> bool {
          false
      }
  }
  ```


## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 **以下各位** 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

   > 无

2. 此外，我也参考了 **以下资料** ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

   > 无

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。