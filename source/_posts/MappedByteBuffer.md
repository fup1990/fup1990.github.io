---
title: MappedByteBuffer
date: 2018-08-20 17:51:43
categories:
- IO
tags:
- NIO
- buffer
---

# 内存管理

- MMC：CPU的内存管理单元
- 物理内存：内存条的内存空间
- 虚拟内存：计算机系统内存管理的一种技术，它使得应用程序认为它拥有连续的可用内存（一个连续完整的地址空间）；而实际上，它通常是被分隔成多个物理内存碎片，还有部分暂时存储在外部磁盘存储器上，在需要时进行数据交换
- 页面文件：操作系统反映构建并使用虚拟内存的硬盘空间大小而创建的文件
- 缺页中断：当程序试图访问已映射在虚拟空间中但未被加载到物理内存的一个分页时，由MMC发出中断；如果操作系统判断此次访问是有效的，则尝试将相关的页从虚拟内存文件中载入物理内存

# MappedByteBuffer

MappedByteBuffer继承自ByteBuffer，是java nio引入的文件内存映射方案，读写性能极高。很多高性能Java应用都使用内存映射文件来持久化数据。

| 常用操作 | 说明                       |
| -------- | -------------------------- |
| get      | 读数据                     |
| put      | 写数据                     |
| force    | 强制将内存中的数据写入文件 |
| slice    | 缓存区分片                 |

## 示例

```java
public static void main(String[] args) throws HttpProcessException, NoSuchMethodException {

	File file = new File("E://data.txt");
	long len = file.length();
	byte[] ds = new byte[(int) len];
	try {
		MappedByteBuffer mappedByteBuffer = new RandomAccessFile(file, "rw")
				.getChannel()
				//读写的方式将文件map到虚拟内存
				.map(FileChannel.MapMode.READ_WRITE, 0, len);
		for (int offset = 0; offset < len; offset++) {
			//从MappedByteBuffer中读数据
			byte b = mappedByteBuffer.get();
			ds[offset] = b;
		}
		System.out.println(new String(ds));
		for (int offset = 0; offset < len; offset++) {
			//往MappedByteBuffer中写数据
			mappedByteBuffer.put(offset, ds[(int)len - offset -1]);
		}
		//刷新到磁盘
		mappedByteBuffer.force();
	} catch (IOException e) {
		e.printStackTrace();
	}

}
```

FileChannel提供了map方法把文件映射到虚拟内存，通常情况可以映射整个文件，如果文件比较大，可以进行分段映射。MappedByteBuffer继承自ByteBuffer，内部维护了一个逻辑地址address。map0()函数返回一个地址address，这样就无需调用read或write方法对文件进行读写，通过address就能够操作文件。底层采用unsafe.getByte方法，通过（address + 偏移量）获取指定内存的数据。

### MapMode类型

- MapMode.READ_ONLY：只读，试图修改得到的缓冲区将导致抛出异常。
- MapMode.READ_WRITE：读/写，对得到的缓冲区的更改最终将写入文件；**但该更改对映射到同一文件的其他程序不一定是可见的**。
- MapMode.PRIVATE：私用，可读可写,但是修改的内容不会写入文件，只是buffer自身的改变，这种能力称之为”copy on write”。

## 分析

采用内存映射的读写效率要比传统的read/write性能高。 read()是系统调用，首先将文件从硬盘拷贝到内核空间的一个缓冲区，再将这些数据拷贝到用户空间，实际上进行了两次数据拷贝。而map()也是系统调用，但没有进行数据拷贝，直接将文件从硬盘拷贝到用户空间，只进行了一次数据拷贝。

## 注意

1.  MappedByteBuffer使用虚拟内存，因此分配(map)的内存大小不受JVM的-Xmx参数限制。
2. 关闭资源。
3. 不要经常调用MappedByteBuffer.force()方法，这个方法强制操作系统将内存中的内容写入硬盘，所以如果你在每次写内存映射文件后都调用force()方法，你就不能真正从内存映射文件中获益，而是跟disk IO差不多。
4. 如果被请求的页面不在内存中，内存映射文件会导致页面错误。
5. MappedByteBuffer和文件映射在缓存被GC之前都是有效的。sun.misc.Cleaner可能是清除内存映射文件的唯一选择。  

## MappedByteBuffer与DirectByteBuffer的区别

这两种类型的buffer的不同在于：MappedByteBuffers是被操作系统分配在虚拟内存空间；而DirectByteBuffer是被分配在可靠的物理内存中。