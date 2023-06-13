# BigTable 论文阅读笔记



## 什么是 BigTable

> **A Distributed Storage System for Structured Data**

根据论文标题可知：目标数据为结构化数据，底层实现为分布式，且本身为一个存储系统

> Many projects at Google store data in Bigtable,including web indexing, Google Earth, and Google Finance.

对于 Google 内部应用十分广泛，但是不同的应用会存在不同的需求：

- 数据格式不同

- 吞吐量和延迟不同

- 集群规模不同

  

因此可以得知 BigTable 在具体实现上可以做到对上述侧重点的兼容性

> Bigtable has achieved several goals: wide applicability, scalability, high performance, and high availability. 

主要实现了以下特点：

- 广泛的实用性 >>> 数据格式不同
- 高性能 && 高可用 >>> 吞吐量、低延迟
- 可扩展性 >>> 集群规模不同



相对于 Parallel Databases 和 Main-Memmory Databases，实现上虽然借鉴其可扩展性与高性能的特点，但又提供与其不同的接口实现 

> Bigtable does not support a full relational data model

有过对关系型数据库的背景，会明白这是最大的不同点，关系型数据库本身数据的格式信息是可提前预知的，使得数据易于理解和管理，有助于保持数据的一致性和完整性。而 BigTable 为以下不同的表述：

- 不支持完整的关系数据模型，只为用户提供简单的数据模型
- 该模型支持动态控制数据布局和格式，并允许客户端推理底层存储中所表示的数据的局部性属性
- 数据使用可以是任意字符串的行和列名称进行索引
- 将数据视为未解释的字符串，尽管客户端未必真的就是用字符串格式进行存储，比如客户端经常将各种 structured and semi-structured（结构化和半结构化，它们不是纯文本）的序列化到 BigTable 的字符串中
- 允许客户端动态控制 BitTable 的数据来源：内存 or 磁盘



## Data Model

> **A Bigtable is a sparse, distributed, persistent multidimensional sorted map.**

BigTable 在名称上具有一定的迷惑性，（直译：大表），第一时间想到关系型数据库中的大表（数据量很大的表）如：

<img src="https://spongecaptain.cool/images/img_paper/filter-form-Controls-filtable.jpg" style="float: left; zoom: 50%;" alt="filter-form-Controls-filtable" />

但实际上是一个稀疏、分布式、持久化存储的多维有序映射表：

- Sparse: 稀疏的，RowKey + ColKey 的形式存储，如：一个 Person 表，含有 Id、Name、Phone、Address、Email、City 的列信息，但是每条数据并不一定含有所有的列信息

  | Row  | Columns                   |
  | ---- | :------------------------ |
  | Row1 | {ID, Name, Phone}         |
  | Row2 | {ID, Name, Address, City} |
  | Row3 | {ID, Address, Email}      |

- Distributed: 数据存储为分布式

- Persistent: 数据持久化存储

- Multidimensional Sorted Map: 这个 **Map 按照 Row Key 进行排序**，这个 Key 是一个由 `{Row Key, Column Key, Timestamp}` 组成的多维结构

即 BigTable 的每一个键值对的 Key 为 **rowKey + colKey + timestamp**，Value 则为一个字符串（未解释字节数组）：

```go
(row:string, column:string,time:int64)` -> `string
```

> 未解释的字节数组是指在 Bigtable 中存储的数据值，它们没有特定的格式或结构，而是被视为一系列字节。这些字节可以包含任何类型的数据，例如文本、数字、图像、音频等等。在 Bigtable 中，客户端通常将各种形式的结构化和半结构化数据序列化为这些未解释的字节数组，以便存储和检索。由于这些字节数组没有特定的格式或结构，因此客户端需要自己负责解析和处理这些数据



**这里重点结合论文中给的示例说一下什么是 Multidimensional Sorted Map：**

