# Metis 及 SPIN 框架分析

Metis 以 SPIN model checker 为基础进行构建，其核心模块位于 `fs-state` 下。

> The main folder of the Metis model checker. This checker will run a set of file system operations (system calls) non-deterministically on multiple file systems, and then compare their behavior. If there's any discrepancy, the checker will log it.

下以 MCFS(Model Checking File System) 指代 Metis。

## How SPIN Works

Ref: [SPIN Introduction Lecture](./SpinIntro.pdf)

SPIN 框架使用 Promela (Process Meta Language) 语言描述系统。

> * PROMELA (PROcess MEta LAnguage) is a modeling language to describe concurrent (distributed) systems: n network protocols, telephone systems n multi-threaded (-process) programs that communicate via shared variables or synchronous/asynchronous message-passing.
> * SPIN (Simple Promela INterpreter) is a tool for analyzing Promela programs leading to detection of errors in the design of systems, e.g, n deadlocks, race conditions, assertion violations, n safety properties (system is never in a “bad” state) n liveness properties (system eventually arrives in a “good” state)

**SPIN tests for possible schedules.**

* system state, transition, choice point
* We can think of the possible schedules (execution traces) as forming a computation tree.

****

**SPIN's two simulation modes:**

* SPIN in random simulation mode.

  ```shell
  $ spin <pml_file>
  ```

* SPIN in guided simulation mode.

  ```shell
  $ spin -i <pml_file>
  ```

**SPIN's analyzer generation mode:**

```mermaid
graph LR
mysys.pml --SPIN--> pan.* --gcc--> pan --violation--> pml.trail
```

```shell
$ spin -a <pml_file>
$ gcc pan.c -o pan
```

****

**The main strength of SPIN is its automatic exhaustive search capabilities.**

* SPIN performs exhaustive depth-first search.

* SPIN holds a **state vector**.

  * The state vector holds the value of all variables as well as program counters (current position of execution) for each process.

* SPIN implements a **depth-first stack** of transitions.

  * Since the analyzer is not holding states in the stack, if it needs to back-track and return to a previously encountered state, it needs an “undo” operation to run the transitions in the reverse direction.

  * Since the analyzer is not holding states in the stack, when providing variable values as diagnostic information for an error path, the analyzer needs a simulation mode where choice points are decided by the transitions.

    ```shell
    $ spin -t <pml_file>
    ```

  * Users can set a bound on the depth of SPIN’s search (i.e., entries in SPIN’s depth-first stack)

    ```shell
    $ ./pan –mDEPTHBOUND

* SPIN maintains a **Seen State set** (implemented as a hash table) of states that have been seen before.
  * Due to the use of the Seen Set, checking a non-terminating system may terminate if the system only has a finite number of states.

## MCFS 如何使用 SPIN 选择测试指令

MCFS 模型在 `fs-state/mcfs-main.pml` 中定义。使用 `proctype` 关键字定义了进程 `worker` 作为 MCFS 核心的测试循环。

```promela
proctype worker()
{
    /* Non-deterministic test loop */
    int create_flag, create_mode, write_flag, offset, writelen, writebyte, filelen, chmod_mode, chown_owner, chown_group;
    do 
    :: pick_create_open_flag(create_flag);
       pick_create_open_mode(create_mode);
       atomic {
        c_code {
            /* create, check: return, errno, existence */
            ...
        };
    };
    :: pick_write_open_flag(write_flag);
       pick_write_offset(offset);
       pick_write_size(writelen);
       pick_write_byte(writebyte);
       atomic {
        c_code {
        	/* write, check: retval, errno, content */
            ...
        };
    };
    :: pick_truncate_len(filelen);
       atomic {
        c_code {
        	/* truncate, check: retval, errno, existence */
            ...
        };
    };
    :: atomic {
        c_code {
        	/* unlink, check: retval, errno, existence */
            ...
        }
    };
    :: ... /* other operations */
    od
};
```

以下形式的 `do` 控制流会循环随机选择 `instr1` ~ `instr3` 中的一条指令执行。

```promela 
do
	:: instr1;
	:: instr2;
	:: instr3;
