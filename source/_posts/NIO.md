---
title: NIO
date: 2018-08-20 17:34:40
categories:
- IO
tags:
- NIO
---

NIO（Non-blocking I/O，在Java领域，也称为New I/O），是一种同步非阻塞的I/O模型，也是I/O多路复用的基础，已经被越来越多地应用到大型应用服务器，成为解决高并发与大量连接、I/O处理问题的有效方式。

NIO主要有三大核心部分：**Channel(通道)**，**Buffer(缓冲区)**, **Selector(选择器)**。传统IO基于字节流和字符流进行操作，而NIO基于Channel(通道)和Buffer(缓冲区)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector(选择器)用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个线程可以监听多个数据通道。

# 常见I/O模型对比

![img](NIO\IO_model.png)

以socket.read()为例子：传统的BIO里面socket.read()，如果TCP RecvBuffer里没有数据，函数会一直阻塞，直到收到数据，返回读到的数据。对于NIO，如果TCP RecvBuffer有数据，就把数据从网卡读到内存，并且返回给用户；反之则直接返回0，永远不会阻塞。NIO一个重要的特点是：socket主要的读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的I/O操作是同步阻塞的（消耗CPU但性能非常高）。

# Channel

Channel(通道)类似流，但又有些不同。

1. 既可以从通道中读取数据，又可以写数据到通道。但流（InputStream和OutputStream）

   的读写通常是单向的。 

2. 通道可以异步地读写。 

3. 通道中的数据总是要先读到一个Buffer，或者总是要从一个Buffer中写入。 

![img](NIO\channel.png)

## Channel的实现

### FileChannel

从文件中读写数据，FileChannel无法设置为非阻塞模式，它总是运行在阻塞模式下。

```java
RandomAccessFile read = new RandomAccessFile("a.txt", "r");
RandomAccessFile write = new RandomAccessFile("b.txt", "rw");
FileChannel readChannel = read.getChannel();
FileChannel writeChannel = write.getChannel();
ByteBuffer buffer = ByteBuffer.allocate(1024);
while (readChannel.read(buffer) != -1) {
    buffer.flip();
    writeChannel.write(buffer);
    buffer.compact();
}
readChannel.close();
writeChannel.close();
```

### **DatagramChannel**

能通过UDP协议读写网络中的数据。

### **SocketChannel**

能通过TCP读写网络中的数据。

### **ServerSocketChannel**

可以监听新进来的TCP连接，对每一个新进来的连接都会创建一个SocketChannel。

# Buffer

Buffer缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存。

## 实现

![img](NIO\Buffer.png)

## 使用方法

### 1、分配内存容量

 要想获得一个Buffer对象首先要进行分配。 每一个Buffer类都有一个allocate方法。

```java
 ByteBuffer buf = ByteBuffer.allocate(48);
```

将字节数组包装成Buffer。

```java
byte[] bytes = new byte[48];
ByteBuffer buf = ByteBuffer.wrap(bytes);
```

### 2、写入数据到Buffer

当向buffer写入数据时，buffer会记录下写了多少数据。

从channel中读数据写到buffer

```java
 int bytesRead = inChannel.read(buf);
```

通过put方法写入

```java
 buf.put(127);
```

### 3、调用flip()方法

将Buffer从写模式切换到读模式。

### 4、从Buffer中读取数据

读取之前写入到buffer的所有数据。

从buffer中读数据写到channel

```java
 int bytesWritten = inChannel.write(buf);
```

通过get方法读数据

```java
 byte aByte = buf.get();
```

### 5、清空缓冲区

 一旦读完Buffer中的数据，需要让Buffer准备好再次被写入。可以通过clear()或compact()方法来完成。调用clear()方法或者compact()方法。clear()方法会清空整个缓冲区；compact()方法只会清除已经读过的数据。

### 6、缓冲区切片

将当前的position到limit之间的数据分割出来，返回一个新的ByteBuffer。新Buffer与老Buffer共享切分区间内的数据（position到limit之间的数据）。

```java
ByteBuffer byteBuffer = buf.slice();
```

切片源码

```java
    public ByteBuffer slice() {
        return new HeapByteBuffer(hb,//原始字节数组
                                  -1,//新buffer的mark标记位置设为0
                                  0, //新buffer的position位置设为0
                                  this.remaining(),//新buffer的limit等于老buffer的limit-position
                                  this.remaining(),//新buffer的capacity等于老buffer的limit-position
                                  this.position() + offset);
    }
```

