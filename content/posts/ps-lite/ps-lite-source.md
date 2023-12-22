---
title: "ps-lite源码阅读笔记-1"
date: 2023-12-21T23:30:04+08:00
draft: false
toc: false
images:
tags:
  - c++
---

# 前言
ps-lite是一个分布式参数服务器，github地址为[ps-lite](https://github.com/dmlc/ps-lite)。该库通过几千行代码，实现了包含分布式节点管理，简单故障恢复，参数自定义分片的基于推送/拉取模式建议服务器。本文将自顶向下的对该项目进行分析，深入源码，了解具体实现原理并分析不足之处。  ps-lite集群拓普结构
# 集群拓普
<p align="center">
  <img src="../images/network-topology.png" alt="ps-lite集群拓普结构">
</p>
<p align="center">ps-lite集群拓普结构</p>  

ps-lite的集群相对简单，由一个scheduler，任意个worker和server组成，支持在任意节点部署任意多个worker和server，并且相同节点的worker和server可以共享同一个端口。  
worker/server可以是独立的进程，也可以是同一个进程的不同线程，不同节点上的worker和server的数量也可以任意指定，因此比较好的支持了异构的分布式环境。  
该集群中只能存在一个scheduler，负责管理worker和server的注册，给每一个协作者分配一个唯一的id。同时scheduler也承担着一定程度的故障恢复能力，在故障节点重新上线/被替换后能够将新的节点信息广播给其他节点。确保集群具有一定的自我修复能力。  
ps-lite不支持协作者数量的动态增加和减少。虽然节点总数是确定的，但在某些节点异常时，可以通过在其他节点上重新启动一个新的节点来替换异常节点，从而实现动态加入/退出的效果。
在ps-lite的通信模型中，集群中的任意两个节点的通信不需要经过scheduler的转发。但由于通信方式的可拓展性，具体实现中也可以通过scheduler转发消息。
集群总体采用了Push/Pull的通信模型，即worker向server发送Push请求进行参数更新，Pull请求拉取当前参数。

<p align="center">
  <img src="../images/cluster-initial.png" alt="ps-lite节点流程">
</p>
<p align="center">ps-lite节点流程</p>  

## Scheduler
Scheduler是整个集群的核心，负责管理整个集群的节点信息，包括节点的注册，节点的故障恢复，节点信息的广播。  
当Scheduler启动时，会监听一个端口，等待worker和server的注册。当一个worker或者server启动时，会向Scheduler发送一个注册请求，包含节点的ip地址和端口号。Scheduler会为该节点分配一个唯一的id，并将该id返回给节点。  
当Scheduler发现已注册的协作者数量达到预设的数量时，会向所有节点广播所有协作者的信息，包括协作者的id，角色，ip地址和端口号。当一个节点收到广播消息后，会将消息中的节点信息保存到本地，从而与其他节点进行通信。  
当Scheduler发现新的节点注册消息时，由于网如果不存在，并且故障协作者列表为空，则为该协作者分配一个新的id，并将该协作者id重新上线的信息广播给其他节点。

## Worker
Worker是协作者的一种，即进行参数消费的使用方。负责向server发送参数更新请求，以及接收server的参数更新。  
Worker最重要的工作是定义特征key到id的映射，即将特征key映射到server的id。这个映射关系可以由用户自定义，也可以使用ps-lite默认的平均分片映射。

## Server
Server是整个集群中相对简单的一环，负责存储参数，接收worker的参数更新请求，以及向worker发送参数更新。
Server会使用用户自定义的requestHandler对参数更新请求进行处理，然后将处理结果返回给worker。Server不对集群中的分片规则做感知，只负责存储/更新/响应。

# 项目结构

<p align="center">
  <img src="../images/project-structure.png" alt="ps-lite节点流程">
</p>
<p align="center">项目结构</p>

该项目主要分为三层，分别是代码接入层，节点通信层，和底层传输层。

## 代码接入层
代码接入层主要提供了简易的接入方式，通过简单的指定服务器自定义处理函数以及参数分片方式便可以简单的初始，并开始使用集群进行参数更新及获取，示例代码如下。
```c++
// 初始化一个server 指定自定义处理函数
auto server = new KVServer<float>(0);
server->set_request_handle(EmptyHandler<float>);

// 初始化一个worker 使用默认的平均分片映射
KVWorker<float> kv(0， 0);

// 向server push参数
std::vector<Key> keys(num);
std::vector<float> vals(num);
kv.Wait(kv.Push(keys， vals));

// 从server pull参数
std::vector<float> rets;
kv.Wait(kv.Pull(keys， &rets));
```

## 节点通信层
节点通信层主要负责节点之间的通信，包括节点的注册，节点信息的广播，节点的故障恢复。
### PostOffice
PostOffice每个进程只有一个实例，负责管理整个进程下全部的协作者的实例(worker，server和scheduler)并管理与外部集群的交互，交互节点的探活等等等。它对进程下全部协作者的生命周期负责，在启动时根据角色调用对应的初始化函数，初始化与外界的通信类Van，阻塞节点直到全部协作者初始换完成，并在收到集群终止消息后，释放协作者资源进行优雅关闭。  
正像一个邮局一样，PostOffice为每个注册的客户Customer提供了消息接收的能力。当有消息到达时，PostOffice会调用Customer的回调函数进行处理。同时对消息的发送也需要使用PostOffice下的成员变量Van进行发送。
### Customer
Customer由代码向PostOffice注册，传递一个唯一的id和一个消息处理函数。当PostOffice收到消息时，会调用Customer的消息处理函数进行处理。
### Van
Van是PostOffice的成员变量，负责节点之间的通信。Van包含了很多虚函数，用于用户实现不同的自定义通信方式。目前支持的通信方式有基于TCP的ZMQ以及P3(Priority-based Parameter Propagation)，以及使用libverbs实现的RDMA通信。Van负责记录不同的节点的通信地址，从而将能够消息发送给指定的节点。Van使用protobuf进行消息的序列化和反序列化，从而实现简单轻量的消息传输。