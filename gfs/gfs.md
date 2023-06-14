# GFS 论文阅读笔记

## What is GFS

> Google File System.
>
> A scalable distributed file system for large distributed  data-intensive applications.
>
> Fault tolerance、High aggregate preformance.

一个面向大规模数据密集型应用的、可伸缩的分布式文件系统，提供灾备能力，保证高性能的服务

> Our design has been driven by observations of our application workloads and technological environment, both current and anticipated, that reflflect a marked departure from some earlier file system assumptions.

根据 Google 内部应用的负载和技术分析，与传统的分布式文件系统有很多**明显的不同**

到底存在哪些不同？如传统分布式文件系统的特点？有哪些缺点？GFS 解决了哪些问题？



## Introduction

与传统的分布式文件系统在性能、可扩展性、可靠性、可用性上保持一致的目标

> GFS shares many of the same goals as previous distributed file systems such as performance, scalability, reliability, and availability.

不用之处主要有以下四点：

1. **组件失效被认为是常态事件**：遇到过各种各样的问题，比如 应用程序bug、操作系统的bug、人为失误，甚至还有硬盘、内存、连接器、网络以及电源失效等造成的 问题。所以，持续的监控、错误侦测、灾难冗余以及自动恢复的机制必须集成在GFS中。 

>Component failures are the norm rather than the exception.
>
>Due to problems caused by application bugs, operating system bugs, human errors, and the failures of disks, memory, connectors, networking, and power supplies.

2. **以通常的标准衡量，我们的文件非常巨大**：在采用数亿个 KB 大小的文件管理方式是非常不明智的，需要重新考虑对应的尺寸。

> Files are huge by traditional standards.
>
> When we are regularly working with fast growing data sets of many TBs comprising billions of objects, it is unwieldy to manage billions of approximately KB-sized files even when the file system could support it.

3. **绝大部分文件的修改是采用在文件尾部追加数据，而不是覆盖原有数据的方式**：对文件的随机写入操作在实际中几乎不存在。一旦写完之后，对文件的操作就只有读，而且通常是按顺序读。通常来说针对这中文件数据的访问，客户端对数据块缓存是没有意义的，数据的追加操作是性能优化和原子性保证的主要考量因素。

>Most files are mutated by appending new data rather than overwriting existing data.
>
>Random writes within a file are practically non-existent. Once written, the files are only read, and often only sequentially.

4. **应用程序和文件系统API的协同设计提高了整个系统的灵活性**： 如，我们放松了对GFS一致性模型的要求，这样就减轻了文件系统对应用程序的苛刻要求，大大简化了GFS的设计。

> Co-designing the applications and the file system API benefits the overall system by increasing our flexibility.
>
> We have also introduced an atomic append operation so that multiple clients can append concurrently to a file without extra synchronization between them.



简单总结一下：

- 增加系统的容错性
- 大文件数据处理
- 在文件尾部追加数据很常见，而非随机写或覆盖写
- 降低一致性预期的 API 设计



## Design Overview

### Assumptions

设计之初，对系统的预期目标如下：

1. **系统由许多廉价的普通组件组成，组件失效是一种常态**：失效作为一种常态，则需要系统本身必须有监控自身的能力，能够快速的发现并恢复失效的组件。

> The system is built from many inexpensive commodity components that often fail. It must constantly monitor itself and detect, tolerate, and recover promptly from component failures on a routine basis.

2. **系统能够存储一定量的大文件**：100MB 或者 GB 大小的文件支持，并且能够做到有效的管理，同时也可以兼容小文件。

> The system stores a modest number of large files. We expect a few million files, each typically 100 MB or larger in size. Multi-GB fifiles are the common case and should be managed efficiently. Small files must be supported, but we need not optimize for them.

3. **系统的工作负载主要由两种读操作组成**：大规模的流式读取和小规模的随机读取。

   1. 流式读取：通常是读取同一个文件中连续的一个区域，大规模，如数百 KB 或者 MB 的读取。

   2. 随机读取：通常是在文件某个随机位置读取，通常的做法是把小规模的随机读取操作合并并排序， 之后按顺序批量读取，这样就避免了在文件中前后来回的移动读取位置。 

