# Lab 5: Logging File System

负责助教：[唐傑伟](mailto:22302010060@m.fudan.edu.cn)


本次实验的目的是熟悉日志文件系统和块设备缓存的实现。

> _操作系统的本质：虚拟化、并发、**持久化**_

## 1. 服务器操作

```shell
# 拉取远端仓库
git fetch --all

# 提交你的更改
git add .
git commit -m "your commit message"

# 切换到新lab的分支
git checkout lab5

# 新建一个分支，用于开发
git checkout -b lab5-dev

# 引入你在上个lab的更改
git merge lab4-dev
```

如果合并发生冲突，请参考错误信息自行解决（可在VSCode中手动选择合并，本次实验中sched.c部分可能出现冲突需要合并）

## 2. Unalertable Waiting 机制的更新

*   为 `wait_sem` 等一系列函数（所有在声明时带有WARN RESULT的函数）添加了编译检查，未处理返回值将编译失败。

    ```C
    #define WARN_RESULT __attribute__((warn_unused_result))
    ```
*   添加了一套以 `unalertable_wait_sem` 为代表的函数，unalertable wait 不会被 signal（kill）打断，也没有返回值，**但请严格限制使用，保证 unalertable wait 能够在可预见的短时间内被唤醒，不会使得signal操作时效性过差。**

    > 在我们的Lab中，可能并没有明显的使用的作用区别，这里主要解释清楚，便于大家理解
* 为实现 unalertable wait，我们参考了 Linux 的 `uninterruptable wait` 的实现，为 `procstate` 添加了一项 `DEEPSLEEPING`，`DEEPSLEEPING` 与 `SLEEPING` 基本相同，区别仅在于 alert 操作对 `DEEPSLEEPING` 无效。**你需要在调度器相关代码中将 `DEEPSLEEPING` 纳入考虑。** 除 activate 操作外，一般可以将 `DEEPSLEEPING` 视为 `SLEEPING` 处理。
* 将原先的 `activate_proc` 重命名为 `_activate_proc`，添加了参数 `onalert`，以指出是正常唤醒还是被动打断，并用宏定义 `activate_proc` 为 `onalert = false` 的 `_activate_proc`，`alert_proc` 为 `onalert=true` 的`_activate_proc`。**你需要修改 `_activate_proc` 的代码，并在 `kill` 中换用 `alert_proc` 唤醒进程**

## 3. 文件系统

### 3.1. 从磁盘到块设备抽象

在计算机中，我们基本把除了 CPU 和内存以外的所有东西当作外设，与这些外设进行交互的过程称为输入/输出（I/O）。

Linux 系统中，我们将所有外设抽象成**设备**（Device），设备分为三种：

1. 字符设备（Character Device）：以字符（二进制流）为单位进行 I/O 的设备，比如键盘、鼠标、串口等；
2. 块设备（Block Device）：以块（Block）为单位进行 I/O 的设备，比如硬盘；
3. 网络设备（Network Device）：以数据包（Packet）为单位进行 I/O 的设备，比如网卡。

在 Linux 中，设备进一步被抽象成了特殊的设备文件（File），因此我们可以通过文件系统的接口来访问设备。设备文件的路径常常为`/dev/<device_name>`，比如 `/dev/sda` 就是第一个 SATA 硬盘。

块设备的本质是**实现了以下两个接口的任何对象**：

```c
// 块大小
#define BLOCK_SIZE 512
// 从设备读取数据
int block_read(int block_no, uint8_t buf[BLOCK_SIZE]);
// 将数据写入设备
int block_write(int block_no, const uint8_t buf[BLOCK_SIZE]);
```

### 3.2. 从缓存到块缓存

计算机学科中处理重复的、低速的任务的办法是什么？**缓存**。它是最简单的空间换时间办法，在不同领域还有很多其他名字：动态规划、记忆化搜索……

与 DRAM 相比，磁盘的访问时延很高。因此，理所当然地，我们需要在内存中维护一个**块缓存**（Block Cache），用于加速对块设备的访问。

块缓存是**写回**（Write Back）的，因此我们需要一个**脏位**（Dirty Bit）来标记块缓存中的数据是否已经被修改过。不过，由于我们无法追踪块缓存中的数据是否被修改过，因此我们需要让用户在使用块缓存的时候**手动**标记脏位。这个过程称为**同步**（Sync）；另一方面，由于对同一块的写是互斥的，应该有个类似锁的机制来实现「不让同一块落入两个人手里」。

