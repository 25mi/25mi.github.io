---
layout: post
published: false
title: 翻译：Sidetree的Node.js实现文档
---
This document focuses on the Node.js implementation of the Sidetree protocol.

本文档重点介绍Sidetree协议的Node.js实现。

## Overview 概述

![Architecture diagram](./diagrams/architecture.png)

## Node Types 节点类型

There will exist several Sidetree node configurations, which offer a variety of modes that support different features and trade-offs. The choice to run one type or another largely depends on the type of user, machine, and intent the operator has in mind.

Sidetree有多种节点类型，它们提供各种模式，支持不同的特性和权衡。运行一种类型或另一种类型的选择在很大程度上取决于操作员所考虑的用户、机器和意图的类型。

### Full Node 全节点

A full node offers the largest set of features and highest resolution performance of DIDs, but also requires more significant bandwidth, hardware, storage, and system resource consumption to operate. A full node will attempt to fetch and retain all data associated with the Sidetree operations present in the target system. As such, full nodes are able to quickly resolve DID lookup requests and may feature more aggressive caching of DID state than other node configurations.

全节点提供了最完整的功能集合与最好的DIDs解析性能，但也需要更大的带宽、硬件、存储和系统资源才能运行。一个全节点将尝试获取并保留与目标系统中存在的Sidetree操作相关的所有数据。因此，全节点能够快速解析DID，并且比其它节点的DID缓存具有更积极的特性。

### Light Node 轻节点

A light node is a node that retains the ability to independently resolve DIDs without relying on a trusted party or trusted assertions by other nodes, while minimizing the amount of bandwidth and data required to do so. Light nodes run a copy of the target system's node (e.g. a blockchain) and fetch all minimal _anchor file_ data required to create an independent mapping that enables just-in-time resolution of DIDs.

轻节点保留独立解析DIDs的能力，而不依赖于受信任方或者其它节点的可信断言，同时最大程度地减少了这样做所需的带宽和数据量。轻节点运行目标系统节点（例如区块链）的副本并获取创建独立映射所需的所有最少 _锚文件_ 数据，以实现DID的实时解析。

> NOTE: Light node support is in development, with release of a supporting node implementation in May 2019.

> 注意：轻节点的支持正在开发中，将在2019年5月发布节点实现。

## Operation Processor 操作处理器

The Operation Processor holds most of the state of a Sidetree node. It is a singleton class with the following methods for DID Document state update and retrieval.

操作处理器保持Sidetree节点的大部分状态。它是一个单体类，具有以下DID文档状态更新和检索方法。

### Process 处理

This is the core method to update the state of a DID Document:

这是更新DID文档状态的核心方法：

```javascript
public process (transactionNumber: number, operationIndex: number, operation: Operation)
```
The `operation` is a JSON object representing a create, update, or a delete operation. Recall from the protocol description that the hash of this object is the *operation hash*, which represents the version of the document produced as the result of this operation.

`operation`是一个用于表示创建、更新或删除操作的JSON对象。在协议规范中该对象的哈希值是 *操作哈希值* ，它表示此操作结果生成的文档版本。

The `transactionNumber` and `operationIndex` parameters together provides a deterministic ordering of all operations. The `transactionNumber` is a monotonically increasing number (need NOT be by 1) that identifies a Sidetree transaction. The `operationIndex` is the index of this operation amongst all the operations batched within the same Sidetree transaction.

transactionNumber和operationIndex参数一起提供所有操作的确定性顺序。 transactionNumber是一个单调递增的数字（不一定是1），用于标识Sidetree交易。operationIndex是在同一Sidetree交易中批处理的所有操作中，此操作的索引。

> Note: `transactionNumber` and `operationIndex` are explicitly called out as parameters for the `process` method for clarity. They may be embedded within the `operation` parameter in actual implementation.

注意:为了清晰起见，`transactionNumber`和`operationIndex`被显式地作为`process`方法的参数调用。它们可以嵌入到实际实现中的操作参数中。

