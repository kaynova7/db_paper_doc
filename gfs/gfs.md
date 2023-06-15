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

GFS 提供了一套类似传统文件系统的 API 接口函数，支持常用的操作，如创建新文件、删除文件、打开文件、关闭文件、读和写文件。 

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



### Singe Master

特点：

- 简化整体的设计
- 可以通过全聚德信息精确定位 Chunk 的位置，以及进行复制决策
- 必须减少对 Master 节点的读写，避免 Master 节点成为系统的瓶颈
- 客户端节点并不通过 Master 节点读写文件数据，只是向 Master 节点询问它听该联系的 Chunk 服务器
- 客户端主要缓存 Chunk 的元数据，后续操作可以直接和 Chunk 服务器进行数据读写操作

> Clients never read and write file data through the master. Instead, a client asks the master which chunkservers it should contact. It caches this information for a limited time and interacts with the chunkservers directly for many subsequent operations.



一次简单的读取流程如下：

1. 客户端把文件名和程序制定的字节偏移，根据固定 Chunk 的大小，转换为稳健的 Chunk 索引
2. 客户端将文件名和 Chunk 索引发送给 Master 节点，获取相应 Chunk 标识的副本的位置信息
3. 客户端将获取到的元数据信息使用文件名和 Chunk 索引作为 Key 缓存这些信息
4. 客户端将发送请求（读写）到一个最近的 Chunk 副本处，请求信息包含了 Chunk 的标识和字节范围

> 实际上，客户端通常会在一次请求中查询多个 Chunk 信息，Master 节点的回应也可 能包含了紧跟着这些被请求的 Chunk 后面的 Chunk 的信息。在实际应用中，这些额外的信息在没有任何代 价的情况下，避免了客户端和 Master 节点未来可能会发生的几次通讯。



### Chunk Size

默认为 64MB，并以普通 Linux 文件的形式保存在 Chunk 服务器上，只有在有需要的时候才扩大。

优点：

- 减少客户端与 Master 节点进行通讯的需求（毕竟文件大了 chunk 信息就少了==）
- 客户端能够对一个块进行多次操作，可以保持较长的 TCP 链接来减少网络负载

> Chunk 小尺寸情况下可能分散在多个 Chunk 中的操作现在聚集在同一个 Chunk 中。这使得 Client 与相应 Chunk Server 保持更久的连接，而非在多个 Chunk Server 间切换连接。

- 减少了 Master 节点需要保存的元数据数量（同第一点）

缺点：

- 小文件包含较少的Chunk，甚至只有一个Chunk。当有许多的客户端对同一个小文件进行多次的访问时，存储这些Chunk的Chunk服务 器就会变成热点。（即使是惰性空间分配）



### Metadata

包括：文件和 Chunk 的命名空间、文件和 Chunk 的对应关系、每个 Chunk 副本的存放地点。

以上所有的元数据都保存在 Master 服务器的内存中。

> 前两种类型的元数据(命名空间、文件和 Chunk 的对应关系)同时也会以记录变更日志的方式记录在操作系统的系统日志文件中，日志文件存储在本地磁盘上，同时日志会被复制到其它的远程 Master 服务器上。
>
> 采用保存变更日志的方式，我们能够简单可靠的更新 Master 服务器 的状态，并且不用担心 Master 服务器崩溃导致数据不一致的风险。
>
> Master 服务器启动后，会向各个 Chunk 服务器轮询它们所存储的 Chunk 信息

1. In-Memory Data Structures

>  因为元数据保存在内存中，所以 Master 服务器的操作速度非常快。
>
> Master 服务器可以在后台简单而高效的周期性扫描自己保存的全部状态信息。这种周期性的状态扫描也用于实现 Chunk 垃圾收集、在 Chunk 服务器失效的时重新复制数据、通过 Chunk 迁移实现跨 Chunk 服务器的负载均衡及磁盘使用状况统计等功能。

2. Chunk Locations

   > Master 服务器并不持久化保存哪个 Chunk 服务器存有指定 Chunk 的副本的信息。Master 服务器只是在启动的时候轮询 Chunk服务器以获取这些信息。Master 服务器能够保证它持有的信息始终是最新的，因为它控制了所有的 Chunk 位置的分配，而且通过周期性的心跳信息监控 Chunk 服务器的状态。

