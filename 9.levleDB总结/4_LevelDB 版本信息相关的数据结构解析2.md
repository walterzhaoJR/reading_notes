# LevelDB 版本信息相关的数据结构解析2
* 上篇介绍了leveldb中涉及版本信息的一些数据结构Version、VersionSet、VersionEdit，本篇文章来介绍下版本管理是如何进行的。

* 首先我们可以回忆下leveldb读取一条key的流程：首先在memtable中读取，找到则返回；否则要去sst文件中读取。遇到第一个问题就是，我们如何可以快速的从众多的sst文件中找到具体包含我们目标key的文件。其实在leveldb中有一个版本信息记录了每个sst文件在哪一个level上，每个文件中包含的key的大小范围。那这样，我们要定位到目标key在哪个sst文件中就方便多了，这个版本就想索引一样帮助我们快速定位sst文件。
* 第二个问题：那么有一个版本就够了吗？可以看到一个版本可以表征一个状态下的sst文件情况，随着新的数据不断写入memtable，当memtable被写满了之后，就会转换为level 0层的一个sst文件，或者由level n层和level n+1层的sst合并成新的sst文件，这样就产生了一个新的版本，所以leveldb也需要去管理这些版本。也可以从另外一个角度去看这个问题：假设使用了整个数据库的迭代器，在迭代数据库时，leveldb是提供了一个一致性的视图，也就是只能看到迭代前的写入，而迭代后的写入是看不到，在实现的时候，迭代器会引用这个版本，以及里面的sst，直到迭代结束。如果在迭代过程中，存在删除数据，直接删除某个文件，那么就无法引用到这个文件了，就会出错。所以需要多个版本，有一个版本是当前版本，当新生成版本后，旧的版本如果有被其它线程引用，也需要保留，直到不被引用后，才会被删除，所以一个时刻可能有多个版本存在。
* 我来简单总结下leveldb管理版本用到的一些类：
    * Version 表示一个版本，主要记录的是这个版本包含的sst的信息
    * VersionSet 表示一个版本集合，其中保存了节点为Version的一个双链表，其中有一个Version是当前版本，因为只有一个实例，还保存了其它的一些全局的元数据，Dummy Version是链表头
    * VersionEdit保存了要对Version做的修改，在一个Version上应用一个VersionEdit，可以生成一个新的Version 可以说 Version1 + VersionEdit1 = Version2
    * Builder是一个帮助类，帮助Version上应用VersionEdit，生成新版本

* 版本管理的基本工作流程：
    * VersionSet里保存着当前版本，以及被引用的历史版本
    * 进行Compaction时，会将更改的内容写入到一个VersionEdit中，包含新增的文件，删除的文件
    * 利用Builder将VersionEdit应用到当前版本上面生成一个新的版本
    * 将新版本链接到VersionSet的双链表上面
    * 将新的版本设置为当前版本
    * 将旧的当前版本Unref，就是引用计数减1

* 版本控制中使用了引用计数来管理历史版本:
    * 假设一个没有被访问的数据库，这时候只有一个版本，也就是当前版本v1，它的引用计数是1
    * 假设有一个迭代器开始访问数据库，这时候它会对当前版本v1 Ref，引用计数变成了2
    * 其它线程不断写入，导致了Compaction的生成，这时候生成了一个新的版本v2成为了当前版本，引用计数为1，然后对v1 Unref，这时候v1的引用计数为1
    * 因为v1的引用计数为1，所以不会被删除，迭代器线程还可以继续访问v1
    * 迭代器结束后，对v1进行Unref，这时候v1的引用计数变为0，就会从链表上删除了，这时候又只剩下v2版本了
    * 版本里面的sst也是采用引用计数来管理的，每个版本引用一个sst时会Ref，删除版本时会Unref，如果一个sst的计数为0，那么这个sst就可以被删除了
* 版本变迁的写入，当需要生成一个新版本时，都会向manifest写入一条新纪录，记录了新版本相对旧版本有哪些变化，主要包括以下几部分：
    * 写入时刻的日志编号
    * 写入时刻下个可用文件的编号
    * 写入时刻上一个用掉的SequenceNumber
    * compact_pointer_数组，可能有多条记录，有level和对应的internal key
    * 记录删除了哪些文件，记录了level和对应的file number
    * 当前存在的sst文件，包含元数据，包括在哪一level，文件编号，文件大小，里面的最小键和最大键
* Builder的作用
    * 可能会问：有了VersionEdit，就可以构造出另外一个版本，为什么还需要Builder。
    * 我们知道，当有VersionEdit改变版本时，直接引用到当前的Version生成一个新的Version就可以了。
    * 正常情况下，一次compaction或者memtable写入后sst后，会产生一个VersionEdit，将这个VersionEdit应用到当前Version上生成一个新的Version。但是在版本变更时，也会将VersionEdit的内容写入manifest中。每当重新打开一个数据库时，需要读取manifest重新构造版本信息，这个版本信息由初始的Version和多个VersionEdit生成，如果直接用VersionEdit应用就会产生成多个版本，进行了很多重复的操作，无疑会降低效率
    * 所以，才引入了Builder，使用Builder，可以将多个VersionEdit的内容累积到Builder上，然后一次性应用到当前Version即可生成新的Version。