# Linux文件系统

## **1 概述**

​      Linux继承了UNIX**一切皆文件**的设计哲学，用**文件**和**树形目录**的抽象逻辑概念代替了硬盘和光盘等物理设备使用数据块的概念，用户使用文件系统来保存数据时不必关心数据实际保存在硬盘（或者光盘）的地址为多少的数据块上，只需要记住这个文件的所属目录和文件名。但对于程序员来说，了解文件系统的底层组织方式，是进行Linux系统编程所必备的。接下来的讨论主要聚焦于磁盘文件系统。

## **2 索引式文件系统与日志式文件系统**

## 2.1 索引式文件系统

- **核心数据结构说明**

​       Linux中文件和目录保存在称为块设备的磁盘或磁带上，通常每个磁盘上可以定义一个或多个文件系统，Linux文件系统的运行离不开三个重要的数据结构：superblock，inode，block，当然还包含引导块，存放引导程序，用来读入和启动操作系统。 

**superblock**：记录文件系统的整体信息，包含inode/block的大小、总量、使用量、剩余量，以及文件系统的格式，文件系统挂载时间，最近一次数据写入时间，最近一次校验磁盘的时间等。
**inode**：记录文件的属性，一个文件占用一个inode，inode大小(ext2中)大约为128B，并记录文件数据所在的block号码，具体来说包含的信息如下：

- 文件的字节数
- 文件拥有者的User ID
- 文件的Group ID
- 文件的读、写、执行权限
- 文件的时间戳，共有三个：ctime表示inode上一次变动的时间，mtime表示文件内容上一次变动的时间，atime表示文件上一次打开的时间
- 链接数：指有多少文件名指向这个inode
- 文件数据的block号码
  在文件的block数量很大时，通常会采用多级block来记录block号码，这里采用**Bitmap**标记未使用的inode号码。

**block**：实际记录文件的内容，若文件太大，则会占用多个block，通常的block大小有1K，2K，4K三种，这里内核记录block信息的数据结构是Bitmap。
使用dumpe2fs命令可以查询某块设备上superblock和blockgroup的详细信息，这里不再详细给出。
此外，这里再澄清两个概念：

**Hard Link**：硬链接，硬链接文件和原始文件对应同一个inode号码，增加硬链接文件一般不改变磁盘的空间与inode数目，通常硬链接文件不能跨越文件系统建立，并且不能生成目录的链接文件。

**Symbolic Link**：符号链接，创建一个新的文件，读取该文件时会让数据读取指向它link的文件的文件名，和原始文件的inode号码不同。
如下图，passwd-hd和passwd-so分别是passwd的硬链接和软链接，可见前者和passwd的inode号码一致，而后者不一致。

```
marsdeMacBook-Pro:tmp mars$ ln tmp.sql tmp.sql-hd
marsdeMacBook-Pro:tmp mars$ ln -s tmp.sql tmp.sql-so
marsdeMacBook-Pro:tmp mars$ ll
total 27208
-rw-r--r--   2 mars  staff   545144  5  7 13:14 tmp.sql
-rw-r--r--   2 mars  staff   545144  5  7 13:14 tmp.sql-hd
lrwxr-xr-x   1 mars  staff        7  6 20 10:54 tmp.sql-so -> tmp.sql
```

- **文件读取流程介绍**

  文件系统需要链接到目录树才能被我们使用，也就是所谓的挂载，挂载点一定是目录，该目录是文件系统的入口。假设我们想要读取文件/etc/paswd的内容，那么一般需要从根目录的inode内容开始往下读，直到找到正确的文件名，具体流程在我的mac上如下：

