redis有丰富的数据类型，不同的数据类型在redis的内部使用不同的对象进行存储。大致的看来有，set,hash,list和module。

先来看一下robj的定义：

```
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits decreas time). */
    int refcount;
    void *ptr;
} robj;
```

特别需要注意的是这里的type和encoding，type表示这个对象的类型，如List或Set，encoding表示此种类型在内存里的存储格式。比如List就可以通过quicklist或dict来实现，这个是存储类型。

通过robj的type可以分为以下几种：
多个type类型可能对应的是一种key 类型， redis里类型只有如下这几种：

```
/* The actual Redis Object */
#define OBJ_STRING 0
#define OBJ_LIST 1
#define OBJ_SET 2
#define OBJ_ZSET 3
#define OBJ_HASH 4

/* The "module" object type is a special one that signals that the object
 * is one directly managed by a Redis module. In this case the value points
 * to a moduleValue struct, which contains the object value (which is only
 * handled by the module itself) and the RedisModuleType struct which lists
 * function pointers in order to serialize, deserialize, AOF-rewrite and
 * free the object.
 *
 * Inside the RDB file, module types are encoded as OBJ_MODULE followed
 * by a 64 bit module type ID, which has a 54 bits module-specific signature
 * in order to dispatch the loading to the right module, plus a 10 bits
 * encoding version. */
#define OBJ_MODULE 5

```

robj的encoding可以有：

一共有如下这些robj type类型，也就是最为底层的数据结构。
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */

使用不同编码方式存储是为了有效的节约存储空间。

所有来自于客户端的数据都会统一包装成robj进行内部传输，以及存储到db里的元素，其实都是robj。

#有关robj的操作方法
对象的使用通过引用计数来实现，refcount，当为0的时候，对象将会被回收。有一类型对象是不会被回收的，那就是共享对象，共享对象的refcount为int的最大值，通过常量`INT_MAX`获取。

不同的原始数据都需要通过object里的方法封装成robj再使用，不同的数据类型将会选择不同的封装方法，封装的robj直接通过type来进行区分。


每一种object都有对应的free方法，如freeStringObject、freeListObject。

object引用计数的增加和减少通过incrRefCount和decrRefCount来操作，如果再减少计数的时候发现为0则会对对象进行free操作。

原始的数据通过上述相关的方法将会被封装成对象robj，如果需要拿到原始的值，需要通过对应的方法才可以。redis定义了一些类似这样的方法可以来使用，`getxxxxFromObject `。

redis还定义了一些类似`getLongDoubleFromObjectOrReply`的方法，这个在从object取得数据发生错误时，会将数据发送给client。
