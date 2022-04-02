
# 背景

验证k8s使用的持久存决方案的磁盘性能

# 基础知识

## 磁盘性能指标

1.  IOPS

> 每秒输入/输出操作，最长见到指标是4K随机读取和写入。

2.  吞吐量

> 每秒读写多少MB数据，常见单位是MB/秒。磁盘的吞吐量是有瓶颈的，首先是物理介质，其次是磁盘接口速度，例如SATA：6Gb，SAS：22.5Gb

3.  延时

> 传统旋转磁盘关注此参数，寻道和旋转延时通常15ms-25ms.例如每次随机查找4KB延时0.016秒= 250KB/秒

4.  缓存

> 写缓存再批量写入磁盘。例如：1MB数据是256个4KB，如果每一个写入可能占用磁盘1秒，但是用缓存，可能占用磁盘0.01秒

## 磁盘接口比较

> 队列深度：如果I/O请求的数量超过了最大队列深度，则该事务将在一段时间后无法重新尝试。


| 磁盘接口 | 带宽 | 队列深度 | 工作模式 | 计算机接口 |
| ---- | ---- | ---- | ---- | ---- | 
|STAT |6Gb |32 |能同时执行读写功能 |主机总线适配器（HBA） |
|SAS |22.5Gb |254 |不能同时执行读写功能 |主机总线适配器（HBA） |
|NVMe |PCIe5x1: 30Gb，PCIe5x4: 126Gb |65000 |能同时执行读写功能 |PCIe-直接与 CPU 通信 |
# 压测工具

 https://fio.readthedocs.io/en/latest/ 

## FIO安装

```
#yum 安装
yum install libaio-devel fio

```

## FIO参数详解

### iodepth