因此，块缓存的接口如下：

```c
// 从设备读取一块
Block *block_acquire(int block_no);
// 标记一块为脏
void block_sync(Block *block);
// 释放一块
void block_release(Block *block);
```

从另一个角度来看，也可以把前两者看成一组：它们的作用其实和块设备的两个函数类似：一个负责读，一个负责写。最后一个函数则是释放块缓存。

### 3.3. 从内存布局到磁盘布局

内存和磁盘都是存储设备，然而由于其本身的特性，两者的布局方式有很大的不同。

我们的文件系统布局如下（以 0 为文件系统起点）：

> [!danger]
> **注意**
>
> 这里的 0 代表我们的文件系统的起始块号，不是实际的虚拟存储的起始块号。文件存储布局请参考 Lab 4。

| 起始块号           | 长度                                             | 用途       |
| -------------- | ---------------------------------------------- | -------- |
| 0              | 1                                              | 超级块      |
| `log_start`    | `num_log_blocks`                               | 日志区域     |
| `inode_start`  | `num_inodes * sizeof(InodeEntry) / BLOCK_SIZE` | inode 区域 |
| `bitmap_start` | `num_bitmap_blocks`                            | 位图区域     |
| `data_start`   | `num_data_blocks`                              | 数据区域     |

内存擅长小块随机读写，而硬盘只能大块整读整写，因此位图的效率反而比其他内存中使用过的动态分配器（例如 SLAB）要高。

硬盘和内存的分配器的接口是类似的：

```c
// 内存
void *kalloc(size_t size);
void kfree(void *ptr);

// 硬盘
int block_alloc();
void block_free(int block_no);
```

### 3.4. 日志文件系统是干什么的？

位于磁盘上的文件系统需要面临的一个问题是：当系统崩溃或者意外掉电的时候，如何维持**数据的一致性**。

比如现在为了完成某项功能，需要同时更新文件系统中的两个数据结构 A 和 B，因为磁盘每次只能响应一次读写请求，势必造成 A 和 B 其中一者的更新先被磁盘接收处理。如果正好在其中一个更新完成后发生了掉电或者 crash，那么就会造成不一致的状态。

因此，日志作为一种**写入协议**，基本思想是这样的：**在真正更新磁盘上的数据之前，先往磁盘上写入一些信息，这些信息主要是描述接下来要更新什么**。

一个典型的写入过程如下：（Checkpoint）

1. 将要写入的数据先写入磁盘的日志区域
2. 将日志区域的数据写入磁盘的数据区域
3. 将日志区域的数据删除

对于多个事务的提交，时间序列图如下：

<figure><img src="/assets/lab5-1.png" alt=""><figcaption></figcaption></figure>

一个典型的系统启动中的初始化过程如下：（`recover_from_log`）

1. 检查日志区域是否有数据
2. 如果有，说明上次系统非正常终止，将日志区域的数据写入磁盘的数据区域
3. 将日志区域的数据删除

这个过程中任意时刻发生崩溃都是安全的，顶多是丢数据，但不会导致一致性问题。

> [!important]
> **思考**
>
> 为什么不会导致一致性问题？

日志协议的思想核心是：**持久化脏块本身**。

### 3.5. 事务：文件系统上的操作抽象

在日志文件系统中，我们将**一次文件系统操作**称为一个**事务**（Transaction）。一个事务中可能包含多个写块的操作，但是这些操作要么全部成功，要么全部失败。

> [!important]
> **思考**
>
> 你能够举出“操作要么全部成功，要么全部失败”的例子吗？

先看接口：

```c
// 开始事务
void begin_op(OpContext *ctx);
// 结束事务
void end_op(OpContext *ctx);
```

这两个函数的作用请参考代码注释。我们的设计中已经有很多这样的「对称」方法，这种设计下我们往往更关注的是其中区域（Scope）的含义。

**事务的 Scope 中进行的是对块缓存的操作。** 当所有人都离开 Scope 后，事务才会真正生效（即，按照日志协议读写块缓存）。

> [!info]
> **注意**
>
> 本实验为简单起见，按照日志协议时块缓存总表现为写直达，不使用写回，以确保数据真正落盘。

### 3.6 对外接口的使用示例

读块：

```c
Block *block = bcache->acquire(block_no);

// ... Read block->data here ...

bcache->release(block);
```