![image-20200724221115277](https://spongecaptain.cool/images/img_paper/image-20200724221115277.png)



先了解一下专有名词：尤其是**列簇**概念

- **Tablet**

  在 Bigtable 中，Row Key 相同的数据可以有非常多，为此 Bigtable 中表的行区间需要动态划分（也就是横向进行数据分区，横向的意思便是将表横着切），每个行区间称为一个 Tablet（子表）。**Tablet 是 Bigtable 数据分布和负载均衡的基本单位**，不同的子表可以有不同的大小。为了限制 Tablet 的移动成本与恢复成本，每个子表默认的最大尺寸为 200 MB。Tablet 是一个连续的 Row Key 区间，当 Tablet 的数据量增长到一定大小后可以自动分裂为两个 Tablet。同时 Bigtable 也支持多个连续的 Tablet 合并为一个大的 Tablet。

- **Column Key 与 Column Family**

  Column Key 一般都表示一种数据类型，Column Key 的集合称作 Column Family(列族)。**存储在同一 Column Family 下的数据属于同一种类型，Column Family 下的数据被压缩在一起保存**。

  **Column Family 是 access control（访问控制）、disk and memory accounting（磁盘和内存计算）的基本单元**。数据在被存储之前必须先确定其 Column Family，然后才能确定具体的 Column Key，并且表中的 Column Family 不宜过多，通常几百个。但 Column key 的个数并不进行限制，可以有无限多个。在 Bigtable 中列关键字的命名语法为：`family:qualifier` 即 `"列族:限定词"`，**列族名称必须是可打印的字符串，限定词则可以是任意字符串**。如 Webtable 中名为 anchor 的列族，该列族的每一个列关键字代表一个锚链接；anchor 列族的限定词是引用网页的站点名，每列的数据项是链接文本。

- **TimeStamp**

  **Bigtable 中的表项可以包含同一数据的不同版本**，采用时间戳进行索引。时间戳是 64 位整型，既可以由系统赋值也可由用户指定。时间戳通常以 us（微秒）为单位。时间戳既可以由 Bigtable 进行分配，也可以由客户端进行分配，如果应用程序希望避免冲突，应当生产唯一的时间戳。

  **表项的不同版本按照时间戳倒序排列（大的在前，时间戳越大表明数据加入的时间越晚）**，即最新的数据排在最前面，因而每次查询会先读到最新版本。**为了简化多版本数据的管理，每个列族都有两个设置参数用于版本的自动回收**，用户可以指定保存最近 N 个版本，或保留足够新的版本(如最近 7 天的内容)

按照表的概念举例一些数据：

![](/Users/charlotte/go/src/db_paper_doc/bigtable/image/image-bigtable-demo.png)

anchor、contents、people 为三个列簇，我们可以根据 `rowkey + colkey + ts` 定位到一个具体的 value，如：`com.cnn.www + anchor:cnnsi.com + t9` 定位到 `CNN` 

上述数据转化为一个 Map<K,V> 数据是怎样的呢？规则如下：`rowkey + col family + qualifier + type + timestamp`

```go
{"com.cnn.www","anchor","cnnsi.com","put","t9"} -> "CNN"

同理，其他的KV还有：

{"com.cnn.www","anchor","my.look.ca","put","t8"} -> "CNN.com"

{"com.cnn.www","contents","html","put","t7"} -> "<html>..."

{"com.cnn.www","contents","html","put","t6"} -> "<html>..."

{"com.cnn.www","contents","html","put","t5"} -> "<html>..."

{"com.example.www","people","author","put","t5"} -> "John Doe"
```

到此再来审视一下 BigTable 的描述词：

- 多维：key是一个复合数据结构，由多维元素构成，包括 rowkey、column family、qualifier、type 以及 timestamp
- 稀疏：从上图逻辑表中行 "com.example.www" 可以看出，整整一行仅有一列（people：author）有值，其他列都为空值。在其他数据库中，对于空值的处理一般都会填充 null，而对于 BigTable，空值不需要任何填充。这个特性为什么重要？因为列在理论上是允许无限扩展的，对于成百万列的表来说，通常都会存在大量的空值，如果使用填充null的策略，势必会造成大量空间的浪费。因此稀疏性是列可以无限扩展的一个重要条件
- 排序：构成数据的 KV 在同一个文件中都是有序的，但规则并不是仅仅按照 rowkey 排序，而是按照 KV 中的 key 进行排序——先比较 rowkey，rowkey 小的排在前面；如果 rowkey 相同，再比较 column，即 column family：qualifier，column 小的排在前面；如果column 还相同，再比较时间戳 timestamp，即版本信息，timestamp 大的排在前面。这样的多维元素排序规则对于提升 BigTable 的读取性能至关重要
- 分布式：很容易理解，构成 BigTable 的所有 Map 并不集中在某台机器上，而是分布在整个集群中



## Building Blocks

> 1. Bigtable uses the distributed Google File System (GFS) to store log and data fifiles. 
> 2. A Bigtable cluster typically operates in a shared pool of machines that run a wide variety of other distributed applications,
>
> and Bigtable processes often share the same machines with processes from other applications. 
>
> 3. Bigtable depends on a cluster management system for scheduling jobs, managing resources on shared machines, dealing
>
> with machine failures, and monitoring machine status.

主要依赖如下：

- 依赖 SSTable 来存储数据单元
- 依赖 WorkQueue 负责故障处理和监控
- 依赖 GFS 存储日志文件和数据文件
- 依赖 Chubby 存储元数据和进行主服务器的选择



Bigtable 主要由链接到每个客户端的程序库、主服务器和多个子表服务器组成，其架构如图 2.2 所示。为了适应工作负载的变化，可以动态地向集群中添加或删除子表服务器

Bigtable 本质是一个**进程**，Bigtable 进程通常与其他应用程序的进程共享同一台机器。Bigtable 依赖于集群管理系统来调度作业、管理共享机器上的资源、处理机器故障和监视机器状态

![image-20200725100355004](https://spongecaptain.cool/images/img_paper/image-20200725100355004.png)



### SSTable

> An SSTable provides a persistent, ordered immutable map from keys to values, where both keys and values are arbitrary byte strings. Operations are provided to look up the value associated with a specifified key, and to iterate over all key/value pairs in a specifified key range.

SSTable 为 BigTable 的基本存储单元，提供一个持久化、有序的、不可变的映射：

- 键值对都是任意字节字符串；
- 提供查找与制定关联值的操作；
- 提供迭代遍历指定值所有键值对的操作；

> Internally, each SSTable contains a sequence of blocks (typically each block is 64KB in size, but this is configurable). A block index (stored at the end of the SSTable) is used to locate blocks; the index is loaded into memory when the SSTable is opened. A lookup can be performed with a single disk seek: we first find the appropriate block by performing a binary search in the in-memory index, and then reading the appropriate block from disk. Optionally, an SSTable can be com- pletely mapped into memory, which allows us to perform lookups and scans without touching disk.

每个SSTable内部包含一系列数据块（通常每个块的大小为64KB，但这是可配置的）:

- 块索引（存储在SSTable的末尾）用于定位块，当打开SSTable时，索引会被加载到内存中；
- 可以使用单个磁盘查找来执行查找操作：通过在内存中的索引中执行二进制搜索来找到适当的块，然后从磁盘中读取适当的块；



### GFS

GFS是Google的分布式文件系统，用于在Bigtable中存储SSTable和其他元数据。GFS提供高可靠性、高可扩展性和高性能的存储服务。



### Chubby

**Bigtable 依赖于 Chubby 提供的锁服务**，如果 Chubby 长时间不能访问，Bigtable 会无法使用。Bigtable 依赖 Chubby 完成以下任务：

- 确保任意时间至多存在一个活跃的主服务器副本；

  > 即 At most one active master at any time.

- 存储 Bigtable 中数据的 bootstrap location（引导位置）；

- 发现 Tablet（子表）服务器，并在子表服务器失效时进行善后；

- 存储 Bigtable 的 schema 信息，即表的 column family 信息；

- 存储 access control lists（访问控制列表）；

**注意**：如果集群内的 Chubby 在长时间内不可用（比如宕机或者网络问题），那么整个 Bigtable 系统也将会不可用。但是如果系统内仅仅是部分 Chubby 不可用，那么事实上只会导致 Bigtable 的部分数据不可用。



## Implementation

主要有三部分：

> a library that is linked into every client, one master server, and many tablet server

客户端、Master 主服务器、Tablet 子表服务器



### Master Server

> The master is responsible for assigning tablets to tablet servers, detecting the addition and expiration of tablet servers, balancing tablet-server load, and garbage collection of fifiles in GFS. In addition, it handles schema changes such as table and column family creations

主要用于为子表服务器分配子表、检测子表服务器的加入或过期、 进行子表服务器的负载均衡和对保存在 GFS 上的文件进行垃圾收集。主服务器持有活跃的子表服务器信息、子表的分配信息和未分配子表的信息。如果子表未分配，主服务器会将该子表分配给空间足够的子表服务器。此外在表和列簇创建的时候处理 Schema 变更。



### Tablet Server

> Each tablet server manages a set of tablets (typically we have somewhere between ten to a thousand tablets per tablet server). The tablet server handles read and write requests to the tablets that it has loaded, and also splits tablets that have grown too large.

**每个子表服务器管理一组子表（a set of tablets）**，负责其磁盘上的子表的读写请求，并在子表过大时进行子表的分割。与许多单一主节点的分布式存储系统一样，读写数据时，客户端直接和子表服务器通信，因此在实际应用中，主服务器的负载较轻。



### Client

客户端使用客户端程序库访问 Bigtable，客户端库会缓存子表的位置信息。当客户端访问 Bigtable 时，首先要调用程序库中的 `Open()` 函数获取文件目录，文件目录可能在缓存中，也可能通过与主服务器进行通信得到。最后再与子表服务器通信



### Metadata Information(tablet location information)

#### Tablet location

> We use a three-level hierarchy analogous to that of a ***B+tree***  to store **tablet location information**.
>
> The fifirst level is a fifile stored in Chubby that contains the location of the ***root tablet***,  and it is never
>
> split to ensure that the tablet location hierarchy has no more than three levels.

<img src="https://spongecaptain.cool/images/img_paper/image-20200725103531486.png" alt="image-20200725103531486" style="zoom:67%;" />

可以看到 Bigtable 将子表按照三层关系进行组织

第一层，存储在 Chubby file中，里面包含了Root tablet的位置信息。

第二层，根子表 (Root tablet) ，元数据子表 (METADATA tablets) 中的第一个子表，它存储着元数据表里其他子表的位置信息，根子表随着大小的增长是不会被分割的。

第三层，原数据子表 (METADATA tablets) ，保存其他用户数据表的子表信息。

在查找子表的时候，客户机首先会检查自己的库，看是否已经有这个子表位置的缓存，如果存在这个缓存且这个缓存还有效，就会按照这个位置去获取子表信息。如果这个子表的缓存信息错误，那么客户机将会递归向上一层的子表服务器进行查询。若缓存为空，则客户机将会从 Chubby file 开始获取根子表的位置，查询根子表查寻相应元数据子表的位置，再在元数据字表中找到需要的数据子表的位置，完成完整的一次询问的查找。

基于 GFS 存储系统的 Bigtable 的存储逻辑则如下图所示：

[<img src="https://spongecaptain.cool/images/img_paper/image-20200803220617756.png" alt="image-20200803220617756" style="zoom: 25%;" />](https://spongecaptain.cool/images/img_paper/image-20200803220617756.png)

> 详细花了 Row key_1 的 GFS 存储逻辑，其他 Row key 有着完全相同的存储逻辑。

特点是：

- 拥有相同 row key 的键值对分为多个 Tablet 进行分布式存储（每一个 Tablet 默认大小为 200 MB），如果字节大小不足以填满 200 MB，那么也需要占用一个 Tablet 大小（这种情况不常见）；
- Tablet 是 Bigtable 中数据分布和负载均衡的最基本单位，这个性质对于 GFS 系统来说，就是 GFS 会为每一个 Tablet 默认提供 3 个副本，每一个副本尽量存储在不同机架上的不同主机的磁盘上；





#### Tablet Assignment

前文中有提到主服务器 master 主要负责监控子表服务器状态以及子表的分配。主服务器需要通过 Chubby 确认每个子表服务器是否还在正常工作，跟踪子表都分配给了哪些子表服务器以及哪些子表还没有被分配。

当一个子表服务器启动的时候，它会在 Chubby 特定的目录下建立一个自己的文件并获得互斥锁，主服务器通过文件来监控存在哪些子表服务器，通过周期性尝试获取这些文件的互斥锁来确认这些子表服务器是否还在正常工作。当子表服务器失去与 Chubby 的连接后，就会失去这个互斥锁。但是只要子表服务器上的数据还存在并且 Chubby 相应文件还存在，它还会不断试图请求会这个互斥锁。一旦主服务器获得了这个互斥锁，它就会删除这个文件，导致子表服务器的最终停止。而子表服务器主动停止服务的时候，也会释放这个互斥锁，以便主服务器更快意识到这个子表服务器的退出。

当主服务器和 Chubby 连接被断开后，当前的主服务器会主动关闭自己，这时候系统就会重新选择一个新的主服务器出来。

1. 主服务器启动时，会首先获取一个 Chubby 上的 master 锁以防止其他服务器同时成为主服务器；
2. 然后新主服务器会扫描 Chubby 的特定目录尝试获取互斥锁来获得当前正在工作的子表服务器；
3. 之后再询问每个子表服务器被分配的子表；
4. 最后再统计 METADATA 子表中还未被分配的子表，准备将其分配 (若元数据子表还未被分配，则需先将根子表加入到待分配的子表集合中) 。



#### Tablet Serving(read and write)





## 客户端的数据读取流程

之前已经提到了 Bigtable 的一大特点便是提出一种不同于传统关系型数据库的模型，即更为灵活的 Key-Value 数据存储模型，对外暴露一个逻辑上的多维表：

```
(row:string, column:string, time:int64) → string
```

因此当客户端读取数据时，在内部有如下的执行流程：

- 首先，确定 Row Key；
- 其次，根据 Row Key 查找特定的 Column Key；
- 最后，根据 colomn 以及 version 确定具体读取的内容；

> 这是一个多维表查询的典型过程，这个过程类似于磁盘的直接读取，先确定分区，在顺序读写。

客户端定位子表服务器时：

- 首先，需要访问 Chubby 以获取根子表地址，然后浏览元数据表定位用户数据；
- 然后，子表服务器会从 GFS 中获取数据，并将结果返回给客户端。

Bigtable 数据访问结构如下图所示。如果客户端不知道子表的地址或缓存的地址信息不正确，客户端会递归查询子表的位置。若客户端缓存为空，寻址算法需要三次网络往返通信；如果缓存过期，寻址算法需要六次网络往返通信才能更新数据。地址信息存储在内存中，因而不必访问 GFS，但仍会预取子表地址来进一步减少访问开销。元数据表中还存储了一些次要信息，如子表的事件日志，用于程序调试和性能分析。

[![image-20200725103855414](https://spongecaptain.cool/images/img_paper/image-20200725103855414.png)](https://spongecaptain.cool/images/img_paper/image-20200725103855414.png)

通常而言，为了加快数据访问以及数据的分块存储管理，存储系统通常会提供各种排序逻辑，在 Bigtable 中的排序逻辑主要有三种：

- 利用 Row Key 进行排序，目的是横向化划分为多个 Tablet，避免形成超大块的数据，便于数据管理；
- 利用 Column key 以及 Column family 进行排序，目的是加快检索时的速度；
- 利用 timestamp 的天然时间线排序，目的是提供多版本控制以及过期数据的自动回收；











