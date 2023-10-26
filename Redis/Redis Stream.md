# Redis Stream

> [Redis Stream | Redis](https://redis.io/docs/data-types/streams/)

Redis Stream 是一种类似于 *append-only log* 的数据结构，其基于 *Radix Tree* 进行实现，其除了串型存储数据外还有以下特点：

- ${\rm O}(1)$ 的 *random access*
- 复杂的消费策略（例如消费者组 *consumer group*）
- *etc.*

## Entry ID

Stream 中的每个元素被叫做 *entry*，每个 *entry* 有一个 *Unique ID*

*entry ID* 有两种生成策略：

- 由 Redis 自动生成一个由两部分组成的字符串
	- 形式：消息生成时间-序列号 `<millisecondsTime>-<sequenceNumber>`（例如 `1692632086370-0`）
		- 消息生成时间：“以毫秒为单位的时间戳”
		- 序列号：一个递增的整数，用于在同一毫秒内对消息进行排序
- 由用户指定一个 *ID*
	- 该 *ID* 生成策略需要保证唯一且有序

> 为什么 Redis Stream 生成的 *entry ID* 要包含时间呢？
> 
> 因为包含了时间的 *entry ID* 可以方便 Redis Stream 使用 `XRANGE` 进行高效的“范围查询”

## 向 Stream 中加入 Entry `XADD`

> [XADD | Redis](https://redis.io/commands/xadd/)

当向 Stream 加入 *entry* 时，我们需要告诉 Redis：

- 这个数据类型为 Stream 的 Redis KV-pair 的 Key 是什么
- 怎么生成 *entry ID*
- 这个数据类型为 Stream 的 Redis KV-pair 的 Value 是什么

```
> XADD race:france * rider Castilla speed 29.9 position 1 location_id 2
"1692632147973-0"
```

这个命令为 key 为 `race:france` 的 Stream 对象新建了一个包含了 `rider: Castilla, speed: 29.9, position: 1, location_id: 2` 属性的 *entry*，并使用 `*` 来指定让 Redis 自动生成一个 `entry ID`（第二行看到 Redis 返回了 `1692632147973-0`）

## 从 Stream 获取当前 Entry 数量  `XLEN`

> [XLEN | Redis](https://redis.io/commands/xlen/)

获取指定 Stream 对象中当前 *entries* 的总数量

```
> XLEN race:france
(integer) 4
```

## 从 Stream 读取 Entry

### 根据“范围”获取 Entries `XRANGE` `XREVRANGE`

每个 Stream 对象中存储的 Entries 都可以视作为一个“时间序列 *time series*”，因此我们可以在这个时间序列中通过 `XRANGE` 或 `XREVRANGE` 来获取一个“*ID* 返回”或“时间范围”中的所有 *entries*

`XREVRANGE` 是 `XRANGE` 倒序

#### 从头到尾获取所有 Entries

> Redis 提供了两个特殊的符号 `-` 和 `+`：`-` 代表该 Stream 中的第 1 条消息；`+` 代表该 Stream 中的最后 1 条消息

```
> XRANGE race:france - +
```

#### 获取一定范围内的消息

```
> XRANGE race:france 1692632086369 1692632086371    # 两个时间戳/ID
```

#### 获取一定范围内的前 `N` 个消息

> Redis 提供了 `COUNT` 参数，用于进行 Limit 操作

```
> XRANGE race:france - + COUNT 2    # 前 2 个
```

### 监听新的 Entries `XREAD`

Redis Stream 和 Redis Pub/Sub 有点像，但是 Stream 支持了更多的特性：

- 每个 Stream 都可以有多个监听数据的 *clients* (*consumers*)，每个新到达的 *entry* 都会被 Stream 给 Fan-out 转发到每一个 *consumer*
- 不像 Pub/Sub 的即发即删，Stream 会存储所有的 *entries*（除非被主动删除），因此 *consumer* 可以通过保存“最后收到的消息的 ID”来在 Stream 时间序列中找到“未读的新消息”（例如有 Stream `0, 1, 2, 3, 4, 5, 6`，*consumer* 记录下从当前读到的最后的 *ID* `6`；当 Stream 有新元素后变为 `0, 1, 2, 3, 4, 5, 6, 7, 8, 9`，此时 *consumer* 再去读消息时读取范围 $(6, +\infty)$ 的消息即可）
- Stream *Consumer Groups* 消费者组提供了 Pub/Sub 或 Blocking List 无法提供的控制层级