写块：

```c
// Define an atomic operation context
OpContext ctx;

// Begin an atomic operation
bcache->begin_op(&ctx);
Block *block = bcache->acquire(block_no);

// ... Modify block->data here ...

// Notify the block cache that "I have modified block->data"
bcache->sync(&ctx, block);
bcache->release(block);

// End the atomic operation
bcache->end_op(&ctx);
```

## 4. 任务

> [!important]
> **任务 1**
>
> 由于为 `_wait_sem` 等一系列函数添加了编译检查，未处理返回值将编译失败；对于这一系列函数，确保起返回值被使用。
>
> 特别的，对于`_wait_sem`函数，若起返回为`false`后，不应当继续`wait`，应当立即返回（返回一个布尔值 `false` 表示当前进程未被唤醒，而是因为其他信号（例如进程被`kill`）而中断了等待状态，具体可见`sem.c`的`_wait_sem`函数实现）

> [!important]
> **任务 2**
>
> 补充完成`sched.c`中`bool _activate_proc(Proc *p, bool onalert)`函数。
>
> 对于**DEEPSLEEPING**状态：
>
> * `on alert = true`时表示主动请求唤醒，`DEEPSLEEPING`状态下主动请求唤醒将被忽略，返回false；
> * `on alert = false`时表示被动唤醒，这时 `DEEPSLEEPING` 状态下的进程会被唤醒，需要将其加入调度队列并设置为`RUNNABLE`，返回`true`。
>
> 其余状态不需要更改。

> [!important]
> **思考**
>
> 我们为什么需要`DEEPSLEEPING`状态？

> [!important]
> **任务 3**
>
> 在`kill`函数中，使用`alert_proc`来唤醒进程
>
> * `kill` 使用 `onalert=true` 表示它尝试以一种“安全”方式来唤醒进程。这意味着如果进程在 `DEEPSLEEPING` 中，系统默认不打断它。
> * `kill` 并非总是意味着立即终止，我们此处希望进程在自然状态下处理完成，而不是强制唤醒一个不稳定的进程。
> * 若在之前使用过`_wait_sem`函数（即非直接调用`wait_sem`，而是出现过以`_lock_sem` 与 `_wait_sem`调用函数时），需要指定`alertable`参数

> [!important]
> **任务 4**
>
> 阅读`cache.h`中的数据结构，理解其含义，并完成`cache.c`中的下列函数，最后在`src/fs/test`下通过相关`mock`测试。

```c
Part 1:

/*
 * 返回当前cache中块的个数（可以使用一个全局变量，但需要注意加减的原子性能否保证）。
 */
static usize get_num_cached_blocks()

/* 
 * 我们拥有一个链表记录了所有在cache的块，其软上限是EVICTION_THRESHOLD
 * 首先判断acquire的块在不在cache中，如果在应该如何操作？如果不在如何操作？
 * 如果当我们再插入一个块进入cache就要超过上界时，我们需要uncache一块？（如何选择这
 * 删除的一块？这一块还需要满足哪些条件？）使用device_read从设备读块的内容acquire
 * 返回时，需要保证调用者已获得了该block的锁。
 */
static Block *cache_acquire(usize block_no)

/* 
 * 如何表示这一块不再被acquire，处于可用状态？返回后，需要保证调用者释放了该block锁。
 * 提示：若希望一个信号量post时可以通知所有等待它的信号量，可以使用post_all_sem函数。
 */
static void cache_release(Block *block)

Part 2:

/*
 * 当拥有特别多的操作时，我们需要等待（原因在于一个事务提交的log数量有上限），如何判断
 * 当前是否依然支持更多操作，判断条件是什么？（考虑log的数量限制）如果在进行checkpoint
 * 的过程中，我们是否可以继续begin_op? 上述情况我们都需要等待，如何实现？初始化ctx->rm
 * 为OP_MAX_NUM_BLOCKS，表示其剩余的可用操作数。
 */
static void cache_begin_op(OpContext *ctx)

/*
 * 若ctx为NULL，直接使用device_writex写入块
 * （在调用函数前，调用者必须保证已经通过acquire持有该块的锁）
 * 标记该块为脏块，在log_header中记录该块（如果该块已经被标记为脏块呢？）
 */
static void cache_sync(OpContext *ctx, Block *block)

/*
 * 什么时候进行checkpoint写入操作？（checkpoint详细过程见3.4）
 * 注意：返回该函数时必须保证该事务内的所有写入全部完成。
 */
static void cache_end_op(OpContext *ctx)

/* 
 * 初始化你一切想要初始化的（如锁，信号量，一些赋值变量...）同时根据log完成断电恢复操作。
 */
void init_bcache(const SuperBlock *_sblock, const BlockDevice *_device)  
    

/* 
 * 提示：此部分建议善用bitmap_get,bitmap_set,bitmap_clear函数，见bitmap.h。
 */
Part 3:

/* 
 * 根据位图分配一个可用块，返回其块号，同时请注意：返回的块需要保证数据已经被memset，为干净块。
 */
static usize cache_alloc(OpContext *ctx)

/*
 * 根据位图恢复一个可用块，此时无需完全清除块的内容。
 */
static void cache_free(OpContext *ctx, usize block_no)
```

