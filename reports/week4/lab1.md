# LAB 1 实验报告

## 功能总结

实现了 `sys_task_info` 系统调用，包括获取任务运行状态、任务运行时间、系统调用次数。

1. 任务运行状态：直接返回 `TaskStatus::Running`。
2. 任务运行时间：在任务第一次开始运行时获取系统时间并保存在 `TaskControlBlock` 中，当收到 `sys_task_info` 系统调用时，再次获取系统时间与任务开始时间相减，差值即为任务执行时间。
3. 对于系统调用次数：在 `TaskControlBlock` 中维护一个 `[u32; MAX_SYSCALL_NUM]`数组，每次系统调用时进行记录，在 `sys_task_info` 系统调用时将记录返回。

## 问答作业

1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。请同学们可以自行测试这些内容（运行 [三个 bad 测例 (ch2b_bad_*.rs)](https://github.com/LearningOS/rCore-Tutorial-Test-2024S/tree/master/src/bin) ）， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

   使用 RustSBI 版本 0.3.0-alpha.2。

   * 对于 `ch2b_bad_address`：Kernel 的报错信息为：

     ```
     [kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003ac, kernel killed it.
     ```

     在执行到写入 `0x0` 位置的指令时，发生 `StoreFault` 异常，被内核捕获并报错。

   * 对于 `ch2b_bad_instuctions`：Kernel 的报错信息为：

     ```
     [kernel] IllegalInstruction in application, kernel killed it.
     ```

     在 U 态执行 `sret` 时发生 `IllegalInstruction` 异常，被内核捕获并报错。

   * 对于 `ch2b_bad_register`：Kernel 的报错信息为：

     ```
     [kernel] IllegalInstruction in application, kernel killed it.
     ```

     在 U 态访问 `sstatus` 寄存器时发生 `IllegalInstruction` 异常，被内核捕获并报错。

2. 深入理解 [trap.S](https://github.com/LearningOS/rCore-Tutorial-Code-2024S/blob/ch3/os/src/trap/trap.S) 中两个函数 `__alltraps` 和 `__restore` 的作用，并回答如下问题:

   1. L40：刚进入 `__restore` 时，`a0` 代表了什么值。请指出 `__restore` 的两种使用情景。

      `a0` 为内核栈中 `TrapContext` 的地址。`__restore` 的两种使用情景分别为：

      * 当用户程序第一次由内核态进入用户态时，用于跳转到用户程序入口点执行。
      * 当用户程序由内核态 Trap 返回时，用于恢复用户态寄存器、切换到用户栈，跳转到用户程序继续执行。

   2. L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。

      ```assembly
      ld t0, 32*8(sp)
      ld t1, 33*8(sp)
      ld t2, 2*8(sp)
      csrw sstatus, t0
      csrw sepc, t1
      csrw sscratch, t2
      ```

      汇编代码特殊处理了 `sstatus`、`sscratch`、`sepc`。

      * `sstatus`：`SPP` 等字段给出了 Trap 发生之前 CPU 处在的特权级（S/U）等信息，恢复 `sstatus` 使得执行 `sret` 指令能正确进入用户态。
      * `sscratch` ：首先将在 `TrapContext` 中保存的用户栈指针写入 `sscratch`，此时 `sp` 指向内核栈。后将 `sscratch` 与 `sp` 交换，使得 `sp` 指向用户栈，`sscratch` 指向内核栈，从而在由内核态返回用户态时完成栈切换。同时，由于 `sscratch` 保存了内核栈指针，在用户态下次 Trap 进入内核态时可以利用 `sscratch` 完成由用户栈切换到内核栈。
      * `sepc`：指向 Trap 返回后下一条应该执行的指令，使得进入用户态后能跳转到正确的位置指向。

   3. L50-L56：为何跳过了 `x2` 和 `x4`？

      ```assembly
      ld x1, 1*8(sp)
      ld x3, 3*8(sp)
      .set n, 5
      .rept 27
         LOAD_GP %n
         .set n, n+1
      .endr
      ```

      `x2` 为 `sp`，在恢复其他寄存器时需要用到 `sp`，因此在最后恢复。`x4` 为 `tp`，应用程序目前不会用到。

   4. L60：该指令之后，`sp` 和 `sscratch` 中的值分别有什么意义？

      ```assembly
      csrrw sp, sscratch, sp
      ```

      该指令之后，`sp` 为用户栈指针，`sscratch` 为内核栈指针。

   5.  `__restore`：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？

      状态切换发生在 `sret` 指令。`sret` 会将权限模式设置为 `CSRs[sstatus].SPP`，`sstatus` 在 `__alltraps` 中被保存到了 `TrapContext` 中，在 `__restore` 中从 `TrapContext` 中恢复，因此在调用 `sret` 时，`SPP` 的值为 U 态，故在执行该指令后会进入用户态。

   6. L13：该指令之后，`sp` 和 `sscratch` 中的值分别有什么意义？

      ```assembly
      csrrw sp, sscratch, sp
      ```

      该指令之后，`sp` 为内核栈指针，`sscratch` 为用户栈指针。

   7. 从 U 态进入 S 态是哪一条指令发生的？

      U 态进入 S 态不一定需要执行特定指令，可能的原因有：

      * 用户态指令执行错误，发生异常。
      * 用户态调用 `ecall` 指令。
      * 用户态执行时发生时钟中断。

## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 **以下各位** 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

   > 无

2. 此外，我也参考了 **以下资料** ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

   > 无

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。