## 工作原理

![img](NIO\buffer_yuanli.png)

## 关键属性

### **capacity**

缓冲区数组的总长度。 作为一个内存块，Buffer有一个固定的大小值，也叫“capacity”.你只能往里写capacity个byte、long，char等类型。一旦Buffer满了，需要将其清空（通过读数据或者清除数据）才能继续写数据往里写数据。

### **position**

下一个要操作的数据元素的位置。

当你写数据到Buffer中时，position表示当前的位置。初始的position值为0。当一个byte、long等数据写到Buffer后， position会向前移动到下一个可插入数据的Buffer单元。position最大可为capacity – 1。

当读取数据时，也是从某个特定位置读。当将Buffer从写模式切换到读模式，position会被重置为0. 当从Buffer的position处读取数据时，position向前移动到下一个可读的位置。

### **limit**

缓冲区数组中不可操作的下一个元素的位置：limit<=capacity。

在写模式下，Buffer的limit表示你最多能往Buffer里写多少数据。 写模式下，limit等于Buffer的capacity。

当切换Buffer到读模式时， limit表示你最多能读到多少数据。因此，当切换Buffer到读模式时，limit会被设置成写模式下的position值。换句话说，你能读到之前写入的所有数据（limit被设置成已写数据的数量，这个值在写模式下就是position）。

### mark

用于记录当前position的前一个位置或者默认是0。

通过调用Buffer.mark()方法，可以标记Buffer中的一个特定的position，之后可以通过调用Buffer.reset()方法恢复到这个position。Buffer.rewind()方法将position设回0，所以你可以重读Buffer中的所有数据。limit保持不变，仍然表示能从Buffer中读取多少个元素。

## 原理说明

![img](NIO\allocate.png)

1、通过ByteBuffer.allocate(11)方法创建了一个11个byte的数组的缓冲区，初始状态如下图，position的位置为0，capacity和limit默认都是数组长度11。

![img](NIO\write.png)

2、从Channel读取5个字节的数据，并写入Buffer时，此时position的位置为5，capacity和limit不变。

![img](NIO\flip.png)

3、当调用flip()方法后，将Buffer转换为读模式，读取Buffer中的数据，此时position的位置为0，limit变为5，capacity不变。

# Selector

Selector（选择器）是Java NIO中能够检测一到多个NIO通道，并能够知晓通道是否为诸如读写事件做好准备的组件。这样，一个单独的线程可以管理多个channel，从而管理多个网络连接。

## 创建

```java
 Selector selector = Selector.open();
```

## 注册

```java
// channel设置为非阻塞
channel.configureBlocking(false);    
//channel根据感兴趣的状态，注册到selector
SelectionKey key = channel.register(selector, Selectionkey.OP_READ | SelectionKey.OP_WRITE);    
```

## 就绪的channel

```java
// 阻塞到至少有一个通道在你注册的事件上就绪了
int select();
// 和select()一样，除了最长会阻塞timeout毫秒
int select(long timeout);
// 不会阻塞，不管什么通道就绪都立刻返回
int selectNow();
```

## SelectionKey

SelectionKey对象是用来跟踪注册事件的句柄。在SelectionKey对象的有效期间，Selector会一直监控与SelectionKey对象相关的事件，如果事件发生，就会把SelectionKey对象加入到selected-keys集合中。

| SelectionKey.OP_CONNEC | 连接就绪 |
| ---------------------- | -------- |
| SelectionKey.OP_ACCEPT | 接收就绪 |
| SelectionKey.OP_READ   | 读就绪   |
| SelectionKey.OP_WRITE  | 写就绪   |

## 例子

```java
Selector selector = Selector.open();
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
while(true) {
  int readyChannels = selector.select();
  if(readyChannels == 0) continue;
  Set selectedKeys = selector.selectedKeys();
  Iterator keyIterator = selectedKeys.iterator();
  while(keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.
    } else if (key.isConnectable()) {
        // a connection was established with a remote server.
    } else if (key.isReadable()) {
        // a channel is ready for reading
    } else if (key.isWritable()) {
        // a channel is ready for writing
    }
    keyIterator.remove();
  }
}
```