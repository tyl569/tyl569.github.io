---
title: PHP数组的存储
tags:
  - PHP
  - PHP7
  - PHP源码
id: '737'
categories:
  - - PHP
  - - PHP源码
date: 2020-10-25 19:54:17
---

PHP数组是PHP最复杂的数据结构，没有之一，如果能把数据彻底搞透，那么其他的数据结构也能理解的差不多了。

#### 数据结构

```c

typedef struct _Bucket {
    zval              val;
    zend_ulong        h;                /* hash value (or numeric index)   */
    zend_string      *key;              /* string key or NULL for numerics */
} Bucket;

typedef struct _zend_array HashTable;

struct _zend_array {
    zend_refcounted_h gc;
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    flags,
                zend_uchar    nApplyCount,
                zend_uchar    nIteratorsCount,
                zend_uchar    consistency)
        } v;
        uint32_t flags;
    } u;
    uint32_t          nTableMask; // 中间映射表，用来映射在arData的存储位置
    Bucket           *arData; // 元素的存储数据结构，默认指向第一个元素
    uint32_t          nNumUsed; // arData使用的个数
    uint32_t          nNumOfElements; // 数组的个数
    uint32_t          nTableSize; // 数组的大小
    uint32_t          nInternalPointer;
    zend_long         nNextFreeElement;
    dtor_func_t       pDestructor;
};
```

#### 数组的初始化

```c
ZEND_API void ZEND_FASTCALL _zend_hash_init(HashTable *ht, uint32_t nSize, dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
{
    GC_REFCOUNT(ht) = 1;
    GC_TYPE_INFO(ht) = IS_ARRAY;
    ht->u.flags = (persistent ? HASH_FLAG_PERSISTENT : 0)  HASH_FLAG_APPLY_PROTECTION  HASH_FLAG_STATIC_KEYS;
    ht->nTableMask = HT_MIN_MASK;
    HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
    ht->nNumUsed = 0;
    ht->nNumOfElements = 0;
    ht->nInternalPointer = HT_INVALID_IDX;
    ht->nNextFreeElement = 0;
    ht->pDestructor = pDestructor;
    ht->nTableSize = zend_hash_check_size(nSize);
}

static zend_always_inline void zend_hash_real_init_ex(HashTable *ht, int packed)
{
    HT_ASSERT(GC_REFCOUNT(ht) == 1);
    ZEND_ASSERT(!((ht)->u.flags & HASH_FLAG_INITIALIZED));
    if (packed) {
        HT_SET_DATA_ADDR(ht, pemalloc(HT_SIZE(ht), (ht)->u.flags & HASH_FLAG_PERSISTENT));
        (ht)->u.flags = HASH_FLAG_INITIALIZED  HASH_FLAG_PACKED;
        HT_HASH_RESET_PACKED(ht);
    } else {
        (ht)->nTableMask = -(ht)->nTableSize;
        HT_SET_DATA_ADDR(ht, pemalloc(HT_SIZE(ht), (ht)->u.flags & HASH_FLAG_PERSISTENT));
        (ht)->u.flags = HASH_FLAG_INITIALIZED;
        if (EXPECTED(ht->nTableMask == (uint32_t)-8)) {
            Bucket *arData = ht->arData;

            HT_HASH_EX(arData, -8) = -1;
            HT_HASH_EX(arData, -7) = -1;
            HT_HASH_EX(arData, -6) = -1;
            HT_HASH_EX(arData, -5) = -1;
            HT_HASH_EX(arData, -4) = -1;
            HT_HASH_EX(arData, -3) = -1;
            HT_HASH_EX(arData, -2) = -1;
            HT_HASH_EX(arData, -1) = -1;
        } else {
            HT_HASH_RESET(ht);
        }
    }
}
```

在初始化之前，会先调用\_zend\_hash\_init进行一些简单的初始化操作，但是这部分做的事情比较少，只是初始化了nTableSize=8，arData的内存大小是根据这个值确定的，它的大小是8的幂次方，最小为8；