> [!important]
> **思考**
>
> 若存在一个事务在begin\_op阶段需要等待checkpoint完成的信号才能继续，但若其还未运行至`wait_sem`时，checkpoint完成的信号已经发送，这样的并发问题是否存在？如何解决？

## 5. 测试方式

有趣的是，文件系统本身作为一个抽象实现，理论上只要提供了正确的接口（例如块设备、内存分配器、锁等），本身应当是具有良好跨平台性的。因此，本次实验我们使用了基于 Mock 的评测方法，离开 QEMU 环境来测试你的文件系统。相关 C/C++ 代码在 `src/fs/test` 目录下。

我们仅 mock 了以下方法，因此请保证你只调用了在前面实验中出现的这些方法（理论上你也不该用到其他方法，否则说明你的实现不具有跨平台性）：

```c
void *kalloc(isize x);
void kfree(void *ptr);
void printk(const char *fmt, ...);
void yield();

// list.c 宏/方法集
// SpinLock 方法集
// Semaphore 方法集
// PANIC、assert 宏
// RefCount 方法集
```

首次评测时请在 `src/fs/test` 下执行：

```sh
$ mkdir build
$ cd build
$ cmake ..
```

之后每次评测时请在 `src/fs/test/build` 下执行：

```sh
$ make && ./cache_test
```

如果报错，请尝试清理：

```sh
$ make clean
```

通过标准：最后一行出现 `(info) OK: 23 tests passed.`。此外助教会通过测试的输出判断测试是否在正常运行。

助教的测试结果样例输出：（仅供参考，无需完全一致）

```shell
(info) "init" passed.
(info) "read_write" passed.
(info) "loop_read" passed.
(info) "reuse" passed.
(debug) #cached = 20, #read = 154
(info) "lru" passed.
(info) "atomic_op" passed.
(fatal) 
(info) "overflow" passed.
(info) "resident" passed.
(info) "local_absorption" passed.
(info) "global_absorption" passed.
(info) "replay" passed.
(fatal) 
(info) "alloc" passed.
(info) "alloc_free" passed.
(info) "concurrent_acquire" passed.
(info) "concurrent_sync" passed.
(info) "concurrent_alloc" passed.
(info) "simple_crash" passed.
(trace) running: 1000/1000 (296 replayed)
(info) "single" passed.
(trace) running: 1000/1000 (334 replayed)
(info) "parallel_1" passed.
(trace) running: 1000/1000 (291 replayed)
(info) "parallel_2" passed.
(trace) running: 500/500 (115 replayed)
(info) "parallel_3" passed.
(trace) running: 500/500 (125 replayed)
(info) "parallel_4" passed.
(trace) throughput = 71764.50 txn/s
(trace) throughput = 67016.50 txn/s
(trace) throughput = 66486.50 txn/s
(trace) throughput = 62407.00 txn/s
(trace) throughput = 60668.50 txn/s
(trace) throughput = 75762.00 txn/s
(trace) throughput = 64034.00 txn/s
(trace) throughput = 75834.50 txn/s
(trace) throughput = 58649.00 txn/s
(trace) throughput = 67872.00 txn/s
(trace) throughput = 51930.50 txn/s
(trace) throughput = 54758.00 txn/s
(trace) throughput = 62521.00 txn/s
(trace) throughput = 72498.50 txn/s
(trace) throughput = 66931.00 txn/s
(trace) throughput = 59653.50 txn/s
(trace) throughput = 63922.50 txn/s
(trace) throughput = 57582.00 txn/s
(trace) throughput = 71582.50 txn/s
(trace) throughput = 64026.50 txn/s
(trace) throughput = 56911.50 txn/s
(trace) throughput = 62170.50 txn/s
(trace) throughput = 51186.50 txn/s
(trace) throughput = 61406.50 txn/s
(trace) throughput = 66081.00 txn/s
(trace) throughput = 68124.50 txn/s
(trace) throughput = 64132.50 txn/s
(trace) throughput = 69123.50 txn/s
(trace) throughput = 62546.00 txn/s
(trace) throughput = 73620.50 txn/s
(trace) running: 30/30 (3 replayed)
(info) "banker" passed.
(info) OK: 23 tests passed.
```