3. Operation Log

   > 操作日志包含了关键的元数据变更历史记录。这对 GFS 非常重要。这不仅仅是因为操作日志是元数据唯一的持久化存储记录，它也作为判断同步操作顺序的逻辑时间基线(，都由它们创建的逻辑时间唯一的、永久的标识。
   >
   > Master 服务器在灾难恢复时，通过重演操作日志把文件系统恢复到最近的状态。
   >
   > Master 服务器在日志增长到一定量时对系统状态做一个 Checkpoint，然后恢复的之后从制定 Checkpoint 开始恢复状态。



### Consistency Model

GFS 支持一个宽松的一致性模型，这个模型能够很好的支撑我们的高度分布的应用，同时还保持了相对简单且容易实现的优点。

1. ### GFS一致性保障机制 

​	文件命名空间的修改(例如，文件创建)是原子性的。它们仅由 Master 节点的控制:命名空间锁提供了原子性和正确性(4.1章)的保障；Master节点的操作日志定义了这些操作在全局的顺序(2.6.3章)。 

<img src="./image/image-gfs-consisency.png" alt="img" style="zoom:67%;" />

> 这张表从下到上对并发修改的要求逐渐增高，Failure 表示并发修改失败，Concurrent success 表示并发修改成功，Serial success 则表示串行修改成功，串行要求最高，但是其如同单线程写一样不存在任何并发问题。 

首先我们应当理解 GFS 系统中对 file region 状态的概念定义：

- consistent：所有 GFS Client 将总是看到完全相同的数据，无论 GFS Client 最终是从哪一个 GFS chunkserver replica 上进行数据读取；

> A file region is consistent if all clients will always see the same data, regardless of which replicas they read from.

- defined：当一个文件数据修改之后如果 file region 还是保持 consistent 状态，并且所有 client 能够看到全部修改（且已经写入 chunkserver）的内容；

>  A region is ***defined*** after a fifile data mutation if it is consistent and clients will see what the mutation writes in its entirety. When a mutation succeeds without interference from concurrent writers, the affected region is defifined (and by implication consistent): all clients will always see what the mutation has written.

- consistent but undefined：从定义上来看，就是所有 client 能够看到相同的数据，但是并不能及时反映并发修改中的任意修改；

> All clients see the same data, but it may not reflect what any one mutation has written.

- inconsistent：因为处于 inconsistent 状态，因此一定也处于 undeﬁned 状态，造成此状态的操作也被认为是 failed 的。不同的 Client 在能读到的数据不一致，同一个 Client 在不同的时刻读取的文件数据也不一致。

> Different clients may see different data at different times.

其次数据修改操作分为写入或者记录追加两种：

> A write causes data to be written at an application-specified file offset. 
>
> A record append causes data (the “record”) to be appended atomically at least once even in the presence of concurrent mutations, but at an offset of GFS’s choosing.

- Write：修改 File 中的原有数据，具体来说就是在指定文件的偏移地址下写入数据（这就是覆写操作）；

  > GFS 没有为这类覆写操作提供完好的一致性保证：如果多个的 Client 并发地写入同一块文件区域，操作完成后这块区域的数据可能由各次写入的数据碎片所组成，此时的状态最好也就是 consistant but undefined 状态。

- Record Append：即在原有 File 末尾 Append (追加)数据，这种操作被 GFS 系统确保为原子操作，这是 GFS 系统最重要的优化之一。GFS 中的 append 操作并不简单地在原文件的末尾对应的 offset 处开始写入数据（这是通常意义下的 append 操作），而是通过选择一个 offset，这一点在下面会详细说到。最后该被选择的 offset 会返回给 Client，代表此次 record 的起始数据偏移量。由于 GFS 对于 Record Append 采用的是 at least once 的消息通信模型，在绝对确保此次写操作成功的情况下，**可能造成在重复写数据**。



Notes：在一系列成功的修改操作以后，被修改的文件区域的状态是 **defined** 并包含最后一次修改的写内容。GFS 通过以下两种方式实现这一目标：

- 在所有写操作相关的 replicas 上以同一顺序采用给 chunk 进行修改；

- 使用 chunk version numbers（也就是版本号）去检测 replicas 上的数据是否已经 stale（过时），这种情况可能是由于 chunkserver 出现暂时的宕机(down)；

  > 注意，一旦一个 replicas 被判定为过时，那么 GFS 就不会基于此 replicas 进行任何修改操作，客户机再向 Master 节点请求元数据时，也会自动滤除过时的 replicas。并且 Master 通常会及时地对过时的 replicas 进行 garbage collected（垃圾回收）。



出现的相关问题：

- 缓存未过期时 replica 出现过时的问题：因为在客户机存在缓存 cache 的缘故，在缓存被刷新之前，客户机还是有机会从 stale replica 上读取文件数据。这个时间穿窗口取决于缓存的超时时间设置以及下一次打开同一文件的限制。另一方面，GFS 中的大多数文件都是 append-only，因此 stale replica 上的读仅仅是返回一个 premature end of chunk，也就是说仅仅没有包含最新的追加内容的 chunk，而不是被覆写了的数据（因为无法覆写），这样造成的影响也不会很大。**当一个Reader 重新尝试并联络 Master 服务器时，它就会立刻得到最新的 Chunk 位置信息。** 

> When a reader retries and contacts the master, it will immediately get current chunklocations.

- 组件故障问题：Master 节点进行通过与所有 chunkserver 进行 regular handshake（定期握手）来检测出现故障的 chunkserver，通过 checksumming（校验和）来检测数据是否损坏。一旦出现问题，GFS 会尽快地从有效的 replicas 上进行数据恢复，除非在 Master 节点检测到故障之前，存储相同内容的 3 块 replica 都出现故障时才会导致不可逆的数据丢失。**不过即使是这样，GFS 系统也不会不可用，而是会及实地给客户端回应数据出错的响应，而不是返回出错的数据。** 

> Even in this case, it be comes unavailable, not corrupted: applications receive clear errors rather than corrupt data.



2. ### 程序的实现

   使用GFS的应用程序可以利用一些简单技术实现这个宽松的一致性模型，这些技术也用来实现一些其它的目标功能，包括:尽量采用追加写入而不是覆盖，Checkpoint，自验证的写入操作，自标识的记录。比如：

   - 尽量选择 append 追加，而不是 overwrite 覆写，这是因为 GFS 仅仅保证 append 操作的一致性，但是覆写操作没有一致性保证；
   - 写入数据时使用额外的校验信息，比如校验和（或者其他 hash 技术）；
   - 可以选择加入额外的唯一标识符来去除因为 write at least once 而造成的重复数据（直白一点就是客户端在读取数据时如果发现唯一标识符已经读过了，那么就舍弃这部分数据）；



##  SYSTEM INTERACTIONS

一个原则：最小化所有操作和Master节点的交互

三个主体功能：数据修改操作、原子的记录追加操作以及快照



### Leases and Mutation Order

变更是一个会改变 Chunk 内容或者元数据的操作，比如写入操作或者记录追加操作。变更操作会在 Chunk 的所有副本上执行。我们使用租约 (lease) 机制来保持多个副本间变更顺序的一致性。Master节点为 Chunk的一个副本建立一个租约，我们把这个副本叫做主 Chunk。主 Chunk 对 Chunk 的所有更改操作进行序列化。所有的副本都遵从这个序列进行修改操作。因此，修改操作全局的顺序首先由 Master节点选择的租约的顺序决定，然后由租约中主 Chunk 分配的序列号决定。 

   **设计租约机制的目的是为了最小化Master节点的管理负担**。租约的初始超时设置为60秒。不过，只要 Chunk被修改了，主Chunk就可以申请更长的租期，通常会得到Master节点的确认并收到租约延长的时间。这些租约延长请求和批准的信息通常都是附加在Master节点和Chunk服务器之间的心跳消息中来传递。有时Master节点会试图提前取消租约(例如，Master节点想取消在一个已经被改名的文件上的修改 操作)。即使Master节点和主Chunk失去联系，它仍然可以安全地在旧的租约到期后和另外一个Chunk 副本签订新的租约。 在图2中，我们依据步骤编号，展现写入操作的控制流程。 

![img](https://nxwz51a5wp.feishu.cn/space/api/box/stream/download/asynccode/?code=NDNjMmRmNGNkZjc4YWZlZWRkNGMwMmY0MmJjZDU4YzZfc3lUYUh3Y1dOM29HYnFiSTNUVjZvM0V3bE45R2gzRExfVG9rZW46Ym94Y25VWW5FbW9Kc1FhNmVEUmdraUZ3MXFoXzE2ODY4MDgyNjM6MTY4NjgxMTg2M19WNA)



1.  客户机向Master节点询问哪一个Chunk服务器持有当前的租约，以及其它副本的位置。如果没有一 个Chunk持有租约，Master节点就选择其中一个副本建立一个租约(这个步骤在图上没有显示)。 
2. Master节点将主Chunk的标识符以及其它副本(又称为secondary副本、二级副本)的位置返回给 客户机。客户机缓存这些数据以便后续的操作。只有在主Chunk不可用，或者主Chunk回复信息表明它已不再持有租约的时候，客户机才需要重新跟Master节点联系。
3.  客户机把数据推送到所有的副本上。客户机可以以任意的顺序推送数据。Chunk服务器接收到数据并保存在它的内部LRU缓存中，一直到数据被使用或者过期交换出去。由于数据流的网络传输负载非常高，通过分离数据流和控制流，我们可以基于网络拓扑情况对数据流进行规划，提高系统性能，而不用去理会哪个Chunk服务器保存了主Chunk。3.2章节会进一步讨论这点。  
4. 当所有的副本都确认接收到了数据，客户机发送写请求到主Chunk服务器。这个请求标识了早前推送到所有副本的数据。主Chunk为接收到的所有操作分配连续的序列号，这些操作可能来自不同的客户机，序列号保证了操作顺序执行。它以序列号的顺序把操作应用到它自己的本地状态中(alex 注:也就是在本地执行这些操作，这句话按字面翻译有点费解，也许应该翻译为“它顺序执行这些操作，并更新自己的状态”) 
5. 主Chunk把写请求传递到所有的二级副本。每个二级副本依照主Chunk分配的序列号以相同的顺序执行这些操作。
6. 所有的二级副本回复主Chunk，它们已经完成了操作 
7. 主Chunk服务器(alex注:即主Chunk所在的Chunk服务器)回复客户机。任何副本产生的任何错误都会返回给客户机。在出现错误的情况下，写入操作可能在主Chunk和一些二级副本执行成 功。(如果操作在主Chunk上失败了，操作就不会被分配序列号，也不会被传递。)客户端的请求 被确认为失败，被修改的region处于不一致的状态。我们的客户机代码通过重复执行失败的操作来处理这样的错误。在从头开始重复执行之前，客户机会先从步骤(3)到步骤(7)做几次尝试。 

   如果应用程序一次写入的数据量很大，或者数据跨越了多个Chunk，GFS客户机代码会把它们分成多个写 操作。这些操作都遵循前面描述的控制流程，但是可能会被其它客户机上同时进行的操作打断或者覆盖。 因此，共享的文件region的尾部可能包含来自不同客户机的数据片段，尽管如此，由于这些分解后的写入 操作在所有的副本上都以相同的顺序执行完成，Chunk的所有副本都是一致的。这使文件region处于2.7 节描述的一致的、但是未定义的状态。 



### Data Flow

  为了提高网络效率，我们采取了把数据流和控制流分开的措施。在控制流从客户机到主Chunk、然后再到所有二级副本的同时，数据以管道的方式，顺序的沿着一个精心选择的Chunk服务器链推送。我们的目标是充分利用每台机器的带宽，避免网络瓶颈和高延时的连接，最小化推送所有数据的延时。 

​           为了充分利用每台机器的带宽，数据沿着一个Chunk服务器链顺序的推送，而不是以其它拓扑形式分散推送(例如，树型拓扑结构)。线性推送模式下，每台机器所有的出口带宽都用于以最快的速度传输数据， 而不是在多个接受者之间分配带宽。 

 为了尽可能的避免出现网络瓶颈和高延迟的链接(eg，inter-switch最有可能出现类似问题)，每台机器都尽量的在网络拓扑中选择一台还没有接收到数据的、离自己最近的机器作为目标推送数据。假设客户机 把数据从Chunk服务器S1推送到S4。它把数据推送到最近的Chunk服务器S1。S1把数据推送到S2，因 为S2和S4中最接近的机器是S2。同样的，S2把数据传递给S3和S4之间更近的机器，依次类推推送下 去。我们的网络拓扑非常简单，通过IP地址就可以计算出节点的“距离”。 

​        最后，我们利用基于TCP连接的、管道式数据推送方式来最小化延迟。Chunk服务器接收到数据后，马上开始向前推送。管道方式的数据推送对我们帮助很大，因为我们采用全双工的交换网络。接收到数据后立刻向前推送不会降低接收的速度。在没有网络拥塞的情况下，传送B字节的数据到R个副本的理想时间是 B/T+RL ，T是网络的吞吐量，L是在两台机器数据传输的延迟。通常情况下，我们的网络连接速度是 100Mbps(T)，L将远小于1ms。因此，1MB的数据在理想情况下80ms左右就能分发出去。 



### Atomic Record Appends

 GFS提供了一种原子的数据追加操作–记录追加。传统方式的写入操作，客户程序会指定数据写入的偏移量。对同一个region的并行写入操作不是串行的:region尾部可能会包含多个不同客户机写入的数据片段。使用记录追加，客户机只需要指定要写入的数据。GFS保证至少有一次原子的写入操作成功执行(即 写入一个顺序的byte流)，写入的数据追加到GFS指定的偏移位置上，之后GFS返回这个偏移量给客户机。这类似于在Unix操作系统编程环境中，对以O_APPEND模式打开的文件，多个并发写操作在没有竞态条件时的行为。 

​        记录追加在我们的分布应用中非常频繁的使用，在这些分布式应用中，通常有很多的客户机并行地对同一个文件追加写入数据。如果我们采用传统方式的文件写入操作，客户机需要额外的复杂、昂贵的同步机制，例如使用一个分布式的锁管理器。在我们的工作中，这样的文件通常用于多个生产者/单一消费者的队列系统，或者是合并了来自多个客户机的数据的结果文件。 

   记录追加是一种修改操作，它也遵循3.1节描述的控制流程，除了在主Chunk有些额外的控制逻辑。客户机把数据推送给文件最后一个Chunk的所有副本，之后发送请求给主Chunk。主Chunk会检查这次记录追加操作是否会使Chunk超过最大尺寸(64MB)。如果超过了最大尺寸，主Chunk首先将当前Chunk填充到最大尺寸，之后通知所有二级副本做同样的操作，然后回复客户机要求其对下一个Chunk重新进行记 录追加操作。(记录追加的数据大小严格控制在Chunk最大尺寸的1/4，这样即使在最坏情况下，数据碎片的数量仍然在可控的范围。)通常情况下追加的记录不超过Chunk的最大尺寸，主Chunk把数据追加到自己的副本内，然后通知二级副本把数据写在跟主Chunk一样的位置上，最后回复客户机操作成功。 

   如果记录追加操作在任何一个副本上失败了，客户端就需要重新进行操作。重新进行记录追加的结果是， 同一个Chunk的不同副本可能包含不同的数据–重复包含一个记录全部或者部分的数据。GFS并不保证 Chunk的所有副本在字节级别是完全一致的。它只保证数据作为一个整体原子的被至少写入一次。这个特性可以通过简单观察推导出来:如果操作成功执行，数据一定已经写入到Chunk的所有副本的相同偏移位 置上。这之后，所有的副本至少都到了记录尾部的长度，任何后续的记录都会追加到更大的偏移地址，或者是不同的Chunk上，即使其它的Chunk副本被Master节点选为了主Chunk。就我们的一致性保障模型而言，记录追加操作成功写入数据的region是已定义的(因此也是一致的)，反之则是不一致的(因此也就是未定义的)。正如我们在2.7.2节讨论的，我们的程序可以处理不一致的区域。



### Snapshot

  快照操作几乎可以瞬间完成对一个文件或者目录树(“源”)做一个拷贝，并且几乎不会对正在进行的其它操作造成任何干扰。我们的用户可以使用快照迅速的创建一个巨大的数据集的分支拷贝(而且经常是递归的拷贝)，或者是在做实验性的数据操作之前，使用快照操作备份当前状态，这样之后就可以轻松的提交或者回滚到备份时的状态。 

  就像AFS(alex注:AFS，即Andrew File System，一种分布式文件系统)，我们用标准的copy-on- write技术实现快照。当Master节点收到一个快照请求，它首先取消作快照的文件的所有Chunk的租约。 这个措施保证了后续对这些Chunk的写操作都必须与Master交互以找到租约持有者。这就给Master 节点一个率先创建Chunk的新拷贝的机会。 

​         租约取消或者过期之后，Master节点把这个操作以日志的方式记录到硬盘上。然后，Master节点通过复制源文件或者目录的元数据的方式，把这条日志记录的变化反映到保存在内存的状态中。新创建的快照文件和源文件指向完全相同的Chunk地址。 

​         在快照操作之后，当客户机第一次想写入数据到Chunk C，它首先会发送一个请求到Master节点查询当前的租约持有者。Master节点注意到Chunke C的引用计数超过了1(alex注:不太明白为什么会大于1.难道 是Snapshot没有释放引用计数?)。Master节点不会马上回复客户机的请求，而是选择一个新的Chunk 句柄C`。之后，Master节点要求每个拥有Chunk C当前副本的Chunk服务器创建一个叫做C`的新 Chunk。通过在源Chunk所在Chunk服务器上创建新的Chunk，我们确保数据在本地而不是通过网络复制(我们的硬盘比我们的100Mb以太网大约快3倍)。从这点来讲，请求的处理方式和任何其它Chunk没 什么不同:Master节点确保新Chunk C`的一个副本拥有租约，之后回复客户机，客户机得到回复后就可 以正常的写这个Chunk，而不必理会它是从一个已存在的Chunk克隆出来的。 





## Master Operation

 Master节点执行所有的名称空间操作。此外，它还管理着整个系统里所有Chunk的副本:它决定Chunk 的存储位置，创建新Chunk和它的副本，协调各种各样的系统活动以保证Chunk被完全复制，在所有的 Chunk服务器之间的进行负载均衡，回收不再使用的存储空间。本节我们讨论上述的主题。 

### 4.1命名空间管理与锁定

   Master节点的很多操作会花费很长的时间:比如，快照操作必须取消Chunk服务器上快照所涉及的所有的 Chunk的租约。我们不希望在这些操作的运行时，延缓了其它的Master节点的操作。因此，我们允许多个 操作同时进行，使用名称空间的region上的锁来保证执行的正确顺序。 



   不同于许多传统文件系统，GFS没有针对每个目录实现能够列出目录下所有文件的数据结构。GFS也不支持文件或者目录的链接(即Unix术语中的硬链接或者符号链接)。在逻辑上，GFS的名称空间就是一个全 路径和元数据映射关系的查找表。利用前缀压缩，这个表可以高效的存储在内存中。在存储名称空间的树 型结构上，每个节点(绝对路径的文件名或绝对路径的目录名)都有一个关联的读写锁。 

   每个Master节点的操作在开始之前都要获得一系列的锁。通常情况下，如果一个操作涉及/d1/d2/.../dn /leaf，那么操作首先要获得目录/d1，/d1/d2，...，/d1/d2/.../dn的读锁，以及/d1/d2/.../dn/leaf的读 写锁。注意，根据操作的不同，leaf可以是一个文件，也可以是一个目录。    

现在，我们演示一下在/home/user被快照到/save/user的时候，锁机制如何防止创建文件/home/user /foo。快照操作获取/home和/save的读取锁，以及/home/user和/save/user的写入锁。文件创建操作获 得/home和/home/user的读取锁，以及/home/user/foo的写入锁。这两个操作要顺序执行，因为它们试 图获取的/home/user的锁是相互冲突。文件创建操作不需要获取父目录的写入锁，因为这里没有”目 录”，或者类似inode等用来禁止修改的数据结构。文件名的读取锁足以防止父目录被删除。 

​     采用这种锁方案的优点是支持对同一目录的并行操作。比如，可以再同一个目录下同时创建多个文件:每 一个操作都获取一个目录名的上的读取锁和文件名上的写入锁。目录名的读取锁足以的防止目录被删除、 改名以及被快照。文件名的写入锁序列化文件创建操作，确保不会多次创建同名的文件。 

   因为名称空间可能有很多节点，读写锁采用惰性分配策略，在不再使用的时候立刻被删除。同样，锁的获 取也要依据一个全局一致的顺序来避免死锁:首先按名称空间的层次排序，在同一个层次内按字典顺序排序。 

### 4.2    副本的位置 

   GFS集群是高度分布的多层布局结构，而不是平面结构。典型的拓扑结构是有数百个Chunk服务器安装在许多机架上。Chunk服务器被来自同一或者不同机架上的数百个客户机轮流访问。不同机架上的两台机器 间的通讯可能跨越一个或多个网络交换机。另外，机架的出入带宽可能比机架内所有机器加和在一起的带 宽要小。多层分布架构对数据的灵活性、可靠性以及可用性方面提出特有的挑战。 

​    Chunk副本位置选择的策略服务两大目标:最大化数据可靠性和可用性，最大化网络带宽利用率。为了实 现这两个目的，仅仅是在多台机器上分别存储这些副本是不够的，这只能预防硬盘损坏或者机器失效带来 的影响，以及最大化每台机器的网络带宽利用率。我们必须在多个机架间分布储存Chunk的副本。这保证 Chunk的一些副本在整个机架被破坏或掉线(比如，共享资源，如电源或者网络交换机造成的问题)的情 况下依然存在且保持可用状态。这还意味着在网络流量方面，尤其是针对Chunk的读操作，能够有效利用 多个机架的整合带宽。另一方面，写操作必须和多个机架上的设备进行网络通信，但是这个代价是我们愿 意付出的。 

### 4.3   创建，重新复制，重新负载均衡 

   Chunk的副本有三个用途:Chunk创建，重新复制和重新负载均衡。 

   当Master节点创建一个Chunk时，它会选择在哪里放置初始的空的副本。Master节点会考虑几个因 素。(1)我们希望在低于平均硬盘使用率的Chunk服务器上存储新的副本。这样的做法最终能够平衡 Chunk服务器之间的硬盘使用率。(2)我们希望限制在每个Chunk服务器上”最近”的Chunk创建操作的次数。虽然创建操作本身是廉价的，但是创建操作也意味着随之会有大量的写入数据的操作，因为Chunk 在Writer真正写入数据的时候才被创建，而在我们的”追加一次，读取多次”的工作模式下，Chunk一旦写入成功之后就会变为只读的了。(3)如上所述，我们希望把Chunk的副本分布在多个机架之间。 

   当Chunk的有效副本数量少于用户指定的复制因数的时候，Master节点会重新复制它。这可能是由几个原 因引起的:一个Chunk服务器不可用了，Chunk服务器报告它所存储的一个副本损坏了，Chunk服务器 的一个磁盘因为错误不可用了，或者Chunk副本的复制因数提高了。每个需要被重新复制的Chunk都会根 据几个因素进行排序。一个因素是Chunk现有副本数量和复制因数相差多少。例如，丢失两个副本的 Chunk比丢失一个副本的Chunk有更高的优先级。另外，我们优先重新复制活跃(live)文件的Chunk而 不是最近刚被删除的文件的Chunk(查看4.4节)。最后，为了最小化失效的Chunk对正在运行的应用程序的影响，我们提高会阻塞客户机程序处理流程的Chunk的优先级。 

​        Master节点选择优先级最高的Chunk，然后命令某个Chunk服务器直接从可用的副本”克隆”一个副本出 来。选择新副本的位置的策略和创建时类似:平衡硬盘使用率、限制同一台Chunk服务器上的正在进行的 克隆操作的数量、在机架间分布副本。为了防止克隆产生的网络流量大大超过客户机的流量，Master节点 对整个集群和每个Chunk服务器上的同时进行的克隆操作的数量都进行了限制。另外，Chunk服务器通过 调节它对源Chunk服务器读请求的频率来限制它用于克隆操作的带宽。 

​        最后，Master服务器周期性地对副本进行重新负载均衡:它检查当前的副本分布情况，然后移动副本以便 更好的利用硬盘空间、更有效的进行负载均衡。而且在这个过程中，Master服务器逐渐的填满一个新的 Chunk服务器，而不是在短时间内用新的Chunk填满它，以至于过载。新副本的存储位置选择策略和上面 讨论的相同。另外，Master节点必须选择哪个副本要被移走。通常情况，Master节点移走那些剩余空间 低于平均值的Chunk服务器上的副本，从而平衡系统整体的硬盘使用率。

### 4.4垃圾收集

   GFS在文件删除后不会立刻回收可用的物理空间。GFS空间回收采用惰性的策略，只在文件和Chunk级的常规垃圾收集时进行。我们发现这个方法使系统更简单、更可靠。 

### 4.4.1机制

   当一个文件被应用程序删除时，Master节点象对待其它修改操作一样，立刻把删除操作以日志的方式记录 下来。但是，Master节点并不马上回收资源，而是把文件名改为一个包含删除时间戳的、隐藏的名字。当 Master节点对文件系统命名空间做常规扫描的时候，它会删除所有三天前的隐藏文件(这个时间间隔是可 以设置的)。直到文件被真正删除，它们仍旧可以用新的特殊的名字读取，也可以通过把隐藏文件改名为 正常显示的文件名的方式“反删除”。当隐藏文件被从名称空间中删除，Master服务器内存中保存的这个文件的相关元数据才会被删除。这也有效的切断了文件和它包含的所有Chunk的连接(alex注:原文是This effectively severs its links to all its chunks)。 

   在对Chunk名字空间做类似的常规扫描时，Master节点找到孤儿Chunk(不被任何文件包含的Chunk) 并删除它们的元数据。Chunk服务器在和Master节点交互的心跳信息中，报告它拥有的Chunk子集的信 息，Master节点回复Chunk服务器哪些Chunk在Master节点保存的元数据中已经不存在了。Chunk服务 器可以任意删除这些Chunk的副本。 

### 4.4.2讨论

   虽然分布式垃圾回收在编程语言领域是一个需要复杂的方案才能解决的难题，但是在GFS系统中是非常简单的。我们可以轻易的得到Chunk的所有引用:它们都只存储在Master服务器上的文件到块的映射表中。 我们也可以很轻易的得到所有Chunk的副本:它们都以Linux文件的形式存储在Chunk服务器的指定目录下。所有Master节点不能识别的副本都是”垃圾”。 

   垃圾回收在空间回收方面相比直接删除有几个优势。首先，对于组件失效是常态的大规模分布式系统，垃圾回收方式简单可靠。Chunk可能在某些Chunk服务器创建成功，某些Chunk服务器上创建失败，失败的副本处于无法被Master节点识别的状态。副本删除消息可能丢失，Master节点必须重新发送失败的删除消息，包括自身的和Chunk服务器的(alex注:自身的指删除metadata的消息)。垃圾回收提供了一致的、可靠的清除无用副本的方法。第二，垃圾回收把存储空间的回收操作合并到Master节点规律性的后台活动中，比如，例行扫描和与Chunk服务器握手等。因此，操作被批量的执行，开销会被分散。另外， 垃圾回收在Master节点相对空闲的时候完成。这样Master节点就可以给那些需要快速反应的客户机请求 提供更快捷的响应。第三，延缓存储空间回收为意外的、不可逆转的删除操作提供了安全保障。 

   根据我们的使用经验，延迟回收空间的主要问题是，延迟回收会阻碍用户调优存储空间的使用，特别是当存储空间比较紧缺的时候。当应用程序重复创建和删除临时文件时，释放的存储空间不能马上重用。我们通过显式的再次删除一个已经被删除的文件的方式加速空间回收的速度。我们允许用户为命名空间的不同部分设定不同的复制和回收策略。例如，用户可以指定某些目录树下面的文件不做复制，删除的文件被即时的、不可恢复的从文件系统移除。 

## 4.5   过期失效的副本检测 

   当Chunk服务器失效时，Chunk的副本有可能因错失了一些修改操作而过期失效。Master节点保存了每个Chunk的版本号，用来区分当前的副本和过期副本。 

   无论何时，只要Master节点和Chunk签订一个新的租约，它就增加Chunk的版本号，然后通知最新的副本。Master节点和这些副本都把新的版本号记录在它们持久化存储的状态信息中。这个动作发生在任何客 户机得到通知以前，因此也是对这个Chunk开始写之前。如果某个副本所在的Chunk服务器正好处于失效 状态，那么副本的版本号就不会被增加。Master节点在这个Chunk服务器重新启动，并且向Master节点 报告它拥有的Chunk的集合以及相应的版本号的时候，就会检测出它包含过期的Chunk。如果Master节 点看到一个比它记录的版本号更高的版本号，Master节点会认为它和Chunk服务器签订租约的操作失败 了，因此会选择更高的版本号作为当前的版本号。

   Master节点在例行的垃圾回收过程中移除所有的过期失效副本。在此之前，Master节点在回复客户机的 Chunk信息请求的时候，简单的认为那些过期的块根本就不存在。另外一重保障措施是，Master节点在通 知客户机哪个Chunk服务器持有租约、或者指示Chunk服务器从哪个Chunk服务器进行克隆时，消息中 都附带了Chunk的版本号。客户机或者Chunk服务器在执行操作时都会验证版本号以确保总是访问当前版 本的数据。