Under normal processing, the _observer_ would process operations in chronological order. However the implementation accepts `process` calls with out-of-ordered operations. This is used to handle delays and out-of-orderedness introduced by the CAS layer.

在正常处理下，_观察者_ 将按时间顺序处理操作。但是，由于CAS层引入的延迟和无序，实现需要考虑接受具有无序操作的流程调用。

It is useful to view the operations as producing a collection of version *chains* one per DID. Each create operation introduces a new chain and each update operation adds an edge to an existing chain. There could be holes in the chain if some historical update is missing - as noted above, this could be caused due to CAS delays.

将操作视为一个版本链集合（每个DID生成一个版本链）很有用。 每个create操作都会引入一个新链，每个更新操作都会为现有链添加一条边。 如果缺少一些历史更新，链上可能会出现漏洞 - 如上所述，这可能是由于CAS延迟造成的。

When two update operations reference the same (prior) version of a DID Document, the cache removes from consideration the later of the two operations and all operations directly and indirectly referencing the removed operation. This ensures that the document versions of a particular DID form a chain without any forks. For illustration, assume we have recorded four operations for a particular DID producing the following chain:

当两个更新操作引用DID文档的相同(先前的)版本时，缓存将移除两个操作的后一个操作，或者直接或间接引用删除操作的操作。这确保了特定文档的版本确实形成了一个没有分叉的链。举个例子，假设我们已经记录了一个特定DID的四个操作，产生了如下链:
```
v0 -> v1 -> v2 -> v3
```
If we find an earlier update operation `v0 -> v4`, the new chain for the DID would be:

如果我们找到更早的更新操作 `v0 -> v4`，则DID的新链将是：
```
v0 -> v4
```

In the above description, *earlier* and *later* refer to the logical time of the operation derived from the position of the operation in the blockchain.

在以上描述中，*较早* 和 *较晚* 指的是从区块链中的操作的位置导出的操作的逻辑时间。(译者注：简而言之，如果操作链因为延迟或各种问题分叉，那么分叉将会选择较早的或者没有删除操作的分叉以确保DID安全)

### Rollback 回滚

This method is used to handle rollbacks (forks) in the blockchain.

此方法用于处理区块链中的回滚（分叉）。

```javascript
public rollback (transactionNumber: number)
```

The effect of this method is to delete the effects of any operation included in a transaction with a transaction number greater than or equal to the _transactionNumber_ provided.

此方法的作用是删除交易序号大于或等于 _transactionNumber_ 的交易中包含的任何操作的效果。

### Resolve 解析

The resolve method returns the latest document for a given DID.

resolve方法返回指定DID的最新文档。

## Batch Writer 批处理写入器
The Batch Writer batches pending (Create, Update, Delete and Recover) operations and anchors them on the blockchain at a periodic interval.

批处理写入器批量处理操作（创建、更新、删除和恢复），并定期将操作锚定在区块链上。

The batching interval can specified by the `batchingIntervalInSeconds` configuration parameter.

批处理周期可以在 `batchingIntervalInSeconds` 配置参数中指定。

## Observer 观察者

The _Observer_ watches the public blockchain to identify Sidetree operations, then parses the operations into data structures that can be used for efficient DID resolutions.
The primary goals for the _Observer_ are to:

_观察者_ 通过监视区块链来识别Sidetree操作，然后将这些操作处理成用于解析DID的数据结构。_观察者_ 的目标是：

1. Maximize ingestion processing rate.

   最大化操作提取处理速度。
1. Allow horizontal scaling for high DID resolution throughput.

   允许水平扩展以获得高DID解析通量。
1. Allow sharing of the processed data structure by multiple Sidetree nodes to minimize redundant computation.

   允许多个Sidetree节点共享已处理的数据结构，以最大限度地减少冗余计算。

