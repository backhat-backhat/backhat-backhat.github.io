---
layout: post
title: "redis源码分析 - 字典结构"
date: 2016-10-30 23:30:00
tags: redis dict
---
redis 字典对象分析

字典，又称为符号表 (symbol table)、关联数组 (associated array)或映射 (map)，是一种用于保存键值对 (key-value pair)的抽象数据结构。在 redis 中，哈希键和数据库都是通过字典作为底层实现的。

# 结构体
redis 中字典的结构如下： 

{% highlight ruby %}
	typedef struct dictEntry {
	    void *key;	//键
	    union {
	        void *val;
	        uint64_t u64;
	        int64_t s64;
	        double d;
	    } v;	//值，使用联合体，可以是指针，也可以是其他类型的值
	    struct dictEntry *next;	//使用拉链法解决哈希冲突问题
	} dictEntry;
{% endhighlight %}

对两个64位类型解释如下

	typedef unsigned __int64 uint64_t;
	typedef signed __int64 int64_t;
	
可以将 `__int64`理解成 `long long`

{% highlight ruby %}
	typedef struct dictType {
	    unsigned int (*hashFunction)(const void *key);
	    void *(*keyDup)(void *privdata, const void *key);
	    void *(*valDup)(void *privdata, const void *obj);
	    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
	    void (*keyDestructor)(void *privdata, void *key);
	    void (*valDestructor)(void *privdata, void *obj);
	} dictType;
{% endhighlight %}

`dictType`结构体为一簇操作特定类型键值对的函数，`privdata`为传给这些函数的可选参数

{% highlight ruby %}
	/* This is our hash table structure. Every dictionary has two of this as we
	 * implement incremental rehashing, for the old to the new table. */
	typedef struct dictht {
	    dictEntry **table;
	    unsigned long size;
	    unsigned long sizemask;
	    unsigned long used;
	} dictht;
{% endhighlight %}

dictht 为哈希表结构，哈希表结构有一个 dictEntry 的双重指针（可理解成 dictEntry 指针数组），size 为哈希表大小，sizemask 为哈希表大小掩码，用于计算哈希索引值，总是等于 `size-1`，used 为哈希表元素个数，即已有的节点的数量。

{% highlight ruby %}	
	typedef struct dict {
	    dictType *type;
	    void *privdata;
	    dictht ht[2];
	    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
	    int iterators; /* number of iterators currently running */
	} dict;
{% endhighlight %}

dict 为字典结构，包括两个哈希表ht[2], 一个用于正常使用，另一个，当需要扩大哈希表时，将ht[0]中的节点rehashed 到 ht[1] 上，然后再将 ht[1] 设置为ht[0], ht[1]置空，准备下一次 rehashed。 rehashidx 记录了 rehash 的进度，当没有做 rehash 时，它的值为 -1， 当在做 rehash 的值时，它的值表示的是当前 rehash 到了 ht[0] 中的哪一个位置了(可以理解成 ht[0].table[rehashidx])。 redis 的 rehash 是渐进式的，即通过 rehashidx 一个一个递增的形式 rehash 的。

# 字典的详细实现
## 计算哈希值和索引值
redis计算哈希值和索引值，是根据键值来计算的，先计算出哈希值，然后根据哈希值和 sizemask 计算索引值。

{% highlight ruby %}
	/* Returns the index of a free slot that can be populated with
	 * a hash entry for the given 'key'.
	 * If the key already exists, -1 is returned.
	 *
	 * Note that if we are in the process of rehashing the hash table, the
	 * index is always returned in the context of the second (new) hash table. */
	static int _dictKeyIndex(dict *d, const void *key)
	{
	    unsigned int h, idx, table;
	    dictEntry *he;
	
	    /* Expand the hash table if needed */
	    if (_dictExpandIfNeeded(d) == DICT_ERR)		//判断是否需要扩大哈希表的大小，如果正在 rehash，返回 true
													//如果used/size 大于安全阀值5时，将需要扩大哈希表大小
	        return -1;
	    /* Compute the key hash value */
	    h = dictHashKey(d, key);		//计算哈希值
	    for (table = 0; table <= 1; table++) {
	        idx = h & d->ht[table].sizemask;	//计算索引值，在哈希表中查找是否存在，存在返回-1，不存在，返回索引值
	        /* Search if this slot does not already contain the given key */
	        he = d->ht[table].table[idx];
	        while(he) {
	            if (dictCompareKeys(d, key, he->key))
	                return -1;
	            he = he->next;
	        }
	        if (!dictIsRehashing(d)) break;
	    }
	    return idx;
	}
{% endhighlight %}

