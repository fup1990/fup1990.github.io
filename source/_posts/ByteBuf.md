---
title: ByteBuf缓冲区
date: 2018-08-20 17:55:55
categories:
- netty
tags:
- buffer
- netty
- IO
---

ByteBuf是Netty中实现的字节缓冲区。

# 工作原理

ByteBuf与Java NIO中ByteBuffer最大的不同点点在于，ByteBuffer中只维护了一个索引，因此需要调用flip()方法切换读写模式。而ByteBuf中，维护了两套索引，一个读索引，记录了已读取的位置；另一个写索引，记录了可写的位置。ByteBuf中名称以read或write开头的方法会推进相应的索引，而以set或get开头的不会。可以指定ByteBuf最大的容量，默认是Integer.MAX_VALUE。

![img](ByteBuf\buffer_init.png)

上图展示了一个刚初始化的ByteBuf，读索引和写索引的初始值为0。

![img](ByteBuf\three_part.png)

随着读、写指针的偏移，ByteBuf被分隔成了三个部分。

- 可丢弃数据：指已经读取过的数据，通过discardReadBytes()方法，可以清空已读的数据。不建议频繁使用。
- 可读取数据：指未读的数据。
- 可写入数据：指可写入的空间。