The above goals lead to a design where minimal processing of the operations at the time of ingestion and defers the heavy processing such as signature validation and JSON patch to the time of DID resolution.

上述目标导致了一种设计，即在操作提取时进行最少的处理，并且将复杂处理（例如签名验证和JSON补丁）推迟到DID解析时进行。

### Blockchain REST API 区块链REST API
The blockchain REST API interface aims to abstract the underlying blockchain away from the main protocol logic. This allows the underlying blockchain to be replaced without affecting the core protocol logic. The interface also allows the protocol logic to be implemented in an entirely different language while interfacing with the same blockchain.

区块链REST API接口旨在从主协议逻辑抽象底层的区块链。这允许在不影响核心协议逻辑的情况下替换底层区块链。该接口还允许使用完全不同的语言实现协议逻辑，同时使用相同的区块链接口。

### Response HTTP status codes HTTP响应状态码

| HTTP status code | Description                              |
| ---------------- | ---------------------------------------- |
| 200              | Everything went well.                    |
| 400              | Bad client request.                      |
| 401              | Unauthenticated or unauthorized request. |
| 404              | Resource not found.                      |
| 500              | Server error.                            |



### Get latest blockchain time 获取最新的区块链时间
Gets the latest logical blockchain time. This API allows the Observer and Batch Writer to determine protocol version to be used.

获取最新的逻辑区块链时间。此API允许观察者和批处理编写器确定要使用的协议版本。

A _blockchain time hash_ **must not** be predictable/pre-computable, a canonical implementation would be to use the _block number_ as the time and the _block hash_ as the _time hash_. It is intentional that the concepts related to _blockchain blocks_ are  hidden from the layers above.

区块链时间哈希必须不可预测/预计算，规范实现将使用块号作为时间，块散列作为时间散列。有意将与区块链块相关的概念从上面的层中隐藏起来。

|                     |      |
| ------------------- | ---- |
| Minimum API version | v1.0 |

#### Request path 请求路径
```
GET /<api-version>/time
```

#### Request headers 请求头
None.

#### Request body schema 请求体格式
None.

#### Request example 请求示例
```
Get /v1.0/time
```

#### Response body schema 响应体格式
```json
{
  "time": "The logical blockchain time.",
  "hash": "The hash associated with the blockchain time."
}
```

#### Response body example 响应体示例
```json
{
  "time": 545236,
  "hash": "0000000000000000002443210198839565f8d40a6b897beac8669cf7ba629051"
}
```



### Get blockchain time by hash 通过哈希值获取区块链时间
Gets the time identified by the time hash.

|                     |      |
| ------------------- | ---- |
| Minimum API version | v1.0 |

#### Request path 请求路径
```
GET /<api-version>/time/<time-hash>
```

#### Request headers 请求头
None.

#### Request body schema 请求体格式
None.

#### Request example 请求示例
```
Get /v1.0/time/0000000000000000001bfd6c48a6c3e81902cac688e12c2d87ca3aca50e03fb5
```

#### Response body schema 响应体格式
```json
{
  "time": "The logical blockchain time.",
  "hash": "The hash associated with the blockchain time, must be the same as the value given in query path."
}
```

#### Response body example  响应体示例
```json
{
  "time": 545236,
  "hash": "0000000000000000002443210198839565f8d40a6b897beac8669cf7ba629051"
}
```



### Fetch Sidetree transactions 获取Sidetree交易
Fetches Sidetree transactions in chronological order.

按时间顺序获取Sidetree交易。

> Note: The call may not to return all Sidetree transactions in one batch, in which case the caller can use the transaction number of the last transaction in the returned batch to fetch subsequent transactions.

> 注意:调用可能不会一次返回所有Sidetree交易，在这种情况下，调用者可以使用返回批中的最后一个交易的交易序号来获取后续交易。

|                     |      |
| ------------------- | ---- |
| Minimum API version | v1.0 |

