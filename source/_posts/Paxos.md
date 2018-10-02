---
title: Paxos算法
date: 2018-10-02 11:01:28
categories:
- 分布式
tags:
- Paxos
- 分布式一致性
---

# Paxos算法

Paxos算法是一种基于消息传递且具有高度容错性的一致性算法，是最有效的解决分布式一致性问题的算法之一。

# 参与角色

- Proposer： 倡议者可以提出提案以供投票表决
- Acceptor：提案接收者，并可以进行投票表决
- Learner：提案接受者，没有投票权

# 条件

1. 一个Acceptor必须批准它收到的第一个提案
2. 如果一个提案[M，V]被选定后，那么之后任何Proposer产生的编号更高的提案，其Value值都为V

# 算法陈述

Paxos算法需要一个类似于两阶段提交的执行过程。

## 阶段一

1. Proposer选择一个提案编号M，然后向Acceptor的某个超过半数的子集成员发送编号为M的**Prepare请求**。
2. 如果一个Acceptor收到一个编号为M的Prepare请求，且编号M大于该Acceptor已经响应的所有Prepare请求的编号，那么它就会将它已经批准过的最大编号的提案作为响应反馈给Proposer，同时该Acceptor会承诺不会再批准任何编号小于M的提案。

## 阶段二

1. 如果Proposer收到来自半数以上Acceptor对其发出编号为M的Prepare请求的响应后，那么它就会发送一个针对[M，V]提案的Accept请求给半数以上的Acceptor。注意：V就是收到的响应中编号最大的提案的值，如果响应中不包含任何提案，那么V就由Proposer自己决定。
2. 如果Acceptor收到一个针对[M，V]提案的Accept请求，只要该Acceptor没有对编号大于M的Prepare请求做出过响应，它就接受该提案。