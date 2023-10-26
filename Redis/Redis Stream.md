# Redis Stream

> 官方文档：[Redis Stream | Redis](https://redis.io/docs/data-types/streams/)

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

> 为什么 Redis Stream 生成的 *entry ID* 要包含时间呢？因为包含了时间的 *entry ID* 可以方便 Redis Stream 使用 `XRANGE` 进行高效的“范围查询”