#### Request path 请求路径
```
GET /<api-version>/transactions?since=<transaction-number>&transaction-time-hash=<transaction-time-hash>
```

#### Request headers 请求头
None.


#### Request query parameters 请求查询参数
- `since`

  Optional. A transaction number. When not given, all Sidetree transactions since inception will be returned.
  When given, only Sidetree transactions after the specified transaction will be returned.

  可选，一个交易序号。如果不指定，则返回自创建以来的所有Sidetree交易。当指定时，只返回指定交易之后的Sidetree交易。

- `transaction-time-hash`

  Optional, but MUST BE given if `since` parameter is specified.

  可选，但如果指定了since参数，则必须指定。

  This is the hash associated with the time the transaction specified by the `since` parameter is anchored on blockchain.
  Multiple transactions can have the same _transaction time_ and thus the same _transaction time hash_.

  这是与 `since` 参数指定在区块链上的交易时间对应的哈希值。多个交易可以具有相同的交易时间，因此具有相同的交易时间哈希值。

  The _transaction time hash_ helps the blockchain layer detect block reorganizations (temporary forks); `HTTP 400 Bad Request` with `invalid_transaction_number_or_time_hash` as the `code` parameter value in a JSON body is returned on such events.

  _交易时间哈希值_ 帮助区块链层检测区块重组(临时分叉)；当检测到区块重组（译者注：区块分支被修剪）时，请求会返回`HTTP 400 错误请求`并在JSON格式的返回体当中设置`code`参数为`invalid_transaction_number_or_time_hash`

#### Request example 请求示例
```
GET /v1.0/transactions?since=170&transaction-time-hash=00000000000000000000100158f474719e5a319933856f7f464fcc65a3cb2253
```

#### Response body schema 返回体格式
```json
{
  "moreTransactions": "True if there are more transactions beyond the returned batch. False otherwise.",
  "transactions": [
    {
      "transactionNumber": "A monotonically increasing number (need NOT be by 1) that identifies a Sidtree transaction.",
      "transactionTime": "The logical blockchain time this transaction is anchored. Used for protocol version selection.",
      "transactionTimeHash": "The hash associated with the transaction time.",
      "anchorFileHash": "Hash of the anchor file of this transaction."
    },
    ...
  ]
}
```

#### Response example 响应示例
```http
HTTP/1.1 200 OK

{
  "moreTransactions": false,
  "transactions": [
    {
      "transactionNumber": 89,
      "transactionTime": 545236,
      "transactionTimeHash": "0000000000000000002352597f8ec45c56ad19994808e982f5868c5ff6cfef2e",
      "anchorFileHash": "QmWd5PH6vyRH5kMdzZRPBnf952dbR4av3Bd7B2wBqMaAcf"
    },
    {
      "transactionNumber": 100,
      "transactionTime": 545236,
      "transactionTimeHash": "00000000000000000000100158f474719e5a319933856f7f464fcc65a3cb2253",
      "anchorFileHash": "QmbJGU4wNti6vNMGMosXaHbeMHGu9PkAUZtVBb2s2Vyq5d"
    }
  ]
}
```

#### Response example - Block reorganization detected 响应示例 - 检测到区块重组

```http
HTTP/1.1 400 Bad Request

{
  "code": "invalid_transaction_number_or_time_hash"
}
```


### Get first valid Sidetree transaction 获取首个有效的Sidetree交易
Given a list of Sidetree transactions, returns the first transaction in the list that is valid. Returns 404 NOT FOUND if none of the given transactions are valid. This API is primarily used by the Sidetree core library to determine a transaction that can be used as a marker in time to reprocess transactions in the event of a block reorganization (temporary fork).

提供一个Sidetree交易列表，返回其中返回第一个有效的交易。如果列表中的交易都是无效的，则返回 404 NOT FOUND。这个API主要由Sidetree核心库用于确定一个交易，该交易可以在区块重组（临时分叉）时做为标记，用来重新处理交易。

