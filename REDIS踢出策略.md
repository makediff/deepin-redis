[https://redis.io/topics/lru-cache]

LFU策略和LRU原则。


### @todo 需要进一步说明lru的细节。

# 不同的类型，能不能使用相同的名字？ 不能。key在同一个db里是唯一的。所以对每一个key进行操作的时候，都需要先判断类似是不是吻合的。

每次key有数据涉及到修改时，会增加`server.dirty`计数，代表key被修改的次数。