上面这个函数，为获取哈希值和索引值的过程， 首先通过 `dictHashKey(d, key)` 获取哈希值，这是一个宏函数

	#define dictHashKey(d, key) (d)->type->hashFunction(key)
	
然后根据哈希值求索引值，`idx = h & d->ht[table].sizemask`，如果此元素在哈希表中已存在，返回-1，不需要再次插入，否则返回一个可以用的位置。搜索哈希表时，需要在两个哈希表中都要搜索。

那么，当 `_dictExpandIfNeed (d)` 判断出需要扩大哈希表时，是如何扩大哈希表的呢？`_dictExpandIfNeed (d)`这个函数在判断时： <br>
1. 如果 `rehashidx ！= -1` 说明，哈希表在 `rehash`，此时表明就是正在扩大哈希<br>
2. 如果 `used/size` （这两个都是 `dictht` 的成员）大于阀值 `dict_force_resize_ratio`（为5）时，就需要扩大哈希表。

扩大哈希表的函数如下

{% highlight ruby %}
	/* Expand or create the hash table */
	int dictExpand(dict *d, unsigned long size)
	{
	    dictht n; /* the new hash table */
	    unsigned long realsize = _dictNextPower(size);
	
	    /* the size is invalid if it is smaller than the number of
	     * elements already inside the hash table */
	    if (dictIsRehashing(d) || d->ht[0].used > size)	//如果rehashidx不是-1，说明正在扩，不需重复操作；而且扩大后，used 应该小于 size
	        return DICT_ERR;
	
	    /* Rehashing to the same table size is not useful. */
	    if (realsize == d->ht[0].size) return DICT_ERR;
	
	    /* Allocate the new hash table and initialize all pointers to NULL */
	    n.size = realsize;
	    n.sizemask = realsize-1;
	    n.table = zcalloc(realsize*sizeof(dictEntry*));
	    n.used = 0;
	
	    /* Is this the first initialization? If so it's not really a rehashing
	     * we just set the first hash table so that it can accept keys. */
	    if (d->ht[0].table == NULL) {
	        d->ht[0] = n;
	        return DICT_OK;
	    }
	
	    /* Prepare a second hash table for incremental rehashing */
	    d->ht[1] = n;	// 将 ht[1] 设置为扩大后的哈希表，然后将 rehashidx 置为0，表明从 0 开始 rehash
	    d->rehashidx = 0;
	    return DICT_OK;
	}
{% endhighlight %}

此函数需要注意最后一段代码

	    d->ht[1] = n;	
	    d->rehashidx = 0;
		
将扩大后的哈希设置为 ht[1]，然后设置 rehashidx 为0，启动 rehash，将 ht[0] 都 从 ht[0].table[0] 开始全部 rehash 到 ht[1].table 中，后面将详细介绍 redis 的 rehash 的过程。

### 计算哈希值
字典在计算哈希值时，是通过调用宏函数 `(d)->type->hashFunction (key)`得到的，这只是一个函数指针，在 redis 中，有两个方法计算哈希值。

