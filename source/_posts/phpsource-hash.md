---
title: PHP源码分析之哈希表
tags: []
id: '178'
categories:
  - - PHP
  - - PHP源码
comments: false
date: 2017-09-20 20:04:15
---

PHP中使用最频繁的莫过于字符串和数组了，然而PHP中的数组主要是基于哈希表，这篇文章主要是分析一下PHP源码的内部哈希表的实现方式。PHP版本基于PHP 5.3。

<!-- more -->

## 基本概念

哈希表在实践中使用非常的广泛，哈希表的优势在于查询的时间复杂度是O(1), 哈希表提供增删改查等操作，这些操作在最坏的情况下就是和链表的性能O(n)一样。

哈希表主要有一下组成：

*   键（key）：用于操作数据的标示，例如PHP数组中的索引，或者字符串键等等。
*   槽(slot/bucket)：哈希表中用于保存数据的一个单元，也就是数据真正存放的容器。
*   哈希函数(hash function)：将key映射(map)到数据应该存放的slot所在位置的函数。
*   哈希冲突(hash collision)：哈希函数将两个不同的key映射到同一个索引的情况。

哈希表可以理解为数组的拓展，哈希表使用的是键的方式，然后通过哈希函数映射到一个索引，这个索引可以理解称是这个值得实际的存储位置。

```
hash（key）-> index
```

通过合理设计的哈希函数，我们就能将key映射到合适的范围，因为我们的key空间可以很大(例如字符串key)， 在映射到一个较小的空间中时可能会出现两个不同的key映射被到同一个index上的情况， 这就是我们所说的出现了冲突。 目前解决hash冲突的方法主要有两种：链接法和开放寻址法。

## 哈希冲突

### 链接法

链接法通常是通过一个链表保存bucket值的方式来解决冲突，链接法的最坏情况就是所有的key都映射到一个槽位，这样就使哈希表成了一个链表，然后在查询的时候，时间复杂度就成了O（n）。所以选择一哈希函数非常重要，最好能够使哈希值的散列度大一些，分布均匀一些。

由于目前大部分的编程语言的哈希表实现都是开源的，大部分语言的哈希算法都是公开的算法， 虽然目前的哈希算法都能良好的将key进行比较均匀的分布，而这个假使的前提是key是随机的，正是由于算法的确定性， 这就导致了别有用心的黑客能利用已知算法的可确定性来构造一些特殊的key，让这些key都映射到 同一个槽位导致哈希表退化成单链表，导致程序的性能急剧下降，从而造成一些应用的吞吐能力急剧下降， 尤其是对于高并发的应用影响很大，通过大量类似的请求可以让服务器遭受DoS(服务拒绝攻击)， 这个问题一直就存在着，只是最近才被各个语言重视起来。

哈希冲突攻击利用的哈希表最根本的弱点是：**开源算法和哈希实现的确定性以及可预测性**，这样攻击者才可以利用特殊构造的key来进行攻击。要解决这个问题的方法则是让攻击者无法轻易构造 能够进行攻击的key序列。