其他的初始化操作，是通过zend\_hash\_real\_init\_ex函数进行的：

*   首先，将u.flags设置为`初始化`的状态
*   将映射关系nTableMask设置为nTableSize大小的相反数
*   申请地址空间，设备的地址空间大小是nTableMask和nTableSize的两个地址空间大小
*   设置arData的每个元素的值为-1

#### 数组的赋值\_zend\_hash\_add\_or\_update\_i

```c
static zend_always_inline zval *_zend_hash_add_or_update_i(HashTable *ht, zend_string *key, zval *pData, uint32_t flag ZEND_FILE_LINE_DC)
{
    zend_ulong h;
    uint32_t nIndex;
    uint32_t idx;
    Bucket *p;

    IS_CONSISTENT(ht);
    HT_ASSERT(GC_REFCOUNT(ht) == 1);

    if (UNEXPECTED(!(ht->u.flags & HASH_FLAG_INITIALIZED))) {
        CHECK_INIT(ht, 0);
        goto add_to_hash;
    } else if (ht->u.flags & HASH_FLAG_PACKED) {
        zend_hash_packed_to_hash(ht);
    } else if ((flag & HASH_ADD_NEW) == 0) {
        p = zend_hash_find_bucket(ht, key);

        if (p) {
            zval *data;

            if (flag & HASH_ADD) {
                if (!(flag & HASH_UPDATE_INDIRECT)) {
                    return NULL;
                }
                ZEND_ASSERT(&p->val != pData);
                data = &p->val;
                if (Z_TYPE_P(data) == IS_INDIRECT) {
                    data = Z_INDIRECT_P(data);
                    if (Z_TYPE_P(data) != IS_UNDEF) {
                        return NULL;
                    }
                } else {
                    return NULL;
                }
            } else {
                ZEND_ASSERT(&p->val != pData);
                data = &p->val;
                if ((flag & HASH_UPDATE_INDIRECT) && Z_TYPE_P(data) == IS_INDIRECT) {
                    data = Z_INDIRECT_P(data);
                }
            }
            if (ht->pDestructor) {
                ht->pDestructor(data);
            }
            ZVAL_COPY_VALUE(data, pData);
            return data;
        }
    }

    ZEND_HASH_IF_FULL_DO_RESIZE(ht);        /* If the Hash table is full, resize it */

add_to_hash:
    idx = ht->nNumUsed++;
    ht->nNumOfElements++;
    if (ht->nInternalPointer == HT_INVALID_IDX) {
        ht->nInternalPointer = idx;
    }
    zend_hash_iterators_update(ht, HT_INVALID_IDX, idx);
    p = ht->arData + idx;
    p->key = key;
    if (!ZSTR_IS_INTERNED(key)) {
        zend_string_addref(key);
        ht->u.flags &= ~HASH_FLAG_STATIC_KEYS;
        zend_string_hash_val(key);
    }
    p->h = h = ZSTR_H(key);
    ZVAL_COPY_VALUE(&p->val, pData);
    nIndex = h  ht->nTableMask;
    Z_NEXT(p->val) = HT_HASH(ht, nIndex);
    HT_HASH(ht, nIndex) = HT_IDX_TO_HASH(idx);

    return &p->val;
}
```

*   首先，在赋值之前，会先判断数组是否已经被初始化过，如果初始化那么就进行赋值的操作
*   赋值的时候，会把nNumUsed进行累加，然后得到arData的位置
*   将赋值的bucket设置hash code和key等成员变量
*   根据hash code，计算出在nTableMask对应的位置为nIndex
*   将arData的位置，存储到nTableMask对应的nIndex的值上面

![](/uploads/2020/01/绘图1.png)

本文链接: [https://feilong.tech/2020/10/25/php-array](https://feilong.tech/2020/10/25/php-array)