{% highlight ruby %}
	/* MurmurHash2, by Austin Appleby
	 * Note - This code makes a few assumptions about how your machine behaves -
	 * 1. We can read a 4-byte value from any address without crashing
	 * 2. sizeof(int) == 4
	 *
	 * And it has a few limitations -
	 *
	 * 1. It will not work incrementally.
	 * 2. It will not produce the same results on little-endian and big-endian
	 *    machines.
	 */
	unsigned int dictGenHashFunction(const void *key, int len) {
	    /* 'm' and 'r' are mixing constants generated offline.
	     They're not really 'magic', they just happen to work well.  */
	    uint32_t seed = dict_hash_function_seed;
	    const uint32_t m = 0x5bd1e995;
	    const int r = 24;
	
	    /* Initialize the hash to a 'random' value */
	    uint32_t h = seed ^ len;
	
	    /* Mix 4 bytes at a time into the hash */
	    const unsigned char *data = (const unsigned char *)key;
	
	    while(len >= 4) {
	        uint32_t k = *(uint32_t*)data;
	
	        k *= m;
	        k ^= k >> r;
	        k *= m;
	
	        h *= m;
	        h ^= k;
	
	        data += 4;
	        len -= 4;
	    }
	
	    /* Handle the last few bytes of the input array  */
	    switch(len) {
	    case 3: h ^= data[2] << 16;
	    case 2: h ^= data[1] << 8;
	    case 1: h ^= data[0]; h *= m;
	    };
	
	    /* Do a few final mixes of the hash to ensure the last few
	     * bytes are well-incorporated. */
	    h ^= h >> 13;
	    h *= m;
	    h ^= h >> 15;
	
	    return (unsigned int)h;
	}
{% endhighlight %}

当字典被用作数据库的底层实现或者哈希键的底层实现时， `redis` 使用 `Murmurhash2` 算法来计算哈希的值。 `Murmurhash` 哈希算法是有 `Austin Appleby` 与 2008 年发明的，这种算法的优点在于，即使输入的键是有规律的，算法仍能给出一个很好的随机分布性，并且算法的计算速度也非常快。

	/* And a case insensitive hash function (based on djb hash) */
	// 这是一个比较简单的计算哈希值的方法，字符串哈希，就是不断乘以33
	unsigned int dictGenCaseHashFunction(const unsigned char *buf, int len) {
	    unsigned int hash = (unsigned int)dict_hash_function_seed;		//哈希方法的种子
	
	    while (len--)
	        hash = ((hash << 5) + hash) + (tolower(*buf++)); /* hash * 33 + c */
		//hash << 5 + hash = hash * 2^5 + hash = hash * 32 + hash = hash * 33
	    return hash;
	}

## 解决键冲突
当不同的 key 利用哈希算法得到相同的 hash 值时，哈希表时如何解决冲突问题的呢？通过前面的哈希节点的结构可以看到`dictEntry`结构是一个链表的节点，有一个指向 `dictEntry`节点的指针成员，确实，哈希表就是通过拉链法（或者链地址法）来解决冲突问题的，每一个哈希节点都有一个next指针，相同哈希索引的多个节点可以通过next指针相连构成一个单链表，这就解决了冲突问题。<br>
![hash list](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/redis_analysis/hash_list.png)

因为dictEntry 组成的单链表是没有尾节点的，每次插入一个节点都是从头部插入。

{% highlight ruby %}
	/* Low level add. This function adds the entry but instead of setting
	 * a value returns the dictEntry structure to the user, that will make
	 * sure to fill the value field as he wishes.
	 *
	 * This function is also directly exposed to the user API to be called
	 * mainly in order to store non-pointers inside the hash value, example:
	 *
	 * entry = dictAddRaw(dict,mykey);
	 * if (entry != NULL) dictSetSignedIntegerVal(entry,1000);
	 *
	 * Return values:
	 *
	 * If key already exists NULL is returned.
	 * If key was added, the hash entry is returned to be manipulated by the caller.
	 */
	/*如果key已经在哈希表中已经存在，返回NULL；否则返回该哈希节点*/
	dictEntry *dictAddRaw(dict *d, void *key)
	{
	    int index;
	    dictEntry *entry;
	    dictht *ht;
	
		/* 判断是否正在 rehash，正在 rehash 中的哈希表与没有 rehash 的哈希表插入操作不同
		 1. 当 rehash 时，需要将ht[0] 中的所有节点 rehash 到 ht[1]中，这时，为了保证ht[0]中的节点是一直减少的，插入时插入到ht[1]中
		 2. 没有 rehash ，直接将节点插入到 ht[0]中
		*/	
	    if (dictIsRehashing(d)) _dictRehashStep(d);	
	
	    /* Get the index of the new element, or -1 if
	     * the element already exists. */
	    if ((index = _dictKeyIndex(d, key)) == -1)
	        return NULL;
	
	    /* Allocate the memory and store the new entry */
	    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
	    entry = zmalloc(sizeof(*entry));
	    entry->next = ht->table[index];		//从链表头插入
	    ht->table[index] = entry;
	    ht->used++;
	
	    /* Set the hash entry fields. */
	    dictSetKey(d, entry, key);
	    return entry;
	}
{% endhighlight %}