> **NOTE** 在笔者编写这节内容的时候PHP语言也采取了相应的措施来防止这类的攻击，PHP采用的是一种 治标不治本的做法: [限制用户提交数据字段数量](http://cn2.php.net/manual/en/info.configuration.php#ini.max-input-vars) 这样可以避免大部分的攻击，不过应用程序通常会有很多的数据输入方式，比如，SOAP，REST等等， 比如很多应用都会接受用户传入的JSON字符串，在执行json\_decode()的时候也可能会遭受攻击。 所以最根本的解决方法是让哈希表的碰撞key序列无法轻易的构造，目前PHP中还没有引入不增加额外的复杂性情况下的完美解决方案。

目前PHP中HashTable的哈希冲突解决方法就是链接法。

### 开放寻址法

通常还有另外一种解决冲突的方法：开放寻址法。使用开放寻址法是槽本身直接存放数据， 在插入数据时如果key所映射到的索引已经有数据了，这说明发生了冲突，这是会寻找下一个槽， 如果该槽也被占用了则继续寻找下一个槽，直到寻找到没有被占用的槽，在查找时也使用同样的策略来进行。

由于开放寻址法处理冲突的时候占用的是其他槽位的空间,这可能会导致后续的key在插入的时候更加容易出现 哈希冲突，所以采用开放寻址法的哈希表的装载因子不能太高，否则容易出现性能下降。

> **NOTE** _装载因子_是哈希表保存的元素数量和哈希表容量的比，通常采用链接法解决冲突的哈希表的装载 因子最好不要大于1，而采用开放寻址法的哈希表最好不要大于0.5。

## PHP的实现

PHP中的哈希表实现在Zend/zend\_hash.c中，还是按照上一小节的方式，先看看PHP实现中的数据结构， PHP使用如下两个数据结构来实现哈希表，HashTable结构体用于保存整个哈希表需要的基本信息， 而Bucket结构体用于保存具体的数据内容，如下：

```c
typedef struct _hashtable { 
    uint nTableSize;        // hash Bucket的大小，最小为8，以2x增长。
    uint nTableMask;        // nTableSize-1 ， 索引取值的优化
    uint nNumOfElements;    // hash Bucket中当前存在的元素个数，count()函数会直接返回此值 
    ulong nNextFreeElement; // 下一个数字索引的位置
    Bucket *pInternalPointer;   // 当前遍历的指针（foreach比for快的原因之一）
    Bucket *pListHead;          // 存储数组头元素指针
    Bucket *pListTail;          // 存储数组尾元素指针
    Bucket **arBuckets;         // 存储hash数组
    dtor_func_t pDestructor;    // 在删除元素时执行的回调函数，用于资源的释放
    zend_bool persistent;       //指出了Bucket内存分配的方式。如果persisient为TRUE，则使用操作系统本身的内存分配函数为Bucket分配内存，否则使用PHP的内存分配函数。
    unsigned char nApplyCount; // 标记当前hash Bucket被递归访问的次数（防止多次递归）
    zend_bool bApplyProtection;// 标记当前hash桶允许不允许多次访问，不允许时，最多只能递归3次
#if ZEND_DEBUG
    int inconsistent;
#endif
} HashTable;
```

nTableSize字段用于标示哈希表的容量，哈希表的初始容量最小为8。首先看看哈希表的初始化函数:

```c
ZEND_API int _zend_hash_init(HashTable *ht, uint nSize, hash_func_t pHashFunction,
                    dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
{
    uint i = 3;
    //...
    if (nSize >= 0x80000000) {
        /* prevent overflow */
        ht->nTableSize = 0x80000000;
    } else {
        while ((1U << i) < nSize) {
            i++;
        }
        ht->nTableSize = 1 << i;
    }
    // ...
    ht->nTableMask = ht->nTableSize - 1;

    /* Uses ecalloc() so that Bucket* == NULL */
    if (persistent) {
        tmp = (Bucket **) calloc(ht->nTableSize, sizeof(Bucket *));
        if (!tmp) {
            return FAILURE;
        }
        ht->arBuckets = tmp;
    } else {
        tmp = (Bucket **) ecalloc_rel(ht->nTableSize, sizeof(Bucket *));
        if (tmp) {
            ht->arBuckets = tmp;
        }
    }

    return SUCCESS;
}
```

例如如果设置初始大小为10，则上面的算法将会将大小调整为16。也就是始终将大小调整为接近初始大小的 2的整数次方。

为什么会做这样的调整呢？我们先看看HashTable将哈希值映射到槽位的方法，上一小节我们使用了取模的方式来将哈希值 映射到槽位，例如大小为8的哈希表，哈希值为100， 则映射的槽位索引为: 100 % 8 = 4，由于索引通常从0开始， 所以槽位的索引值为3，在PHP中使用如下的方式计算索引：

```c
h = zend_inline_hash_func(arKey, nKeyLength);
nIndex = h & ht->nTableMask;
```

从上面的\_zend\_hash\_init()函数中可知，ht->nTableMask的大小为ht->nTableSize -1。 这里使用&操作而不是使用取模，这是因为是相对来说取模操作的消耗和按位与的操作大很多。

> **NOTE** mask的作用就是将哈希值映射到槽位所能存储的索引范围内。 例如：某个key的索引值是21， 哈希表的大小为8，则mask为7，则求与时的二进制表示为： 10101 & 111 = 101 也就是十进制的5。 因为2的整数次方-1的二进制比较特殊：后面N位的值都是1，这样比较容易能将值进行映射， 如果是普通数字进行了二进制与之后会影响哈希值的结果。那么哈希函数计算的值的平均分布就可能出现影响。

设置好哈希表大小之后就需要为哈希表申请存储数据的空间了，如上面初始化的代码， 根据是否需要持久保存而调用了不同的内存申请方法。如前面PHP生命周期里介绍的,是否需要持久保存体现在：持久内容能在多个请求之间访问，而非持久存储是会在请求结束时释放占用的空间。 具体内容将在内存管理章节中进行介绍。

HashTable中的nNumOfElements字段很好理解，每插入一个元素或者unset删掉元素时会更新这个字段。 这样在进行count()函数统计数组元素个数时就能快速的返回。

nNextFreeElement字段非常有用。先看一段PHP代码：

```php
<?php
$a = array(10 => 'Hello');
$a[] = 'TIPI';
var_dump($a);

// ouput
array(2) {
  [10]=>
  string(5) "Hello"
  [11]=>
  string(5) "TIPI"
}
```

PHP中可以不指定索引值向数组中添加元素，这时将默认使用数字作为索引， 和[C语言中的枚举](http://en.wikipedia.org/wiki/Enumerated_type)类似， 而这个元素的索引到底是多少就由nNextFreeElement字段决定了。 如果数组中存在了数字key，则会默认使用最新使用的key + 1，例如上例中已经存在了10作为key的元素， 这样新插入的默认索引就为11了。

#### 数据容器：槽位

下面看看保存哈希表数据的槽位数据结构体：

```c
typedef struct bucket {
    ulong h;            // 对char *key进行hash后的值，或者是用户指定的数字索引值
    uint nKeyLength;    // hash关键字的长度，如果数组索引为数字，此值为0
    void *pData;        // 指向value，一般是用户数据的副本，如果是指针数据，则指向pDataPtr
    void *pDataPtr;     //如果是指针数据，此值会指向真正的value，同时上面pData会指向此值
    struct bucket *pListNext;   // 整个hash表的下一元素
    struct bucket *pListLast;   // 整个哈希表该元素的上一个元素
    struct bucket *pNext;       // 存放在同一个hash Bucket内的下一个元素
    struct bucket *pLast;       // 同一个哈希bucket的上一个元素
    // 保存当前值所对于的key字符串，这个字段只能定义在最后，实现变长结构体
    char arKey[1];              
} Bucket;
```

如上面各字段的注释。h字段保存哈希表key哈希后的值。这里保存的哈希值而不是在哈希表中的索引值， 这是因为索引值和哈希表的容量有直接关系，如果哈希表扩容了，那么这些索引还得重新进行哈希在进行索引映射， 这也是一种优化手段。 在PHP中可以使用字符串或者数字作为数组的索引。 数字索引直接就可以作为哈希表的索引，数字也无需进行哈希处理。h字段后面的nKeyLength字段是作为key长度的标示， 如果索引是数字的话，则nKeyLength为0。在PHP数组中如果索引字符串可以被转换成数字也会被转换成数字索引。 **所以在PHP中例如'10'，'11'这类的字符索引和数字索引10， 11没有区别。**

上面结构体的最后一个字段用来保存key的字符串，而这个字段却申明为只有一个字符的数组， 其实这里是一种长见的[变长结构体](http://stackoverflow.com/a/4690976/319672)，主要的目的是增加灵活性。 以下为哈希表插入新元素时申请空间的代码

```c
p = (Bucket *) pemalloc(sizeof(Bucket) - 1 + nKeyLength, ht->persistent);
if (!p) {
    return FAILURE;
}
```

如代码，申请的空间大小加上了字符串key的长度，然后把key拷贝到新申请的空间里。 在后面比如需要进行hash查找的时候就需要对比key这样就可以通过对比p->arKey和查找的key是否一样来进行数据的 查找。申请空间的大小-1是因为结构体内本身的那个字节还是可以使用的。

> **NOTE** 在PHP5.4中将这个字段定义成const char\* arKey类型了。

![哈希表](/uploads/2017/09/03-01-02-zend_hashtable.png)memcpy(p->arKey, arKey, nKeyLength);

上图来源于[网络](http://gsm56.com/?p=124)。

*   Bucket结构体维护了两个双向链表，pNext和pLast指针分别指向本槽位所在的链表的关系。
*   而pListNext和pListLast指针指向的则是整个哈希表所有的数据之间的链接关系。 HashTable结构体中的pListHead和pListTail则维护整个哈希表的头元素指针和最后一个元素的指针。

> **NOTE** PHP中数组的操作函数非常多，例如：array\_shift()和array\_pop()函数，分别从数组的头部和尾部弹出元素。 哈希表中保存了头部和尾部指针，这样在执行这些操作时就能在常数时间内找到目标。 PHP中还有一些使用的相对不那么多的数组操作函数：next()，prev()等的循环中， 哈希表的另外一个指针就能发挥作用了：pInternalPointer，这个用于保存当前哈希表内部的指针。 这在循环时就非常有用。

如图中左下角的假设，假设依次插入了Bucket1，Bucket2，Bucket3三个元素：

1.  插入Bucket1时，哈希表为空，经过哈希后定位到索引为1的槽位。此时的1槽位只有一个元素Bucket1。 其中Bucket1的pData或者pDataPtr指向的是Bucket1所存储的数据。此时由于没有链接关系。pNext， pLast，pListNext，pListLast指针均为空。同时在HashTable结构体中也保存了整个哈希表的第一个元素指针， 和最后一个元素指针，此时HashTable的pListHead和pListTail指针均指向Bucket1。
2.  插入Bucket2时，由于Bucket2的key和Bucket1的key出现冲突，此时将Bucket2放在双链表的前面。 由于Bucket2后插入并置于链表的前端，此时Bucket2.pNext指向Bucket1，由于Bucket2后插入。 Bucket1.pListNext指向Bucket2，这时Bucket2就是哈希表的最后一个元素，这是HashTable.pListTail指向Bucket2。
3.  插入Bucket3，该key没有哈希到槽位1，这时Bucket2.pListNext指向Bucket3，因为Bucket3后插入。 同时HashTable.pListTail改为指向Bucket3。

简单来说就是哈希表的Bucket结构维护了哈希表中插入元素的先后顺序，哈希表结构维护了整个哈希表的头和尾。 在操作哈希表的过程中始终保持预算之间的关系。

### 哈希表的操作接口

和上一节类似，将简单介绍PHP哈希表的操作接口实现。提供了如下几类操作接口：

*   初始化操作，例如zend\_hash\_init()函数，用于初始化哈希表接口，分配空间等。
*   查找，插入，删除和更新操作接口，这是比较常规的操作。
*   迭代和循环，这类的接口用于循环对哈希表进行操作。
*   复制，排序，倒置和销毁等操作。

本小节选取其中的插入操作进行介绍。 在PHP中不管是对数组的添加操作（zend\_hash\_add），还是对数组的更新操作（zend\_hash\_update）， 其最终都是调用\_zend\_hash\_add\_or\_update函数完成，这在面向对象编程中相当于两个公有方法和一个公共的私有方法的结构， 以实现一定程度上的代码复用。

```c
ZEND_API int _zend_hash_add_or_update(HashTable *ht, const char *arKey, uint nKeyLength, void *pData, uint nDataSize, void **pDest, int flag ZEND_FILE_LINE_DC)
{
     //...省略变量初始化和nKeyLength <=0 的异常处理

    h = zend_inline_hash_func(arKey, nKeyLength);
    nIndex = h & ht->nTableMask;

    p = ht->arBuckets[nIndex];
    while (p != NULL) {
        if ((p->h == h) && (p->nKeyLength == nKeyLength)) {
            if (!memcmp(p->arKey, arKey, nKeyLength)) { //  更新操作
                if (flag & HASH_ADD) {
                    return FAILURE;
                }
                HANDLE_BLOCK_INTERRUPTIONS();

                //..省略debug输出
                if (ht->pDestructor) {
                    ht->pDestructor(p->pData);
                }
                UPDATE_DATA(ht, p, pData, nDataSize);
                if (pDest) {
                    *pDest = p->pData;
                }
                HANDLE_UNBLOCK_INTERRUPTIONS();
                return SUCCESS;
            }
        }
        p = p->pNext;
    }

    p = (Bucket *) pemalloc(sizeof(Bucket) - 1 + nKeyLength, ht->persistent);
    if (!p) {
        return FAILURE;
    }
    memcpy(p->arKey, arKey, nKeyLength);
    p->nKeyLength = nKeyLength;
    INIT_DATA(ht, p, pData, nDataSize);
    p->h = h;
    CONNECT_TO_BUCKET_DLLIST(p, ht->arBuckets[nIndex]); //Bucket双向链表操作
    if (pDest) {
        *pDest = p->pData;
    }

    HANDLE_BLOCK_INTERRUPTIONS();
    CONNECT_TO_GLOBAL_DLLIST(p, ht);    // 将新的Bucket元素添加到数组的链接表的最后面
    ht->arBuckets[nIndex] = p;
    HANDLE_UNBLOCK_INTERRUPTIONS();

    ht->nNumOfElements++;
    ZEND_HASH_IF_FULL_DO_RESIZE(ht);        /*  如果此时数组的容量满了，则对其进行扩容。*/
    return SUCCESS;
}
```

整个写入或更新的操作流程如下：

1.  生成hash值，通过与nTableMask执行与操作，获取在arBuckets数组中的Bucket。
2.  如果Bucket中已经存在元素，则遍历整个Bucket，查找是否存在相同的key值元素，如果有并且是update调用，则执行update数据操作。
3.  创建新的Bucket元素，初始化数据，并将新元素添加到当前hash值对应的Bucket链表的最前面（CONNECT\_TO\_BUCKET\_DLLIST）。
4.  将新的Bucket元素添加到数组的链接表的最后面（CONNECT\_TO\_GLOBAL\_DLLIST）。
5.  将元素个数加1，如果此时数组的容量满了，则对其进行扩容。这里的判断是依据nNumOfElements和nTableSize的大小。 如果nNumOfElements > nTableSize则会调用zend\_hash\_do\_resize以2X的方式扩容（nTableSize << 1）。

## 参考文献

*   《深入理解PHP内核》