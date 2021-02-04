# Redis数据结构与对象 

## Redis对象
Redis中的每个对象都是由一个redisObject结构表示
代码：server.c、server.h
```c
typedef struct redisObject {
    unsigned type:4; // 类型
    unsigned encoding:4; // 编码
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;    // 引用计数
    void *ptr;  // 指向底层实现数据结构的指针
} robj;

```
### 对象类型
```c
/* The actual Redis Object */
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```
命令
```
redis> TYPE msg
string
```
### 编码
对象的底层实现数据结构由对象的encoding属性决定
```c
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
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```
命令
```
redis> OBJECT ENCODING msg
"embstr"
```
```
类型	         编码	      对象
 STRING	 ENCODING_INT	    使用整数值实现的字符串对象。
 STRING	 ENCODING_EMBSTR	使用 embstr 编码的简单动态字符串实现的字符串对象。
 STRING	 ENCODING_RAW	    使用简单动态字符串实现的字符串对象。
 LIST	 ENCODING_ZIPLIST	使用压缩列表实现的列表对象。
 LIST	 ENCODING_QUICKLIST	使用双端链表实现的列表对象。
 HASH	 ENCODING_ZIPLIST	使用压缩列表实现的哈希对象。
 HASH	 ENCODING_HT	    使用字典实现的哈希对象。
 SET	 ENCODING_INTSET	使用整数集合实现的集合对象。
 SET	 ENCODING_HT	    使用字典实现的集合对象。
 ZSET	 ENCODING_ZIPLIST	使用压缩列表实现的有序集合对象。
 ZSET	 ENCODING_SKIPLIST	使用跳跃表和字典实现的有序集合对象。
每种类型的对象都至少使用了两种不同的编码,Redis可以根据不同的使用场景来为一个对象设置不同的编码，从而优化对象在某一场景下的效率。
```
例如(object.c)：
```c
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```

## SDS
sds ：Simple Dynamic String，简单动态字符串。
代码：sds.c、sds.h
### sds的作用：
（1）实现字符串对象（StringObject）；
（2）在Redis程序内部用作char* 类型的替代品,因为char*类型的功能单一，抽象层次低，并且不能高效地支持一些Redis常用的操作（比如追加操作和长度计算操作）；

sds采用预分配冗余空间和惰性空间释放的方式来减少内存的频繁分配。
sds是二进制安全的，可以存放任何二进制的数据和文本数据，包括’\0’。

### sds结构:
```c
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

```

### 空间预分配
扩容策略
SDS_MAX_PREALLOC的容量大小定义在sds.h文件中
SDS_MAX_PREALLOC (1024*1024)
```c
len = sdslen(s);
sh = (char*)s-sdsHdrSize(oldtype);
newlen = (len+addlen);
if (newlen < SDS_MAX_PREALLOC)
// 如果新长度小于最大预分配长度则分配扩容为2倍
    newlen *= 2;
else
// 如果新长度大于最大预分配长度则仅追加SDS_MAX_PREALLOC长度,避免浪费
    newlen += SDS_MAX_PREALLOC;

type = sdsReqType(newlen);
```
### 惰性空间释放
当要缩短SDS保存的字符串时，程序并不立即使用内存分配来回收缩短后多出来的空间
```c
void sdsclear(sds s) {
    sdssetlen(s, 0);
    s[0] = '\0';
}

static inline void sdssetlen(sds s, size_t newlen) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            {
                unsigned char *fp = ((unsigned char*)s)-1;
                *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS);
            }
            break;
        case SDS_TYPE_8:
            SDS_HDR(8,s)->len = newlen;
            break;
        case SDS_TYPE_16:
            SDS_HDR(16,s)->len = newlen;
            break;
        case SDS_TYPE_32:
            SDS_HDR(32,s)->len = newlen;
            break;
        case SDS_TYPE_64:
            SDS_HDR(64,s)->len = newlen;
            break;
    }
}
```

## 链表
Redis使用的C语言并没有内置这种数据结构，所以Redis构建了自己的链表实现。
代码：adlist.c、adlist.h

### 链表结构
```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

typedef struct list {
    listNode *head;  // 表头节点
    listNode *tail;  // 表尾节点
    void *(*dup)(void *ptr);  // 节点值复制函数
    void (*free)(void *ptr);  // 节点值释放函数
    int (*match)(void *ptr, void *key); // 节点值对比函数
    unsigned long len;  // 节点数量
} list;
```

Redis 的链表实现的特性可以总结如下：  
双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O(1)。  
无环：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问以NULL为终点。  
带表头指针和表尾指针：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为 O(1) 。  
带链表长度计数器：程序使用list结构的len属性来对list持有的链表节点进行计数，程序获取链表中节点数量的复杂度为 O(1) 。  
多态：链表节点使用void*指针来保存节点值， 并且可以通过 list 结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。  