从链表头插入，不需要单向遍历到链表尾部在插入节点，降低了插入节点时的时间复杂度 O(1)
## rehash
哈希表的一个重要的比例参数为_负载因子_(load factor)，它的计算公式为

	load_factor = used / size; //代码中的注释为 USED/BUCKETS ratio
	
随着操作的不断进行，哈希表中的节点会逐渐的增加或者减少，为了维持 load factor 在合适的范围之内，程序需要对哈希表进行扩展 (expand) 或者收缩，而这，是通过 rehash 来完成的。

>redis 对字典的哈希表进行 rehash 操作的步骤如下： <br>
>1) 为 ht[1] 分配空间，这个空间的大小取决于要执行的操作（扩展还是收缩），以及 ht[0] 当前包含的键值对的数量（ht[0].used）: <br>
>　　a. 如果是扩展操作，ht[1] 的大小为不小于 ht[0].used*2 的 2^n；<br>
>　　b. 如果是收缩操作，ht[1] 的大小为不小于 ht[0].used 的 2^n <br>
>2) 将 ht[0] 保存的所有键值对 rehash 到 ht[1] 中， rehash 是指重新计算哈希值和索引值，然后保存到 ht[1] 中； <br>
>3) 当 ht[0] 上的所有键值对都迁移到了 ht[1] 上之后（此时ht[0]是一张空表），释放 ht[0]，同时将 ht[1] 设置为 ht[0]，在为 ht[1] 创建一个空表哈希，为下一次 rehash 做准备。 <br>

**引用中的内容摘自 黄健宏的《Redis设计与实现》4.4节 rehash**

### 扩展
	/* Using dictEnableResize() / dictDisableResize() we make possible to
	 * enable/disable resizing of the hash table as needed. This is very important
	 * for Redis, as we use copy-on-write and don't want to move too much memory
	 * around when there is a child performing saving operations.
	 *
	 * Note that even when dict_can_resize is set to 0, not all resizes are
	 * prevented: a hash table is still allowed to grow if the ratio between
	 * the number of elements and the buckets > dict_force_resize_ratio. */
	static int dict_can_resize = 1;
	static unsigned int dict_force_resize_ratio = 5;
	
以上两个 static 变量说明的是允许哈希表进行扩展或者收缩的前提条件。

redis 使用了写时复制的原则（COW），打个比方，当父进程创建子进程时，这时，采用写时复制的原则，父子进程访问的数据都是同一份拷贝，只有当其中一个需要对数据进行修改时，才会将父进程的内容拷贝一份给子进程，这样能够极大的节约内存。所以，当 redis 创建服务器子进程时，通过写时复制原则节约内存，就希望在子进程存在期间，尽可能的不要对哈希表进行扩展操作，服务器通过增大负载因子来达到这一目的，从而最大限度的节约内存。

`dict_can_resize`设置为1时，表示哈希表可以扩展或者收缩，设置为0时，表示不能操作，但是，当负载因子超过阈值 `dict_forece_resize_ratio`时，会强制进行扩展操作。上文也介绍了 `_dictExpandIfNeeded`这个函数