> [!danger]
>
> **注意**
>
> 上述测试只用于检测`cache.c`中内容是否正确运行， 任务 1-3 的内容是否正确完成至少需要确保在 `build/`运行`make qemu`时能正确跑通（可以打开`core.c`中`proc_test()`和`user_proc_test`尝试运行）

## 6. 评分标准

本次实验的评分标准如下：

- **核心实现**：70%
- **前序 Lab2、Lab3 功能正确**：10%
- **思考题**：40%


| Test              | Score |
| ----------------- | ----- |
| init              | 2     |
| read_write        | 4     |
| loop_read         | 2     |
| reuse             | 2     |
| lru               | 2     |
| atomic_op         | 2     |
| overflow          | 3     |
| resident          | 3     |
| local_absorption  | 3     |
| global_absorption | 3     |
| replay            | 3     |
| alloc             | 3     |
| alloc_free        | 3     |
| concurrent_acquire | 5     |
| concurrent_sync    | 5     |
| concurrent_alloc   | 5     |
| simple_crash | 3     |
| single       | 4     |
| parallel_1   | 4     |
| parallel_2   | 4     |
| parallel_3   | 2     |
| parallel_4   | 2     |
| banker       | 1     |



## 7. 提交

**提交方式**：将实验报告提交到 elearning 上，格式为 `学号-lab5.pdf` 。

**注意**：从 `lab1` 开始，用于评分的代码以实验报告提交时为准。如果需要使用新的代码版本，请重新提交实验报告。

**截止时间**：<mark style="color:red;">**12月7日23:59**</mark>。




> [!danger]
>
> **逾期提交将扣除部分分数**
>
> 计算方式为 $\text{score}_{\text{final}} = \text{score} \cdot \left(1 - n \cdot 20\% \right)$，其中 $n$ 为迟交天数，不满一天按一天计算）。

报告中可以包括下面内容

* 代码运行效果展示
* <mark style="color:red;">**讨论你的程序在事务执行的各个阶段（事务开始、块、写回）崩溃时，以及并发时下如何保证一致性。**</mark>
* <mark style="color:red;">**讨论为了实现文件系统，你对调度做了哪些改动。**</mark>
* 实现思路和创新点
* 对后续实验的建议
*   其他任何你想写的内容

    > ~~你甚至可以再放一只可爱猫猫~~

报告中不应有大段代码的复制。如有使用本地环境进行实验的同学，请在elearning上提交代码，提交前执行`make clean`。使用服务器进行实验的同学，助教会在服务器上检查，不需要另外提交代码。

在服务器上操作的同学，此次实验完成后请提交（或者说创建一个新分支）到 `lab5-submission` 分支，助教会使用你在此分支上提交记录来批作业。如果此分支最后提交时间晚于实验报告提交时间，助教会选择此分支上在实验报告提交时间前的最后一个提交作为批改代码。

**提交操作**：

```shell
# 提交最后的代码
git add .
git commit -m "your final commit message"

# 新建一个分支，用于提交
git checkout -b lab5-submission
```

## 8. 参考资料

1. OS2020助教对Lab中涉及的xv6文件系统的介绍：https://github.com/FDUCSLG/OS-2022Fall-Fudan/blob/lab10/doc/filesystem-v4.pdf （可阅读其中关于本次实验(P. 26-P. 33)的部分，其中还包含了下次实验(inode等)的内容）
2. 聊聊 xv6 中的文件系统：https://www.cnblogs.com/KatyuMarisaBlog/p/14366115.html
3. xv6 中文文档：https://th0ar.gitbooks.io/xv6-chinese/content/content/chapter6.html