![iodepth](https://github.com/231397220/fio/blob/main/iodepth.png)

增加队列深度，可以看到IOPS不会随着队列深度的增加而一直增加，达到一定值后增幅会有所下降。

增加队列深度，可以测试出磁盘的峰值。

### bs

![bs](https://github.com/231397220/fio/blob/main/bs.png)

增加bs大小，可以测试出磁盘带宽峰值。

减小bs大小，可以测试出磁盘IOPS峰值。

### Numjobs
![numjobs](https://github.com/231397220/fio/blob/main/numjobs.png)
numjos与iodepth表现相同。
增加并发数，可以测试出磁盘的峰值。


## FIO输出详解

Fio 将压缩线程字符串，以免在命令行上占用比需要更多的空间。例如，如果您有 10 个读取器和 10 个写入器正在运行，则输出将如下所示：

```
Jobs: 20 (f=20): [R(10),W(10)][4.0%][r=20.5MiB/s,w=23.5MiB/s][r=82,w=94 IOPS][eta 57m:36s]

```

请注意，状态字符串是按顺序显示的，因此可以知道哪些作业当前正在做什么。在上面的示例中，这意味着作业 1-10 是读取器，而 11-20 是写入器。

其他值很容易解释——当前运行和执行 I/O 的线程数、当前打开文件的数量 (f=)、估计的完成百分比、自上次检查以来的 I/O 率（首先列出的读取速度，然后写入速度和可选的修剪速度）在带宽和 IOPS 方面，以及当前运行组的完成时间。无法估计以下组的运行时间（如果有）。

当 fio 完成（或被 中断Ctrl-C）时，它将按顺序显示每个线程、线程组和磁盘的数据。对于每个整体线程（或组），输出如下所示：

> 关键结果数据：加粗红色数据

```
Client1: (groupid=0, jobs=1): err= 0: pid=16109: Sat Jun 24 12:07:54 2017
  write: IOPS=88, BW=623KiB/s (638kB/s)(30.4MiB/50032msec)
    slat (nsec): min=500, max=145500, avg=8318.00, stdev=4781.50
    clat (usec): min=170, max=78367, avg=4019.02, stdev=8293.31
     lat (usec): min=174, max=78375, avg=4027.34, stdev=8291.79
    clat percentiles (usec):
     |  1.00th=[  302],  5.00th=[  326], 10.00th=[  343], 20.00th=[  363],
     | 30.00th=[  392], 40.00th=[  404], 50.00th=[  416], 60.00th=[  445],
     | 70.00th=[  816], 80.00th=[ 6718], 90.00th=[12911], 95.00th=[21627],
     | 99.00th=[43779], 99.50th=[51643], 99.90th=[68682], 99.95th=[72877],
     | 99.99th=[78119]
   bw (  KiB/s): min=  532, max=  686, per=0.10%, avg=622.87, stdev=24.82, samples=  100
   iops        : min=   76, max=   98, avg=88.98, stdev= 3.54, samples=  100
  lat (usec)   : 250=0.04%, 500=64.11%, 750=4.81%, 1000=2.79%
  lat (msec)   : 2=4.16%, 4=1.84%, 10=4.90%, 20=11.33%, 50=5.37%
  lat (msec)   : 100=0.65%
  cpu          : usr=0.27%, sys=0.18%, ctx=12072, majf=0, minf=21
  IO depths    : 1=85.0%, 2=13.1%, 4=1.8%, 8=0.1%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwt: total=0,4450,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=8

```

打印作业名称（或使用时的第一个作业名称`**[group_reporting](https://fio.readthedocs.io/en/latest/fio_man.html#cmdoption-arg-group-reporting)**`），以及组 ID、正在聚合的作业计数、最后看到的错误 ID（当没有错误时为 0）、该线程的 pid/tid 和时间工作/组完成。下面是执行的每个数据方向的 I/O 统计信息（显示上面示例中的写入）。按列出的顺序，它们表示：

**write**

冒号前的字符串显示统计信息的 I/O 方向。 **IOPS**是每秒执行的平均 I/O。 **BW** 是平均带宽速率，显示为：2 格式的幂值（10 格式的幂值）。最后两个值显示：（以 2 格式**执行的总 I/O / 该线程的运行时间**）。

**slat**

提交延迟（**min**是最小值，**max**是最大值，**avg**是平均值，**stdev**是标准偏差）。这是提交 I/O 所花费的时间。对于同步 I/O，此行不显示，因为 slat 实际上是完成延迟（因为队列/完成是那里的一项操作）。该值可以以纳秒、微秒或毫秒为单位——fio 将选择最合适的基数并打印（在上面的示例中，纳秒是最佳比例）。注意：在`**[--minimal](https://fio.readthedocs.io/en/latest/fio_man.html#cmdoption-minimal)**`模式下延迟总是以微秒表示。

**clat**

完成延迟。与 slat 同名，表示从提交到完成 I/O 片段的时间。对于同步 I/O，clat 通常等于（或非常接近）0，因为从提交到完成的时间基本上只是 CPU 时间（I/O 已经完成，参见 slat 说明）。

**lat**

总延迟。与 slat 和 clat 同名，表示从 fio 创建 I/O 单元到完成 I/O 操作的时间。

**bw**

基于样本的带宽统计。与 xlat stats 名称相同，但还包括采样的数量 ( **samples** ) 和该线程在其组中接收的总聚合带宽的近似百分比 ( **per** )。仅当该组中的线程在同一个磁盘上时，最后一个值才真正有用，因为它们会竞争磁盘访问。

**IOPS**

基于样本的 IOPS 统计。与 bw 同名。

**lat (nsec纳秒/usec微妙/****msec****毫秒)**

I/O 完成延迟的分布。这是从 I/O 离开 fio 到完成的时间。与上面单独的读/写/修剪部分不同，此处和其余部分中的数据适用于报告组的所有 I/O。250=0.04% 表示 0.04% 的 I/O 在 250us 内完成。500=64.11% 表示 64.11% 的 I/O 需要 250 到 499us 才能完成。

**CPU**

CPU使用率。用户和系统时间，以及此线程经历的上下文切换次数，系统和用户时间的使用情况，最后是主要和次要页面错误的数量。CPU 利用率数字是该报告组中作业的平均值，而上下文和故障计数器相加。

**IO****depths**

I/O 深度在作业生命周期内的分布。这些数字除以 2 的幂，每个条目涵盖从该值到低于下一个条目的深度 - 例如，16= 涵盖从 16 到 31 的深度。请注意，深度分布条目所涵盖的范围可以是不同于等效的提交/完成分发条目所涵盖的范围。

**IO****submit**

在单个提交调用中提交了多少 I/O。每个条目表示该数量及以下，直到前一个条目——例如，16=100% 意味着我们每次提交调用提交了 9 到 16 个 I/O。请注意，提交分布条目所涵盖的范围可能与等效深度分布条目所涵盖的范围不同。

**IO****complete**

与上面的提交编号类似，但用于完成。

**IO** **issued rwts**

发出的读/写/修剪请求的数量，以及其中有多少是短的或丢弃的。

**IO** **latency**

这些值用于`**[latency_target](https://fio.readthedocs.io/en/latest/fio_man.html#cmdoption-arg-latency-target)**`和相关选项。当使用这些选项时，本节将描述满足指定延迟目标所需的 I/O 深度。

列出每个客户端后，将打印组统计信息。它们看起来像这样：

Run status group 0 (all jobs):

READ: bw=20.9MiB/s (21.9MB/s), 10.4MiB/s-10.8MiB/s (10.9MB/s-11.3MB/s), io=64.0MiB (67.1MB), run=2973-3069msec

WRITE: bw=1231KiB/s (1261kB/s), 616KiB/s-621KiB/s (630kB/s-636kB/s), io=64.0MiB (67.1MB), run=52747-53223msec

对于它打印的每个数据方向：

**bw**

该组中线程的聚合带宽，后跟该组中所有线程的最小和最大带宽。括号外的值是 2 的幂格式，括号内的值是 10 的幂格式的等效值。

**io**

对该组中所有线程执行的聚合 I/O。格式与 bw 相同。

**run**

该组中线程的最小和最长运行时间。

最后，打印磁盘统计信息。这是特定于 Linux 的。它们看起来像这样：

Disk stats (read/write):

sda: ios=16398/16511, merge=30/162, ticks=6853/819634, in_queue=826487, util=100.00%

为读取和写入打印每个值，首先读取。数字表示：

**IOS**

所有组执行的 I/O 数。

**merge**

I/O 调度程序执行的合并数。

**ticks**

我们保持磁盘繁忙的滴答数。

**in_queue**

在磁盘队列中花费的总时间。

**util**

磁盘利用率。值 100% 意味着我们一直保持磁盘繁忙，50% 将是磁盘空闲时间的一半。

也可以让 fio 在运行时转储当前输出，而无需终止作业。为此，请发送 fio **USR1**信号。您还可以通过使用参数或通过在命名 `**[--status-interval](https://fio.readthedocs.io/en/latest/fio_man.html#cmdoption-status-interval)**` 的文件中创建文件来定期获取转储。如果 fio 看到这个文件，它将取消链接并转储当前的输出状态。`/tmpfio-dump-status`

## Longhorn vs Rook vs OS 压测

### 环境信息
- K8S： 1.20
- Longhorn： 1.2.3
- Fio: 3.7

### 压测标准

*   异步I/O
*   IO深度：随机32，顺序16
*   并发数：随机8，顺序4
*   禁用缓存

### 压测命令

```
git clone https://github.com/231397220/fio.git fio-example
fio fio-examle/base-fio.conf
```

### 压测数据
![pr_data](https://github.com/231397220/fio/blob/main/pr_data.png)

#### 通用场景： 压测命令

```
fio --directory=/data/test --name fio_test_file --direct=1 --rw=randread --bs=16k --size=1G --numjobs=16 --time_based --runtime=180 --group_reporting --norandommap

```