{% highlight ruby %}
	/* Expand the hash table if needed */
	static int _dictExpandIfNeeded(dict *d)
	{
	    /* Incremental rehashing already in progress. Return. */
	    if (dictIsRehashing(d)) return DICT_OK;
	
	    /* If the hash table is empty expand it to the initial size. */
	    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
	
	    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
	     * table (global setting) or we should avoid it but the ratio between
	     * elements/buckets is over the "safe" threshold, we resize doubling
	     * the number of buckets. */
	    if (d->ht[0].used >= d->ht[0].size &&
	        (dict_can_resize ||
	         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
	    {
	        return dictExpand(d, d->ht[0].used*2);
	    }
	    return DICT_OK;
	}
{% endhighlight %}

从代码中可以看出：

- 当负载因子(load factor)达到 1 时，如果允许扩展(`dict_can_resize`设置为1)，就进行扩展操作
- 当负载因子超过安全阈值 `dict_force_resize_ratio`时，就会强制进行扩展

### 收缩
当哈希表节点数减少时，为了保证 load factor 在一个合理范围内，需要对哈希表进行收缩，当负载因子小于等于 1 时，就需要做这种操作。

	/* Resize the table to the minimal size that contains all the elements,
	 * but with the invariant of a USED/BUCKETS ratio near to <= 1 */
	int dictResize(dict *d)
	{
	    int minimal;
	
	    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
	    minimal = d->ht[0].used;
	    if (minimal < DICT_HT_INITIAL_SIZE)
	        minimal = DICT_HT_INITIAL_SIZE;
	    return dictExpand(d, minimal);
	}

### 哈希表调整大小的计算
哈希表进行扩展或者收缩，新的哈希大小是如何计算的呢

	/* Our hash table capability is a power of two */
	static unsigned long _dictNextPower(unsigned long size)
	{
	    unsigned long i = DICT_HT_INITIAL_SIZE;
	
	    if (size >= LONG_MAX) return LONG_MAX;
	    while(1) {
	        if (i >= size)
	            return i;
	        i *= 2;
	    }
	}
`DICT_HT_INITAL_SIZE`的值为4，即 2^2，所以每次设置的新哈希表的大小均为 2 的 n 次幂。

### 渐进式 rehash
哈希表 rehash 操作时将 ht[0] 中的所有键值对 rehash 到 ht[1] 中，但是这个动作并不是一次性、集中式的完成的，而是渐进式的、分多次完成的。因为当键值对数量很大时，集中式的 rehash 操作，需要庞大的计算量，可能会导致服务器在一段时间内停止服务。为了避免 rehash 对服务器性能造成影响，所以将 rehash 操作分多次执行，渐进式的操作。

1) 当需要 rehash 时，将 rehashidx 设置为0 <br>
2) 为 ht[1] 分配空间，此时字典同时拥有 ht[0] 和 ht[1] 两个哈希表 <br>
3) 在 rehash 期间，通过判断 rehashidx 是否等于 -1，每次对字典进程添加、删除、查找和更新操作时，出了执行指定操作之外，还会将 `ht[0].table[rehashidx]` 这个单链表上的所有键值对 rehash 到 ht[1] 中，操作完成后，将 rehashidx 加 1 <br>
4) 当 rehash 操作完成后，rehashidx 的值被设置为 -1

{% highlight ruby %}
	/* Performs N steps of incremental rehashing. Returns 1 if there are still
	 * keys to move from the old to the new hash table, otherwise 0 is returned.
	 *
	 * Note that a rehashing step consists in moving a bucket (that may have more
	 * than one key as we use chaining) from the old to the new hash table, however
	 * since part of the hash table may be composed of empty spaces, it is not
	 * guaranteed that this function will rehash even a single bucket, since it
	 * will visit at max N*10 empty buckets in total, otherwise the amount of
	 * work it does would be unbound and the function may block for a long time. */
	// 返回0表示 rehash 结束，返回1 表示还有键值对需要继续 rehash 
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
	            unsigned int h;
	
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
	        d->rehashidx++;		//操作完将 rehashidx 加1
	    }
	
	    /* Check if we already rehashed the whole table... */
	    if (d->ht[0].used == 0) {	//判断是否 rehash 操作完成
	        zfree(d->ht[0].table);
	        d->ht[0] = d->ht[1];
	        _dictReset(&d->ht[1]);
	        d->rehashidx = -1;
	        return 0;
	    }
	
	    /* More to rehash... */
	    return 1;
	}
{% endhighlight %}
## 哈希操作
### 查找操作
查找操作，当 rehashidx 的值不等于 -1 时，说明正在 rehash，那么进行一次rehash操作，在 `_dictRehashStep(d)`中调用 `dictRehash(d, 1)`执行一次。

