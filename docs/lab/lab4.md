# Lab 4: VirtIO Driver

负责助教：[唐傑伟](mailto:22302010060@m.fudan.edu.cn)

到目前为止，我们已经实现了内核的一些重要组件：内存分配、内核/用户进程、页表等。从这个 Lab 开始，我们将关注操作系统中的持久化问题，最终实现一个功能较为完善的文件系统。

## 1. 服务器操作

运行以下命令进行代码的拉取与合并

```shell
# 拉取远端仓库
git fetch --all

# 提交你的更改
git add .
git commit -m "your commit message"

# 切换到新lab的分支
git checkout lab4

# 新建一个分支，用于开发
git checkout -b lab4-dev

# 引入你在上个lab的更改
git merge lab3-dev
```

如果合并发生冲突，请参考错误信息自行解决。

## 2. I/O 框架

硬盘、SD卡等一类设备都是块设备。块设备的特点是数据的读写以块（block）为单位，一次性读取或写入固定大小的数据块。为了实现对块设备的高效管理和操作，操作系统通常会提供一个 I/O 框架，用于抽象和管理这些块设备的访问。

在现代操作系统中，I/O 框架主要分为同步 I/O 和异步 I/O 两种模式。同步 I/O 会阻塞调用的进程，直到 I/O 操作完成。而异步 I/O 则允许进程发起 I/O 操作后立即返回，不会阻塞进程的执行流，从而提高系统的并发度。在本实验中，我们将重点关注同步 I/O 的实现。

I/O 的基本逻辑如下：

* **读**：块设备驱动向设备发送读请求和目标地址，然后等待请求完成。此时发起请求的进程可以休眠，调度器可以调度其他进程（异步）。请求完成后，内核通知休眠进程读请求已完成，数据已准备好。
* **写**：块设备驱动向设备发送写请求、目标地址以及待写入的数据，等待直到设备完成写入。

> [!important]
> **思考**：对于写操作，你能设计出一种机制，让我们无需等待中断，且不会引发一致性问题吗？即 A 发起了写 DATA 到 ADDR 请求后立刻返回，不等待写操作完成，此时 B 读 DATA 可以读出 A 刚刚写入的内容。

### 2.1 本实验具体逻辑

本实验 I/O 的具体逻辑如下：

<figure><img src="/assets/lab4-1.png" alt=""><figcaption></figcaption></figure>

1. 进程调用 `virtio_blk_rw` 向块设备驱动发起读写请求。
2. 块设备驱动（即上述 `virtio_blk_rw`）通过相关接口控制设备。在我们的实验中，块设备驱动主要通过读写特殊寄存器的方式（memory-mapped I/O）来控制设备。
3. 块设备驱动发起请求后，进程休眠，等待请求的完成。我们的内核中并没有提供条件变量，需要我们使用 `Semaphore` 替代条件变量。
4. 经过一段时间后，设备完成读写请求。
5. 设备发起中断，块设备驱动中的中断处理函数处理此中断。
6. 块设备驱动中的中断处理函数**通知**进程请求已完成（即唤醒进程）。

## 3. 设备：`virtio-blk-device`

`virtio-blk-device` 是我们实现块设备驱动的核心。