|                     |      |
| ------------------- | ---- |
| Minimum API version | v1.0 |

#### Request path 请求路径
```http
POST /<api-version>/transactions/firstValid HTTP/1.1
```

#### Request headers 请求头
| Name                  | Value                  |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/json``` |

#### Request body schema 请求体格式
```json
{
  "transactions": [
    {
      "transactionNumber": "The transaction to be validated.",
      "transactionTime": "The logical blockchain time this transaction is anchored. Used for protocol version selection.",
      "transactionTimeHash": "The hash associated with the transaction time.",
      "anchorFileHash": "Hash of the anchor file of this transaction."
    },
    ...
  ]
}
```

#### Request example 请求示例
```http
POST /v1.0/transactions/firstValid HTTP/1.1
Content-Type: application/json

{
  "transactions": [
    {
      "transactionNumber": 19,
      "transactionTime": 545236,
      "transactionTimeHash": "0000000000000000002352597f8ec45c56ad19994808e982f5868c5ff6cfef2e",
      "anchorFileHash": "Qm28BKV9iiM1ZNzMsi3HbDRHDPK5U2DEhKpCYhKk83UPEg"
    },
    {
      "transactionNumber": 18,
      "transactionTime": 545236,
      "transactionTimeHash": "0000000000000000000054f9719ef6ca646e2503a9c5caac1c6ea95ffb4af587",
      "anchorFileHash": "Qmb2wxUwvEpspKXU4QNxwYQLGS2gfsAuAE9LPcn5LprS1nb"
    },
    {
      "transactionNumber": 16,
      "transactionTime": 545200,
      "transactionTimeHash": "0000000000000000000f32c84291a3305ad9e5e162d8cc363420831ecd0e2800",
      "anchorFileHash": "QmbBPdjWSdJoQGHbZDvPqHxWqqeKUdzBwMTMjJGeWyUkEzK"
    },
    {
      "transactionNumber": 12,
      "transactionTime": 545003,
      "transactionTimeHash": "0000000000000000001e002080595267fe034d370897b7b506d119ad29da1541",
      "anchorFileHash": "Qmss3gKdm9uU9YLx3MPRHQTcUq1CR1Xv9Zpdu7EBG9Pk9Y"
    },
    {
      "transactionNumber": 4,
      "transactionTime": 544939,
      "transactionTimeHash": "00000000000000000000100158f474719e5a319933856f7f464fcc65a3cb2253",
      "anchorFileHash": "QmdcDrVPWy3ZXoZcuvFq7fDVqatks22MMqPAxDqXsZzGhy"
    }
  ]
}
```

#### Response body schema 响应体格式
```json
{
  "transactionNumber": "The transaction number of the first valid transaction in the given list",
  "transactionTime": "The logical blockchain time this transaction is anchored. Used for protocol version selection.",
  "transactionTimeHash": "The hash associated with the transaction time.",
  "anchorFileHash": "Hash of the anchor file of this transaction."
}
```

#### Response example 响应示例
```http
HTTP/1.1 200 OK

{
  "transactionNumber": 16,
  "transactionTime": 545200,
  "transactionTimeHash": "0000000000000000000f32c84291a3305ad9e5e162d8cc363420831ecd0e2800",
  "anchorFileHash": "QmbBPdjWSdJoQGHbZDvPqHxWqqeKUdzBwMTMjJGeWyUkEzK"
}
```

#### Response example - All transactions are invalid 响应示例 - 所有交易均无效
```http
HTTP/1.1 404 NOT FOUND
```


### Write a Sidetree transaction 写入一个Sidetree交易
Writes a Sidetree transaction to the underlying blockchain.

将Sidetree交易写入底层区块链。

|                     |      |
| ------------------- | ---- |
| Minimum API version | v1.0 |

#### Request path 请求路径
```
POST /<api-version>/transactions
```

#### Request headers 请求头
| Name 字段名                | Value  值              |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/json``` |