查找时，在 ht[0] 和 ht[1] 中都需要查找

	dictEntry *dictFind(dict *d, const void *key)
	{
	    dictEntry *he;
	    unsigned int h, idx, table;
	
	    if (d->ht[0].size == 0) return NULL; /* We don't have a table at all */
	    if (dictIsRehashing(d)) _dictRehashStep(d);
	    h = dictHashKey(d, key);
	    for (table = 0; table <= 1; table++) {
	        idx = h & d->ht[table].sizemask;
	        he = d->ht[table].table[idx];
	        while(he) {
	            if (dictCompareKeys(d, key, he->key))
	                return he;
	            he = he->next;
	        }
	        if (!dictIsRehashing(d)) return NULL;
	    }
	    return NULL;
	}

### 添加
添加操作在上文中**解决键冲突**一节已经介绍了。
### 删除
删除操作，先在哈希表中查找指定 value 的键值对，然后删除。

{% highlight ruby %}
	int dictDelete(dict *ht, const void *key) {
	    return dictGenericDelete(ht,key,0);
	}
	
	/* Search and remove an element */
	static int dictGenericDelete(dict *d, const void *key, int nofree)
	{
	    unsigned int h, idx;
	    dictEntry *he, *prevHe;
	    int table;
	
	    if (d->ht[0].size == 0) return DICT_ERR; /* d->ht[0].table is NULL */
	    if (dictIsRehashing(d)) _dictRehashStep(d);
	    h = dictHashKey(d, key);	//获取哈希值
	
	    for (table = 0; table <= 1; table++) {
	        idx = h & d->ht[table].sizemask;	//计算索引值
	        he = d->ht[table].table[idx];
	        prevHe = NULL;
	        while(he) {
	            if (dictCompareKeys(d, key, he->key)) {	//找到了
	                /* Unlink the element from the list */
	                if (prevHe)	//从单链表中删除该节点
	                    prevHe->next = he->next;
	                else
	                    d->ht[table].table[idx] = he->next;
	                if (!nofree) {
	                    dictFreeKey(d, he);
	                    dictFreeVal(d, he);
	                }
	                zfree(he);
	                d->ht[table].used--;
	                return DICT_OK;
	            }
	            prevHe = he;
	            he = he->next;
	        }
	        if (!dictIsRehashing(d)) break;
	    }
	    return DICT_ERR; /* not found */
	}
{% endhighlight %}
### 更新
{% highlight ruby %}
	/* Add an element, discarding the old if the key already exists.
	 * Return 1 if the key was added from scratch, 0 if there was already an
	 * element with such key and dictReplace() just performed a value update
	 * operation. */
	int dictReplace(dict *d, void *key, void *val)
	{
	    dictEntry *entry, auxentry;
	
	    /* Try to add the element. If the key
	     * does not exists dictAdd will suceed. */
	    if (dictAdd(d, key, val) == DICT_OK)
	        return 1;
	    /* It already exists, get the entry */
	    entry = dictFind(d, key);
	    /* Set the new value and free the old one. Note that it is important
	     * to do that in this order, as the value may just be exactly the same
	     * as the previous one. In this context, think to reference counting,
	     * you want to increment (set), and then decrement (free), and not the
	     * reverse. */
	    auxentry = *entry;
	    dictSetVal(d, entry, val);
	    dictFreeVal(d, &auxentry);
	    return 0;
	}
{% endhighlight %}

**参考文献：** <br>
1. Redis设计与实现，黄健宏 <br>
2. 博客文章[http://blog.csdn.net/androidlushangderen/article/details/39860693]