## 字典
Redis的字典使用哈希表作为底层实现一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。
代码：dict.c、dict.h
### 哈希表
```c
typedef struct dictht {
    dictEntry **table;  // 哈希表数组
    unsigned long size;  // 哈希表数组
    unsigned long sizemask;  // 哈希表大小掩码，用于计算索引值，总是等于 size - 1
    unsigned long used;  // 该哈希表已有节点的数量
} dictht;
```
table 属性是一个数组，数组中的每个元素都是一个指向dict.h/dictEntry结构的指针，每个dictEntry结构保存着一个键值对。
size 属性记录了哈希表的大小，也即是table数组的大小，而used属性则记录了哈希表目前已有节点（键值对）的数量。
sizemask 属性的值总是等于size - 1 ，这个属性和哈希值一起决定一个键应该被放到table数组的哪个索引上面。

### 哈希表节点
```c
typedef struct dictEntry {
    void *key; // 键
    union {    // 值
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;   // 指向下个哈希表节点，形成链表
} dictEntry;

```
key 属性保存着键值对中的键， 而 v 属性则保存着键值对中的值， 其中键值对的值可以是一个指针， 或者是一个 uint64_t 整数， 又或者是一个 int64_t 整数。
next 属性是指向另一个哈希表节点的指针， 这个指针可以将多个哈希值相同的键值对连接在一次， 以此来解决键冲突（collision）的问题。

### 字典
```c
typedef struct dict {
    dictType *type;  // 类型特定函数
    void *privdata;  // 私有数据
    dictht ht[2];    // 哈希表
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */  // rehash索引,当rehash不在进行时，值为 -1
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
type属性是一个指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数， Redis 会为用途不同的字典设置不同的类型特定函数。
而 privdata 属性则保存了需要传给那些类型特定函数的可选参数。
ht属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表，一般情况下，字典只使用ht[0]哈希表，ht[1]哈希表只会在对 ht[0]哈希表进行rehash时使用。
除了ht[1]之外，另一个和rehash有关的属性就是rehashidx；它记录了rehash目前的进度，如果目前没有在进行rehash ，那么它的值为 -1 。
```c
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key); // 计算哈希值的函数
    void *(*keyDup)(void *privdata, const void *key); // 复制键的函数
    void *(*valDup)(void *privdata, const void *obj); // 复制值的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); // 对比键的函数
    void (*keyDestructor)(void *privdata, void *key); // 销毁键的函数
    void (*valDestructor)(void *privdata, void *obj); // 销毁值的函数
} dictType;
```
### 哈希值计算与使用

```c
#define dictHashKey(ht, key) (ht)->type->hashFunction(key)

 h = dictHashKey(ht, key) & ht->sizemask; //使用哈希表的 sizemask 属性和哈希值，计算出索引值
 he = ht->table[h];

```

### REHASH
 哈希表保存的键值对会逐渐地增多或者减少，为了让哈希表的负载因子（load factor）维持在一个合理的范围之内，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。
 扩展和收缩哈希表的工作可以通过执行rehash（重新散列）操作来完成，Redis对字典的哈希表执行rehash的步骤如下：

为字典的ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量（也即是 ht[0].used属性的值）：
如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used * 2 的 2^n （2的n次方幂）；
如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used的 2^n 。
将保存在ht[0]中的所有键值对rehash到 ht[1]上面；rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上。
当ht[0]包含的所有键值对都迁移到了ht[1]之后（ht[0]变为空表），释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表 为下一次rehash做准备。

```c
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

## 整数集合
代码：intset.c、intset.h
```c
/* Note that these encodings are ordered, so:
 * INTSET_ENC_INT16 < INTSET_ENC_INT32 < INTSET_ENC_INT64. */
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))

typedef struct intset {
    uint32_t encoding; // 保存元素所使用的类型的长度
    uint32_t length; // 元素个数
    int8_t contents[]; // 保存元素的数组
} intset;
```

### 升级
当要添加新元素到intset ，并且intset当前的编码，不适用于新元素的编码时，就需要对intset进行升级
```c
    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }
```

## 压缩列表
ziplist结构是一个连续的内存块，由表头、若干个entry节点和压缩列表尾部标识符zlend组成，通过一系列编码规则，提高内存的利用率，用于存储整数和短字符串。
代码：ziplist.c、ziplist.h
ziplist采用的是类似数组的结构：
<zlbytes><zltail><zllen><entry><entry><zlend>
zlbytes：32位无符号整型，存储整个ziplist当前被分配的空间，包含自身的4个字节，在resize的时候通过该变量，无需遍历即可得到ziplist空间大小；
zltail：32位无符号整型，存储ziplist中最后一个entry相对头部的偏移量，用于从尾端pop，如果没有该变量记录，如果要删除最后一个元素需要从头遍历；

zllen：16位无符号整型，记录ziplist中存储的entry个数，受限于16位的范围，当元素个数为2^16-2时，该值会设置成2^16-1，此时删除元素需要遍历整个结构；

entry：最复杂的部分，真正保存数据的单元，记录了前一个元素和自身占用的空间，下文详细介绍；

