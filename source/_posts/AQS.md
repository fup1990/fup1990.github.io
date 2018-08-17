---
title: AQS
date: 2018-08-16 18:21:03
categories:
- 锁
tags:
- 并发

---

AQS是AbstractQueuedSynchronizer类的简称，即队列同步器。它是构建锁或者其他同步组件的基础框架。

AQS定义两种资源共享方式：Exclusive（独占，只有一个线程能执行，如ReentrantLock）和Share（共享，多个线程可同时执行，如Semaphore/CountDownLatch）。

AQS 使用一个 volatile int 类型的成员变量 state 来表示同步状态：

- 当 state > 0 时，表示已经获取了锁。
- 当 state = 0 时，表示释放了锁。

AQS 通过内置的 FIFO 同步队列来完成资源获取线程的排队工作。如果当前线程获取同步状态失败（锁）时，AQS 则会将当前线程以及等待状态等信息构造成一个节点（Node）并将其加入同步队列，同时会阻塞当前线程 当同步状态释放时，则会把节点中的线程唤醒，使其再次尝试获取同步状态。

![img](AQS/CLH.png)