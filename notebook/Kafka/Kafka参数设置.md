## replica fetcher 线程参数设置

| 参数                                          | 说明                                                         | 默认值  |
| --------------------------------------------- | ------------------------------------------------------------ | ------- |
| num.replica.fetchers                          | 从一个 broker 同步数据的 fetcher 线程数，增加这个值时也会增加该 broker 的 Io 并行度（也就是说：从一台 broker 同步数据，最多能开这么大的线程数） | 1       |
| replica.fetch.wait.max.ms                     | 对于 follower replica 而言，每个 Fetch 请求的最大等待时间，这个值应该比 `replica.lag.time.max.ms` 要小，否则对于那些吞吐量特别低的 topic 可能会导致 isr 频繁抖动 | 500     |
| replica.high.watermark.checkpoint.interval.ms | hw 刷到磁盘频率                                              | 500     |
| replica.lag.time.max.ms                       | 如果一个 follower 在这个时间内没有发送任何 fetch 请求或者在这个时间内没有追上 leader 当前的 log end offset，那么将会从 isr 中移除 | 10000   |
| replica.fetch.min.bytes                       | 每次 fetch 请求最少拉取的数据量，如果不满足这个条件，那么要等待 replicaMaxWaitTimeMs | 1       |
| replica.fetch.backoff.ms                      | 拉取时，如果遇到错误，下次拉取等待的时间                     | 1000    |
| replica.fetch.max.bytes                       | 在对每个 partition 拉取时，最大的拉取数量，这并不是一个绝对值，如果拉取的第一条 msg 的大小超过了这个值，只要不超过这个 topic 设置（defined via message.max.bytes (broker config) or max.message.bytes (topic config)）的单条大小限制，依然会返回。 | 1048576 |
| replica.fetch.response.max.bytes              | 对于一个 fetch 请求，返回的最大数据量（可能会涉及多个 partition），这并不是一个绝对值，如果拉取的第一条 msg 的大小超过了这个值，只要不超过这个 topic 设置（defined via message.max.bytes (broker config) or max.message.bytes (topic config)）的单条大小限制，依然会返回。 | 10MB    |