<img src="https://pic4.zhimg.com/v2-e9f469e1d9fbf41f9fdf1144ea8a20d5_b.jpg" data-rawwidth="634" data-rawheight="68" class="origin_image zh-lightbox-thumb" width="634" data-original="https://pic4.zhimg.com/v2-e9f469e1d9fbf41f9fdf1144ea8a20d5_r.jpg">![img](https://pic4.zhimg.com/80/v2-e9f469e1d9fbf41f9fdf1144ea8a20d5_hd.jpg)

1. 寻找/的inode：通过挂载点信息找到inode号码为2的inode，且其权限可以让我们读取block的内容；
2. /的block：根据block的内容找到含有etc/目录的inode号码261633；
3. etc/的inode：根据261633号inode内容中的权限值，知道可以读取etc/的block内容；
4. etc/的block：根据block号码找到含有passwd文件的inode号码261846；
5. passwd的inode：根据261846号inode内容中的权限值，知道可以读取passwd的block内容；
6. passwd的block：读取block中的内容至内存缓冲区。

#### 2.2 日志式文件系统

上述提到的文件系统称为**索引式文件系统**，Linux内核早期支持中的**ext2**文件系统正是这种类型。这是针对新增一个文件时，需要的步骤如下：

1. 确定使用者对于要增加新文件的目录是否具有w与x的权限，如有，方可新增；
2. 根据inode bitmap寻找到没有使用的inode号码，将新文件的权限/属性写入；
3. 根据block bitmap找到没有使用的block号码，将数据写入block中，更新inode的block指向数据；
4. 将刚刚写入的inode与block数据同步更新至inode bitmap和block bitmap，并更新superblock的内容。

上述的inode bitmap，block bitmap，superblock称为系统的**元数据（metadata）**。在系统故障时，会出现元数据内容与实际数据存放内容不一致的情况，当然在文件系统重新启动时，会调用文件扫描工具fsck来恢复损坏的元数据信息，但在文件系统很大时，要花费很长时间来恢复。针对上述不一致的情况，出现了日志式文件系统，日志式文件系统会专门划出一个区块记录系统写入和修改文件的步骤，此时写文件的步骤如下：

1. 准备：当系统写入一个文件时，会在日志记录区块记录文件要写入的信息；
2. 写入：写入文件的权限和数据，更新元数据的内容；
3. 完成：在数据与元数据更新完成后，在日志记录区块中完成文件的记录。

在出现故障需要恢复时，可根据日志追踪之前提交到主文件系统的更改，大大减少了磁盘的扫描时间，实现丢失数据的快速重建，比传统的索引式文件系统更安全。Linux下的集中日志式文件系统有[XFS](https://link.zhihu.com/?target=http%3A//oss.sgi.com/projects/xfs/)（目前是CentOS7的默认文件系统），[ReiserFS](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/ReiserFS)，[Ext3](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Ext3)，[Ext4](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/Ext4)。

## **3 Ext2/Ext3/Ext4的区别和比较**

**3.1 Ext2与Ext3的比较**

​     ext3和ext2的主要区别在于，ext3引入**Journal（日志）**机制，Linux内核从2.4.15开始支持ext3，它是从文件系统过渡到日志式文件系统最为简单的一种选择，ext3提供了数据完整性和可用性保证**。**

- ext2和ext3的格式完全相同，只是在ext3硬盘最后面有一部分空间用来存放Journal的记录；
- 在ext2中，写文件到硬盘中时，先将文件写入缓存中，当缓存写满时才会写入硬盘中；
- 在ext3中，写文件到硬盘中时，先将文件写入缓存中，待缓存写满时系统先通知Journal，再将文件写入硬盘，完成后再通知Journal，资料已完成写入工作；
- 在ext3中，也就是有Journal机制里，系统开机时检查Journal的内容，来查看是否有错误产生，这样就加快了开机速度；

##### 3.2 Ext3与Ext4的比较

 Linux内核从2.6.28开始支持ext4文件系统，相比于ext3提供了更佳的性能和可靠性。下面先简单罗列出二者的差异，后续文章再来深入探索。
**1. 与 Ext3 兼容**。 执行若干条命令，就能从 Ext3 在线迁移到 Ext4，而无须重新格式化磁盘或重新安装系统。原有 Ext3 数据结构照样保留，Ext4 作用于新数据，当然，整个文件系统因此也就获得了 Ext4 所支持的更大容量。 
**2. 更大的文件系统和更大的文件**。 较之 Ext3 目前所支持的最大 16TB 文件系统和最大 2TB 文件，Ext4 分别支持 1EB的文件系统，以及 最大16TB 的文件。 
**3. 无限数量的子目录**。 Ext3 目前只支持 32,000 个子目录，而 Ext4 支持无限数量的子目录。 
**4. Extents**。 Ext3 采用间接块映射，当操作大文件时，效率极其低下。比如一个 100MB 大小的文件，在 Ext3 中要建立 25,600 个数据块（每个数据块大小为 4KB）的映射表。而 Ext4 引入了现代文件系统中流行的 extents 概念，每个 extent 为一组连续的数据块，上述文件则表示为“该文件数据保存在接下来的 25,600 个数据块中”，提高了不少效率。 
**5. 多块分配**。 当写入数据到 Ext3 文件系统中时，Ext3 的数据块分配器每次只能分配一个 4KB 的块，写一个 100MB 文件就要调用 25,600 次数据块分配器，而 Ext4 的多块分配器“multiblock allocator”（mballoc） 支持一次调用分配多个数据块。 
**6. 延迟分配**。 Ext3 的数据块分配策略是尽快分配，而 Ext4 和其它现代文件操作系统的策略是尽可能地延迟分配，直到文件在 cache 中写完才开始分配数据块并写入磁盘，这样就能优化整个文件的数据块分配，与前两种特性搭配起来可以显著提升性能。 
**7. 快速 fsck**。 以前执行 fsck 第一步就会很慢，因为它要检查所有的 inode，现在 Ext4 给每个组的 inode 表中都添加了一份未使用 inode 的列表，今后 fsck Ext4 文件系统就可以跳过它们而只去检查那些在用的 inode 了。 
**8. 日志校验**。 日志是最常用的部分，也极易导致磁盘硬件故障，而从损坏的日志中恢复数据会导致更多的数据损坏。Ext4 的日志校验功能可以很方便地判断日志数据是否损坏，而且它将 Ext3 的两阶段日志机制合并成一个阶段，在增加安全性的同时提高了性能。 
**9. “无日志”（No Journaling）模式**。 日志总归有一些开销，Ext4 允许关闭日志，以便某些有特殊需求的用户可以借此提升性能。 
**10. 在线碎片整理**。 尽管延迟分配、多块分配和 extents 能有效减少文件系统碎片，但碎片还是不可避免会产生。Ext4 支持在线碎片整理，并将提供 e4defrag 工具进行个别文件或整个文件系统的碎片整理。 
**11. inode 相关特性**。 Ext4 支持更大的 inode，较之 Ext3 默认的 inode 大小 128 字节，Ext4 为了在 inode 中容纳更多的扩展属性（如纳秒时间戳或 inode 版本），默认 inode 大小为 256 字节。Ext4 还支持快速扩展属性（fast extended attributes）和 inode 保留（inodes reservation）。 
**12. 持久预分配（Persistent preallocation）**。 P2P 软件为了保证下载文件有足够的空间存放，常常会预先创建一个与所下载文件大小相同的空文件，以免未来的数小时或数天之内磁盘空间不足导致下载失败。 Ext4 在文件系统层面实现了持久预分配并提供相应的 API（libc 中的 posix_fallocate()），比应用软件自己实现更有效率。 
**13. 默认启用 barrier**。 磁盘上配有内部缓存，以便重新调整批量数据的写操作顺序，优化写入性能，因此文件系统必须在日志数据写入磁盘之后才能写 commit 记录，若 commit 记录写入在先，而日志有可能损坏，那么就会影响数据完整性。Ext4 默认启用 barrier，只有当 barrier 之前的数据全部写入磁盘，才能写 barrier 之后的数据。

## **4 小结**

​    目前，大多数Linux发行版，包括我的Ubuntu 16.04的默认支持文件系统是ext4，ext4也是首个专门为Linux设计的文件系统，我们可以轻易的从ext3迁移到ext4，对于程序员来说，了解文件系统的演化脉络是十分重要的，后续会继续深入讨论Linux下的各个文件系统。

参考资料
【1】[鸟哥的Linux私房菜第八章](https://link.zhihu.com/?target=http%3A//cn.linux.vbird.org/linux_basic/0230filesystem_1.php)
【2】[Ext2、Ext3和Ext4之间的区别](https://link.zhihu.com/?target=http%3A//misujun.blog.51cto.com/2595192/883949)
【3】[Linux 文件系统剖析](https://link.zhihu.com/?target=http%3A//www.ibm.com/developerworks/cn/linux/l-linux-filesystem/)

## 