#### Request body schema 请求体格式
```json
{
  "anchorFileHash": "The hash of a Sidetree anchor file."
}
```

#### Request example 请求示例
```http
POST /v1.0/transactions HTTP/1.1

{
  "anchorFileHash": "QmbJGU4wNti6vNMGMosXaHbeMHGu9PkAUZtVBb2s2Vyq5d"
}
```

#### Response body schema 响应体格式
None.




## CAS REST API Interface CAS REST API 接口
The CAS (content addressable storage) REST API interface aims to abstract the underlying Sidetree storage away from the main protocol logic. This allows the CAS to be updated or even replaced if needed without affecting the core protocol logic. Conversely, the interface also allows the protocol logic to be implemented in an entirely different language while interfacing with the same CAS.


CAS（内容可寻址存储）REST API接口旨在将基础Sidetree存储从主协议逻辑中抽象出来。 这允许在不影响核心协议逻辑的情况下更新或甚至替换CAS。 相反，该接口还允许协议逻辑以完全不同的语言实现，同时与相同的CAS连接。

All hashes used in the API are encoded multihash as specified by the Sidetree protocol.

API中使用的所有哈希都是由Sidetree协议指定的编码multihas。

### Response HTTP status codes HTTP状态响应码

| HTTP status code HTTP状态码 | Description  描述                    |
| ---------------- | ---------------------------------------- |
| 200              | Everything went well. 一切顺利                   |
| 400              | Bad client request. 客户端请求错误                     |
| 401              | Unauthenticated or unauthorized request. 未经身份验证或未经授权的请求。 |
| 404              | Resource not found.  资源未找到                    |
| 500              | Server error.  服务器错误                          |


### Read content 读取内容
Read the content of a given address and return it in the response body as octet-stream.

读取给定地址的内容，并在响应体中将其作为八位字节流返回。

|                     |      |
| ------------------- | ---- |
| Minimum API version | v1.0 |

#### Request path 请求路径
```
GET /<api-version>/<hash>?max-size=<maximum-allowed-size>
```

#### Request query parameters 请求查询参数
- `max-size`

  Required.

  必须

  If the content exceeds the specified maximum allowed size, `HTTP 400 Bad Request` with `content_exceeds_maximum_allowed_size` as the value for the `code` parameter in a JSON body is returned.

  如果内容超过指定的最大大小，则返回`HTTP 400 Bad Request`，其中`content_s_maximum_allowed_size`作为响应体JSON的`code`参数值。


#### Request example 请求示例
```
GET /v1.0/QmWd5PH6vyRH5kMdzZRPBnf952dbR4av3Bd7B2wBqMaAcf
```
#### Response headers 请求头
| Name                  | Value                  |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/octet-stream``` |

#### Response example - Resoucre not found 响应示例 - 资源未找到

```http
HTTP/1.1 404 Not Found
```

#### Response example - Content exceeds maximum allowed size 响应示例 - 内容超过最大允许大小

```http
HTTP/1.1 400 Bad Request

{
  "code": "content_exceeds_maximum_allowed_size"
}
```

### Write content 写入内容
Write content to CAS.

将内容写入到CAS。

|                     |      |
| ------------------- | ---- |
| Minimum API version | v1.0 |

#### Request path 请求路径
```
POST /<api-version>/
```

#### Request headers 请求头
| Name 字段名            | Value  值             |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/octet-stream``` |

#### Response headers 响应头
| Name 字段名            | Value  值              |
| --------------------- | ---------------------- |
| ```Content-Type```    | ```application/json``` |

#### Response body schema 响应体结构
```json
{
  "hash": "Hash of data written to CAS"
}
```

#### Response body example 响应体示例
```json
{
  "hash": "QmWd5PH6vyRH5kMdzZRPBnf952dbR4av3Bd7B2wBqMaAcf"
}
```