> [!info]
> **OS (H) Labs 的历史**
>
> 在 2023 秋季及以前版本的实验中，我们面向的平台是树莓派3B，块设备驱动包含了在树莓派上读写 SD 卡的逻辑。然而，[读写 SD 卡的逻辑非常复杂，多达千余行](https://github.com/FDUCSLG/OS-23Fall-FDU/blob/lab5/src/driver/sddef.h)，缺乏可读性和可维护性。因此，从 2024 秋季开始，我们转向 virt 平台（Lab 0 中简要介绍过），极大简化了块设备驱动的逻辑，以期帮助大家更好地理解 I/O 的相关知识。

VirtIO 是一种现代化的虚拟设备接口，它广泛用于虚拟机中，以提供高效的 I/O 设备模拟。本实验中，我们将利用 `virtio-blk-device` 作为我们的块设备。这意味着我们的块设备是一个虚拟的块设备，块设备内的数据由宿主机共同提供。

由于 VirtIO 是一种虚拟设备接口，前端/后端的概念由此引出：

* **前端**：即我们的块设备驱动。
* **后端**：QEMU、KVM 等宿主机组件的逻辑，负责处理内核（运行在 QEMU 虚拟环境中）中的块设备驱动发起的读写请求。

前端和后端通过共享内存交互，前端的块设备驱动向共享内存里写入读写请求，后端的逻辑从共享内存里读出读写请求并处理，将处理结果提供给前端。

前端与后端的共享内存被称为 `virtqueue`（虚拟队列）。对于块设备，VirtIO 使用一个或多个队列来管理请求。每个virtqueue都包含3张表， `Descriptor Table` 存放了 I/O 请求描述符，即I/O 请求的基本信息，`Available Ring` 记录了当前哪些描述符是可用的， `Used Ring` 记录了哪些描述符已经被后端使用了 \[1]。

```
                          +------------------------------------+
                          |       virtio  guest driver         |
                          +-----------------+------------------+
                            /               |              ^
                           /                |               \
                          put            update             get
                         /                  |                 \
                        V                   V                  \
                   +----------+      +------------+        +----------+
                   |          |      |            |        |          |
                   +----------+      +------------+        +----------+
                   | available|      | descriptor |        |   used   |
                   |   ring   |      |   table    |        |   ring   |
                   +----------+      +------------+        +----------+
                   |          |      |            |        |          |
                   +----------+      +------------+        +----------+
                   |          |      |            |        |          |
                   +----------+      +------------+        +----------+
                        \                   ^                   ^
                         \                  |                  /
                         get             update              put
                           \                |                /
                            V               |               /
                           +----------------+-------------------+
                           |	   virtio host backend          |
                           +------------------------------------+
```

在我们的实验中，`virtqueue` 定义如下：

```c
struct virtq {
    struct virtq_desc *desc;
    struct virtq_avail *avail;
    struct virtq_used *used;
    u16 free_head;
    u16 nfree;
    u16 last_used_idx;

    struct {
        volatile u8 status;
        volatile u8 done;
        u8 *buf;
    } info[NQUEUE];
};
```

## 4. 块设备驱动

块设备驱动中发起读写请求的函数是：

```c
int virtio_blk_rw(Buf *b);
```

其中，`Buf *b` 是一个缓冲区指针，表示要进行 I/O 操作的数据块缓冲区。`virtio_blk_rw` 将依据 `b` 的各字段（如 `data` 是块对应的数据，`block_no` 是该块在硬盘上的编号等，`flag == B_DIRTY` 表示是一个写请求），配置上述的 `Descriptor Table` 和 `Available Ring`，向设备发起请求。

块设备驱动中处理中断的函数是：

```c
static void virtio_blk_intr();
```

其负责读取 `Used Ring` 并通知相关进程 I/O 操作已完成，数据已准备好。

此外，我们需要利用好 `disk.virtq.info[d0]` 以进行块设备驱动与设备间的同步。

## 5. 制作启动盘

现代操作系统通常是作为一个硬盘镜像来发布的，我们的也不例外。但由于历史原因，我们的实验在制作镜像时遵循树莓派的规则，即第一个分区为启动分区 （boot partion），文件系统必须为 FAT32，剩下的分区可由我们自由分配。为了简便，我们采用主引导记录（[Master boot record, MBR](https://en.wikipedia.org/wiki/Master_boot_record)） 来进行分区，第一个分区和第二个分区均约为 64 MB，第二分区是根目录所在的文件系统（我们后续需要实现文件系统管理这个分区）。简言之，SD 卡上的布局如下

```
 512B            FAT32      后续实现的文件系统
+-----+-----+--------------+----------------+
| MBR | ... | boot partion | root partition |
+-----+-----+--------------+----------------+
 \   1MB   / \    64MB    / \     63MB     /
  +-------+   +----------+   +------------+
```

### 5.1 MBR

MBR 位于设备的前 512 Byte，有多种格式，不过大同小异，一种常见的格式如下表：

| Address | Description                              | Size (bytes) |
| ------- | ---------------------------------------- | ------------ |
| 0x0     | Bootstrap code area and disk information | 446          |
| 0x1BE   | Partition entry 1                        | 16           |
| 0x1CE   | Partition entry 2                        | 16           |
| 0x1DE   | Partition entry 3                        | 16           |
| 0x1EE   | Partition entry 4                        | 16           |
| 0x1FE   | 0x55                                     | 1            |
| 0x1FF   | 0xAA                                     | 1            |

但这里我们只需要获得第二个分区的信息，即上表中的 Partition entry 2，这 16B 中有该分区的具体信息，包括它的起始 LBA 和分区大小（共含多少块）如下：

| Offset (bytes) | Field length (bytes) | Description                                   |
| -------------- | -------------------- | --------------------------------------------- |
| ...            | ...                  | ...                                           |
| 0x8            | 4                    | LBA of first absolute sector in the partition |
| 0xC            | 4                    | Number of sectors in partition                |

## 6. 任务

> [!important]
> **任务 1**: 完成 `src/driver/virtio_blk.c:119` TODO 内容，通过 Semaphore 等待 I/O 请求完成。
>
> 提示：
>
> 1. 你可以自行为`Buf` 增加所需字段。
> 2. 本任务本质上是用 Semaphore 实现一个条件变量，需要注意条件变量的使用注意事项。

> [!important]
> **任务 2**: 完成 `src/driver/virtio_blk.c:143` TODO 内容，通知 `virtio_blk_rw` 请求已完成。

> [!important]
> **任务 3**: 在 `kernel_entry` 中调用 `virtio_blk_rw` 解析 MBR 获得第二分区起始块的 LBA 和分区大小以便后续使用。

## 7. 提交

**提交方式**：将实验报告提交到 elearning 上，格式为 `学号-lab4.pdf` 。

**注意**：从 `lab1` 开始，用于评分的代码以实验报告提交时为准。如果需要使用新的代码版本，请重新提交实验报告。

**截止时间**：<mark style="color:red;">**11月30日23:59**</mark>。

> [!danger]
>
> **逾期提交将扣除部分分数**
>
> 计算方式为 $\text{score}_{\text{final}} = \text{score} \cdot \left(1 - n \cdot 20\% \right)$，其中 $n$ 为迟交天数，不满一天按一天计算）。

报告中可以包括下面内容

* 代码运行效果展示
* 实现思路和创新点
* 对后续实验的建议
*   其他任何你想写的内容

    > ~~你甚至可以再放一只可爱猫猫~~

报告中不应有大段代码的复制。如有使用本地环境进行实验的同学，请联系助教提交代码。使用服务器进行实验的同学，助教会在服务器上检查，不需要另外提交代码。

在服务器上操作的同学，此次实验完成后请提交（或者说创建一个新分支）到 `lab4-submission` 分支，助教会使用你在此分支上提交记录来批作业。如果此分支最后提交时间晚于实验报告提交时间，助教会选择此分支上在实验报告提交时间前的最后一个提交作为批改代码。

**提交操作**：

```shell
# 提交最后的代码
git add .
git commit -m "your final commit message"

# 新建一个分支，用于提交
git checkout -b lab4-submission
```

## 8. 参考资料

**\[1]** 简单了解一下Virtio Spec协议 [https://www.openeuler.org/zh/blog/yorifang/virtio-spec-overview.html](https://www.openeuler.org/zh/blog/yorifang/virtio-spec-overview.html)