zlend：8位无符号整型，值为255，没有entry以255开始，遍历时碰到255，说明到了ziplist结尾，entry是通过开头的值来确定entry类型和占用空间的；
```c
/* Create a new empty ziplist. */
unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    unsigned char *zl = zmalloc(bytes); 
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes); //将zlbytes设置为分配的空间大小
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE); //设置zltail
    ZIPLIST_LENGTH(zl) = 0;  // 设置长度
    zl[bytes-1] = ZIP_END;   // ZIP_END 255  标识结束
    return zl;
}
```

### entry结构
<prelen><<encoding+lensize><len>><data>
prelen：前数据项的长度
encoding、len：是此数据项的的类型和数据长度
data: 存储的数据

```c
typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    unsigned int prevrawlen;     /* Previous entry len. */
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                    For example strings have a 1, 2 or 5 bytes
                                    header. Integers always use a single byte.*/
    unsigned int len;            /* Bytes used to represent the actual entry.
                                    For strings this is just the string length
                                    while for integers it is 1, 2, 3, 4, 8 or
                                    0 (for 4 bit immediate) depending on the
                                    number range. */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                    the entry encoding. However for 4 bits
                                    immediate integers this can assume a range
                                    of values and must be range-checked. */
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
} zlentry;

/* Return a struct with all information about an entry. */
void zipEntry(unsigned char *p, zlentry *e) {

    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
    e->headersize = e->prevrawlensize + e->lensize;
    e->p = p;
}
```
## 快速列表
quicklist就是把ziplist和普通的双向链表结合起来。每个双链表节点中保存一个ziplist，然后每个ziplist中存一批list中的数据(具体ziplist大小可配置)，这样既可以避免大量链表指针带来的内存消耗，也可以避免ziplist更新导致的大量性能损耗，将大的ziplist化整为零。
代码:quicklist.c、quicklist.h

```c
/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: 0 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor.
 * 'bookmakrs are an optional feature that is used by realloc this struct,
 *      so that they don't consume memory when not used. */
typedef struct quicklist {
    quicklistNode *head;   // 头结点
    quicklistNode *tail;   // 尾结点
    // 在所有的ziplist中的entry总数
    unsigned long count;        /* total count of all entries in all ziplists */ 
    //quicklist节点总数 
    unsigned long len;          /* number of quicklistNodes */ 
    // 每个节点的最大容量
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    // quicklist的压缩深度，0表示所有节点都不压缩，否则就表示从两端开始有多少个节点不压缩
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    // bookmarks数组的大小，bookmarks是一个可选字段，用来quicklist重新分配内存空间时使用，不使用时不占用空间
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;
```
compress可配置，因为list两端的数据变更最为频繁，像lpush,rpush,lpop,rpop等命令都是在两端操作，如果频繁压缩或解压缩会代码不必要的性能损耗。
### 节点结构
```c
/* quicklistNode is a 32 byte struct describing a ziplist for a quicklist.
 * We use bit fields keep the quicklistNode at 32 bytes.
 * count: 16 bits, max 65536 (max zl bytes is 65k, so max count actually < 32k).
 * encoding: 2 bits, RAW=1, LZF=2.
 * container: 2 bits, NONE=1, ZIPLIST=2.
 * recompress: 1 bit, bool, true if node is temporary decompressed for usage.
 * attempted_compress: 1 bit, boolean, used for verifying during testing.
 * extra: 10 bits, free for future use; pads out the remainder of 32 bits */
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;           // quicklist节点对应的ziplist
    // ziplist的字节数
    unsigned int sz;             /* ziplist size in bytes */
    // ziplist的item数
    unsigned int count : 16;     /* count of items in ziplist */
    // 数据类型，RAW==1 or LZF==2 
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    // 这个节点以前是否压缩过
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    // 未使用到的10位
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

```
fill字段的含义是每个quicknode的节点最大容量，不同的数值有不同的含义，默认是-2，具体数值含义如下：  
-1: 每个quicklistNode节点的ziplist所占字节数不能超过4kb。（建议配置）  
-2: 每个quicklistNode节点的ziplist所占字节数不能超过8kb。（默认配置&建议配置）  
-3: 每个quicklistNode节点的ziplist所占字节数不能超过16kb。  
-4: 每个quicklistNode节点的ziplist所占字节数不能超过32kb。  
-5: 每个quicklistNode节点的ziplist所占字节数不能超过64kb。  
任意正数: 表示：ziplist结构所最多包含的entry个数，最大为215215。  
compress为0表示所有节点都不压缩，否则就表示从两端开始有多少个节点不压缩。  

```c
/* Create a new quicklist.
 * Free with quicklistRelease(). */
quicklist *quicklistCreate(void) {
    struct quicklist *quicklist;

    quicklist = zmalloc(sizeof(*quicklist));
    quicklist->head = quicklist->tail = NULL;
    quicklist->len = 0;
    quicklist->count = 0;
    quicklist->compress = 0;
    quicklist->fill = -2;
    quicklist->bookmark_count = 0;
    return quicklist;
}
```

## 跳跃列表

```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```