> The workloads primarily consist of two kinds of reads: large streaming reads and small random reads.
>
> In large streaming reads, individual operations typically read hundreds of KBs, more commonly 1 MB or more.
>
> A small random read typically reads a few KBs at some arbitrary offset. 

4. **系统的工作负载还包括许多大规模的、顺序的、数据追加方式的写操作**：每次写入的数据的大小和大规模读类似。数据一旦被写入后，文件就很少会被修改了。系统支持小规模的随机位置写入操作，但是可能效率不高。

> The workloads also have many large, sequential writes that append data to files. Typical operation sizes are similar to those for reads. Once written, files are seldom modified again. Small writes at arbitrary positions in a file are supported but do not have to be efficient.

5. **系统必须高效的、行为定义明确的实现多客户端并行追加数据到同一个文件里的语义**：用最小的同步开销来实现的原子的多路追加数据操作是必不可少的。文件可以在稍后读取，或者是消费者在追加的操作的同时读取文件。 

> The system must efficiently implement well-defifined  semantics for multiple clients that concurrently append to the same file.

6. **高性能的稳定网络带宽远比低延迟重要**：目标程序绝大部分要求能够高速率的、大批量的处理 数据，极少有程序对单一的读写操作有严格的响应时间要求。

> High sustained bandwidth is more important than low latency.

### Interface

GFS 提供了一套类似传统文件系统的API接口函数，支持常用的操作，如创建新文件、删除文件、打开文件、关闭文件、读和写文件。 

 此外，GFS 提供了快照和记录追加操作。

- 快照以很低的成本创建一个文件或者目录树的拷贝。
- 记录追加操作允许多个客户端同时对一个文件进行数据追加操作，同时保证每个客户端的追加操作都是原子性的。



### Architecture

一个 GFS 集群包含一个单独的 Master 节点、多台 Chunk 服务器，并且同时被多个客户端访问，如图所示。

<img src="./image/image-gfs-arch.png" alt="image-20230614155510770" style="zoom:67%;" />

GFS 存储的文件都被分割成固定大小的 Chunk。在 Chunk 创建的时候，Master 服务器会给每个 Chunk 分配一个不变的、全球唯一的64位的 Chunk 标识。Chunk 服务器把 Chunk 以 Linux 文件的形式保存在本地硬盘上，并且根据指定的 Chunk 标识和字节范围来读写块数据。出于可靠性的考虑，每个块都会复制到多个块服务器上（通常为 3）。

 Master 节点管理所有的文件系统元数据。这些元数据包括名字空间、访问控制信息、文件和 Chunk 的映射信息、以及当前 Chunk 的位置信息。Master 节点还管理着系统范围内的活动，比如，Chunk 租用管理、孤儿 Chunk 的回收、以及 Chunk 在 Chunk 服务器之间的迁移。Master 节点使用心跳信息周期地和每个 Chunk 服务器通讯，发送指令到各个 Chunk 服务器并接收 Chunk 服务器的状态信息。 

GFS 客户端代码以库的形式被链接到客户程序里。客户端代码实现了 GFS 文件系统的API接口函数、应用程序与 Master 节点和 Chunk 服务器通讯、以及对数据进行读写操作。客户端和 Master 节点的通信只获取元数据，所有的数据操作都是由客户端直接和 Chunk 服务器进行交互的。

无论是客户端还是 Chunk 服务器都不需要缓存文件数据。客户端缓存数据几乎没有什么用处，因为大部分程序要么以流的方式读取一个巨大文件，要么工作集太大根本无法被缓存。无需考虑缓存相关的问题也简化了客户端和整个系统的设计和实现。(不过，客户端会缓存元数据)。Chunk 服务器不需要缓存文件数据的原因是，Chunk 以本地文件的方式保存，Linux 操作系统的文件系统缓存会把经常访问的数据缓存在内存中。 