od
```

可见，在 `worker` 中，使用 `do` 控制流实现了测试指令的随机选择。

`pick_*` 系列函数在 `parameter.pml` 中定义，以 `pick_create_open_flag` 为例：

```promela
inline pick_create_open_flag(value) {
	if
		:: value = 65;
		:: value = 193;
		:: value = 577;
		:: value = 16449;
	fi
}
```

与 `do` 类似，`if` 控制流会随机选择其中的一条指令执行。可见 `pick_*` 系列函数为测试指令随机选择了操作数。

`parameter.pml` 由 `fs-state/parameters.py` 生成。

##MCFS 如何使用 SPIN 管理文件系统状态

> The State Explorer relies on the SPIN model checker to conduct the state-space exploration. SPIN supports the Promela model-description language, and allows **embedding C code** in Promela code. This capability allows us to seamlessly issue low-level file-system syscalls and invoke utilities. **SPIN’s role is to provide optimized state-exploration algorithms (e.g., DFS) and data structures to track and store the status of the state graph**; thus, we do not have to implement these features in the State Explorer.  

SPIN 支持在 Promela 中嵌入 C 代码，包括 `c_expr`、`c_code`、`c_decl`、`c_state`、`c_track` 五个关键字（参考 [SPIN Book](./spin4_ch17.pdf)）。

* `c_code` statement is an uninterpreted state transformer, defined in an external language.
* `c_expr` statement is a user-defined boolean guard, similarly defined in an external language. 
* `c_decl` and `c_state` deal with various ways of declaring data types and data objects in C that either become part of the state vector, or that are deliberately hidden from it. 
* `c_track` primitive is used to instrument the code of the verifier to track the value of data objects holding state information that are declared elsewhere, perhaps even in separately compiled code that is linked with the SPIN-generated verifier.  

****

1. 使用 `c_track` 关键字，指定 SPIN 在内部数据结构（DFS-Stack，Seen State Table）中需要维护的状态。

   `mcfs-main.pml` 中有如下代码：

   ```
   /* DO NOT TOUCH THE COMMENT LINE BELOW */
   /* The persistent content of the file systems */
   c_track "get_fsimgs()[0]" "262144" "UnMatched";	// 目标文件系统 mount 以后自动生成
   c_track "get_fsimgs()[1]" "262144" "UnMatched";	// 目标文件系统 mount 以后自动生成
   
   /* Abstract state signatures of the file systems */
   /* DO NOT TOUCH THE COMMENT LINE ABOVE */
   c_track "get_absfs()" "sizeof(get_absfs())";
   ```

   SPIN 文档中对 `c_track` 的解释如下：

   > The primitive `c_track` is a global primitive that can declare any state object, or more generally any piece of memory, as holding state information. This primitive takes two string arguments. The first argument specifies an address, typically a pointer to a data object declared elsewhere. The second argument gives the size in bytes of that object, or more generally the number of bytes starting at the address that must be tracked as part of the system state .
   >
   > SPIN instruments the code of the verifier to copy all data pointed to via `c_track` primitives into and out of the state vector on forward and backward moves during the depth-first search that it performs.   

   可见 `c_track` 指定的内容将作为 SPIN 在状态搜索算法中维护的状态。MCFS 将 abstract state（`get_absfs` ）以及 concrete state（`get_fsimgs`）交给 SPIN 维护。其中 `"UnMatched"` 指定文件系统镜像 `fsimgs` 只会作为 concrete state 用于文件系统状态恢复，而不像 abstract state 用于状态比较。

   生成的 model checker 源代码文件 `pan.c` 中展示了状态保存的具体逻辑。

   ```c
   // SPIN 更新 abstract state 栈的函数，其逆操作为 c_revert
   long (*c_update_before)(uchar *);
   long (*c_update_after)(uchar *);
   void c_update(uchar *p_t_r)
   {
   #ifdef VERBOSE
   	printf("c_update %p\n", p_t_r);
   #endif
   	if (c_update_before)
   		c_update_before(p_t_r);
   	if( get_absfs() )
   		memcpy(p_t_r,  get_absfs() ,  sizeof(get_absfs()) );
   	else
   		memset(p_t_r, 0,  sizeof(get_absfs()) );
   	p_t_r +=  sizeof(get_absfs()) ;
   	if (c_update_after)
   		c_update_after(p_t_r);
   }
   
   long (*c_revert_before)(uchar *);
   long (*c_revert_after)(uchar *);
   void c_revert(uchar *p_t_r) { ... }
   
   // SPIN 更新 concrete state 的函数，其逆操作为 c_unstack
   long (*c_stack_before)(uchar *);
   long (*c_stack_after)(uchar *);
   void c_stack(uchar *p_t_r)
   {
   #ifdef VERBOSE
   	cpu_printf("c_stack %u\n", p_t_r);
   #endif
   	if (c_stack_before)
   		c_stack_before(p_t_r);
   	if( get_fsimgs()[1] )
   		memcpy(p_t_r,  get_fsimgs()[1] ,  262144 );
   	else
   		memset(p_t_r, 0,  262144 );
   	p_t_r +=  262144 ;
   	if( get_fsimgs()[0] )
   		memcpy(p_t_r,  get_fsimgs()[0] ,  262144 );
   	else
   		memset(p_t_r, 0,  262144 );
   	p_t_r +=  262144 ;
   	if (c_stack_after)
   		c_stack_after(p_t_r);
   }
   
   long (*c_unstack_before)(uchar *);
   long (*c_unstack_after)(uchar *);
   void c_unstack(uchar *p_t_r) { ... }
   ```

   可以看到使用 `c_track` 定义的 abstract state 在 `c_update` 和 `c_revert` 中保存和恢复，使用 `c_track` + `UnMatched` 定义的 concrete state 在 `c_stack` 和 `c_unstack` 中保存和恢复。`c_track` 的两个参数分别作为起始地址和长度用于 `memcpy` 以将状态保存在栈中。

2. 自定义 hook 函数，在状态保存和恢复时实现自定义行为。

   注意到上述 `c_update` 和 `c_stack` 等函数中存在 `c_stack_before`、`c_update_after` 等 hook 函数。因此，可以手动实现这些函数，以在状态保存和恢复时完成自定义行为。

   在 MCFS 中，这些函数在 `fs-state/fileutil.c` 中实现。

   ```c
   //  Called before the spin's checkpoint of concrete state
   static long checkpoint_before_hook(unsigned char *ptr)
   {
       submit_seq("checkpoint\n");
       makelog("[seqid = %d] checkpoint (%zu)\n", count, state_depth);
       mmap_devices(IS_CHECKPOINT);
   #ifdef CBUF_IMAGE
       for (int i = 0; i < get_n_fs(); ++i) {
           insert_circular_buf(fsimg_bufs, i, get_devsize_kb()[i], get_fsimgs()[i],
               state_depth, count, IS_CHECKPOINT);
       }
   #endif
       for (int i = 0; i < get_n_fs(); ++i) {
           if (!is_verifs(get_fslist()[i]))
               continue;
           int res = checkpoint_verifs(state_depth, get_basepaths()[i]); 
           if (res != 0) {
               logerr("Failed to checkpoint a verifiable file system %s.",
                      get_fslist()[i]);
           }
       }
       state_depth++;
       return 0;
   }
   
   // Called after the spin's checkpoint of concrete state
   static long checkpoint_after_hook(unsigned char *ptr)
   {
       unmap_devices();
       // assert(do_fsck());
       // dump_fs_images("snapshots");
       return 0;
   }
   
   //  Called before the spin's restore of concrete state
   static long restore_before_hook(unsigned char *ptr)
   {
       state_depth--;
       submit_seq("restore\n");
       makelog("[seqid = %d] restore (%zu)\n", count, state_depth);
       mmap_devices(IS_SNAPSHOT);
       for (int i = 0; i < get_n_fs(); ++i) {
           if (!is_verifs(get_fslist()[i]))
               continue;
           int res = restore_verifs(state_depth, get_basepaths()[i]);
           if (res != 0) {
               logerr("Failed to restore a verifiable file system %s.",
                       get_fslist()[i]);
           }
       }
       // assert(do_fsck());
       return 0;
   }
   
   //  Called after the spin's restore of concrete state
   static long restore_after_hook(unsigned char *ptr)
   {
       unmap_devices();
       // assert(do_fsck());
       // dump_fs_images("after-restore");
       return 0;
   }
   
   
   // Called before the spin's checkpoint of abstract state
   static long update_before_hook(unsigned char *ptr)
   {
       absfs_set_add(absfs_set, get_absfs());
       return 0;
   }
   
   //  Called after the spin's checkpoint of abstract state
   static long update_after_hook(unsigned char *ptr)
   {
       return 0;
   }
   
   
   //  Called before the spin's restore of abstract state
   static long revert_before_hook(unsigned char *ptr)
   {
       return 0;
   }
   
   //  Called after the spin's restore of abstract state
   static long revert_after_hook(unsigned char *ptr)
   {
       return 0;
   }
   ```

   以 `checkpoint_before_hook` 为例，可以看到进行了将文件系统 `mmap` 到内存等操作。 这些 hook 函数后在标记了 ` __attribute__((constructor))` 的 `init` 函数中赋值给了 `pan.c` 中定义的函数指针，从而在 `pan.c` 中可对其进行调用。

   ```c
   c_stack_before = checkpoint_before_hook;
   c_stack_after = checkpoint_after_hook;
   c_unstack_before = restore_before_hook;
   c_unstack_after = restore_after_hook;
   c_update_before = update_before_hook;
   c_update_after = update_after_hook;
   c_revert_before = revert_before_hook;
   c_revert_after = revert_after_hook;
   ```

##MCFS 每个测试步的处理流程 

在了解了 MCFS 使用 SPIN 选择测例以及管理状态的具体机制后，我们可以完整梳理 MCFS 每次测试循环的处理流程。

下以 `worker` 中 `create_file` 操作为例。其中使用了 `c_code` 嵌入了 C 代码，`atomic` 关键字表示其所括的内容为不可拆分的原子操作。

```promela
proctype worker()
{
    int create_flag, create_mode, write_flag, offset, writelen, writebyte, filelen, chmod_mode, chown_owner, chown_group;
    do 
    :: pick_create_open_flag(create_flag);
       pick_create_open_mode(create_mode);
       atomic {
        c_code {
            /* creat, check: return, errno, existence */
            makelog("BEGIN: create_file\n");
            mountall();
            if (enable_fdpool) {
                int src_idx = pick_random(0, get_fpoolsize() - 1);
                for (int i = 0; i < get_n_fs(); ++i) {
                    makecall(get_rets()[i], get_errs()[i], "%s, %d, 0%o", 
                        create_file, get_filepool()[i][src_idx], Pworker->create_flag, Pworker->create_mode);
                }
            }
            else {
                for (int i = 0; i < get_n_fs(); ++i) {
                    makecall(get_rets()[i], get_errs()[i], "%s, %d, 0%o", 
                        create_file, get_testfiles()[i], Pworker->create_flag, Pworker->create_mode);
                }
            }
            expect(compare_equality_values(get_fslist(), get_n_fs(), get_rets()));
            expect(compare_equality_values(get_fslist(), get_n_fs(), get_errs()));
            expect(compare_equality_absfs(get_fslist(), get_n_fs(), get_absfs()));
            unmount_all_strict();
            makelog("END: create_file\n");
        };
    };
    :: ...
    od
}
```

假设 MCFS 某个测试步时，`do` 控制流随机选择了 `create_file` 作为下一步的测试指令。

1. `pick_create_open_flag` 和 `pick_create_open_flag` 随机选择 `create_flag` 和 `create_mode`。
2. 使用 `mountall` 挂载所有目标文件系统。
3. 对每个目标系统执行 `create_file` 操作。
4. 比较命令返回结果是否相同。
5. 计算并比较各文件系统的 abstract state。
6. 使用 `unmount_all_strict` 解挂所有文件系统（用于同步）。
7. 检查无误后，SPIN 在 Seen State Table 中查询 abstract state。
8. 若该状态已遍历过，则回退。调用 `c_revert` 和 `c_unstack`，相应的 hook 函数会协助完成文件系统状态的恢复。
9. 若该状态未遍历过，则将新状态加入 Seen State Table。调用 `c_update` 和 `c_stack`，相应的 hook 函数会协助完成文件系统状态的保存。