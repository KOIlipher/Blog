---
title: Hyperledger Fabric学习笔记 数据分发
date: 2022-05-13 21:25:04
thumbnailImage: https://img.gejiba.com/images/b800804b3d9a29026f7420469f15d8a7.png
categories: [Hyperledger Fabric]
tags: Blockchain
---
Hyperledger Fabric联盟链中Gossip协议、私有数据组织子集介绍，私有数据分发与使用
<!-- more -->
## Gossip协议

### 特性

安全、可靠、可扩展
保证网络一致性、完整性

- 持续识别节点(状态离线在线)
- 传播账本数据
- 新节点接入网络

### 如何作用

**Peer节点**基于gossip的数据广播操作接收**通道**中其他节点的信息，并将信息**随机**发送给通道上的其他节点 （数量是可配置的），数据交换有如下几种形式：

- PUSH 交换1次  
- PULL 交换2次  
- PUSH、PULL配合使用 交换3次 **（数据一致性角度最佳）**

![PUSH/PULL](https://img.gejiba.com/images/537504f0b98653dbf276b46a2c8bf00b.jpg)

### 优缺点

优点：扩展性、一致性收敛、容错、简单、去中心化（对等）
缺点：冗余（周围节点发送冗余数据）、延迟（随机传播）

## 私有数据分发

### 动机

无需创建通道，就可以实现节点间私有数据的传递。
某一组组织希望在通道内保持对其他组织数据的私有性。
![PrivateData](https://img.gejiba.com/images/a8222059287ffe671fb4b476dca8de44.jpg)

### 授权组织子集

私有数据组织子集的授权在collections_config.json中配置，链码中定义的不同的数据对象（type xxx struct { }）可以有不同的policy，policy中指定的都属于私有数据的授权域，其中的任何一个节点都可以成为背书节点，该配置文件信息会映射到链码。与传统的背书策略指定有所不同，体现在思考角度变化为数据（链码中的数据对象）出发。

```go
// Peers in Org1 and Org2 will have this private data in a side database
type marble struct {
  ObjectType string `json:"docType"`
  Name       string `json:"name"`
  Color      string `json:"color"`
  Size       int    `json:"size"`
  Owner      string `json:"owner"`
}

// Only peers in Org1 will have this private data in a side database
type marblePrivateDetails struct {
  ObjectType string `json:"docType"`
  Name       string `json:"name"`
  Price      int    `json:"price"`
}
```

```json
// collections_config.json
[
  {
       "name": "collectionMarbles",
       "policy": "OR('Org1MSP.member', 'Org2MSP.member')",
       "requiredPeerCount": 0,
       "maxPeerCount": 3,
       "blockToLive":1000000,
       "memberOnlyRead": true
  },

  {
       "name": "collectionMarblePrivateDetails",
       "policy": "OR('Org1MSP.member')",
       "requiredPeerCount": 0,
       "maxPeerCount": 3,
       "blockToLive":3,
       "memberOnlyRead": true
  }
]
```

### 私有数据存储跟踪

- 链码提案的`transient`字段用于私有数据传递。  
- 在未提交交易的时候，私有数据存放在节点的`transient data store`位置。  
- 除了首次客户端提案传递私有数据和通过Gossip分发授权节点私有数据外，其余场景都采用HASH标识，交易提交后HASH落块，临时数据`transient data store`清除，私有数据存入状态数据库。

### 辅助分发

私有数据在组织子集中可以是分布式存储的，背书节点可以通过私有数据的组织子集中的其他节点实现辅助分发，而不仅仅依赖单点的背书节点（即`maxPeerCount=0`情况），达到高可用。

- `maxPeerCount` 和 `requiredPeerCount` 决定背书节点分发私有数据给授权节点的数量约束（collections_config.json 配置）。如果背书节点不能够成功地将私有数据分发到至少 `requiredPeerCount` 的要求，它将会返回一个错误给客户端。

- `pullRetryThreshodld` 表示授权节点在这个门限时间内需要拿到私有数据。（节点配置文件core.yaml -> peer.gossip.pvtData.pullRetryThreshold）。pullRetry过程：在一个被授权节点在提交的时候，若临时存储`transient data store`中没有私有数据的副本，则该授权节点会从其他授权节点尝试PULL私有数据。

{% alert warning %}

注1：pullRetry在`pullRetryThreshodld`内拿到私有数据：存储私有数据，交易提交账本（私有数据HASH），pullRetry时间内未拿到私有数据： 不存储私有数据，交易提交账本（私有数据HASH）。
注2：若节点有被授权但最后未得到私有数据，那么该节点在将来无法背书引用该私有数据的交易，背书无法在状态数据库查询到私有数据的HASH对应数据，到时候链码会报错。
{% endalert %}

参考Doc：[在 Fabric 中使用私有数据](https://hyperledger-fabric.readthedocs.io/zh_CN/release-2.2/private_data_tutorial.html#pd-read-write-private-data)