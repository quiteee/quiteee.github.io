# go 堆内存管理

Go 内存分配是基于 TCMalloc 算法实现的，TCMalloc 具体介绍可以看这里[Go内存-TCMalloc](https://quiteee.github.io/go/tcmalloc)

## 基本概念

### mheap

Go 在程序运行之初，会向操作系统申请一大块（虚拟）内存，这块内存被称为 **mheap**。

Go 的堆内存管理是基于 **mheap** 进行的。

**mheap** 大致分为三个区域
* **Spans**

    存放指针，指向 Arena 里的 mspan。
    
* **Bitmap**

    存储 mspan 中对象的标记信息，例如存储内容是否含有指针、是否被GC标记等。

* **Arena**

    Arena 由页(Page)组成，每个 Page 8KB，可以认为是真正的“堆内存”。
    

### mspan

**mspan** 由 arena 中一个或多个连续的 Page 构成，是内存管理的基本单元。

**mspan** 是一个双向链表，内部包含起始地址、Page数量等。

```
// runtime/mheap.go

type mspan struct {
	next *mspan     // next span in list, or nil if none
	prev *mspan     // previous span in list, or nil if none
	
	startAddr uintptr // address of first byte of span aka s.base()
	npages    uintptr // number of pages in span
	... 
}
```

**mspan** 有多种规格，如下所示。

```
// runtime/sizeclasses.go
//
// class  bytes/obj  bytes/span  objects  tail waste  max waste
//     1          8        8192     1024           0     87.50%
//     2         16        8192      512           0     43.75%
//     3         32        8192      256           0     46.88%
//     4         48        8192      170          32     31.52%
//     5         64        8192      128           0     23.44%
//     6         80        8192      102          32     19.07%
//     7         96        8192       85          32     15.95%
//     8        112        8192       73          16     13.56%
//     9        128        8192       64           0     11.72%
//    10        144        8192       56         128     11.82%
//    11        160        8192       51          32      9.73%
//    12        176        8192       46          96      9.59%
//    13        192        8192       42         128      9.25%
//    14        208        8192       39          80      8.12%
//    15        224        8192       36         128      8.15%
//    16        240        8192       34          32      6.62%
//    17        256        8192       32           0      5.86%
//    18        288        8192       28         128     12.16%
//    19        320        8192       25         192     11.80%
//    20        352        8192       23          96      9.88%
//    21        384        8192       21         128      9.51%
//    22        416        8192       19         288     10.71%
//    23        448        8192       18         128      8.37%
//    24        480        8192       17          32      6.82%
//    25        512        8192       16           0      6.05%
//    26        576        8192       14         128     12.33%
//    27        640        8192       12         512     15.48%
//    28        704        8192       11         448     13.93%
//    29        768        8192       10         512     13.94%
//    30        896        8192        9         128     15.52%
//    31       1024        8192        8           0     12.40%
//    32       1152        8192        7         128     12.41%
//    33       1280        8192        6         512     15.55%
//    34       1408       16384       11         896     14.00%
//    35       1536        8192        5         512     14.00%
//    36       1792       16384        9         256     15.57%
//    37       2048        8192        4           0     12.45%
//    38       2304       16384        7         256     12.46%
//    39       2688        8192        3         128     15.59%
//    40       3072       24576        8           0     12.47%
//    41       3200       16384        5         384      6.22%
//    42       3456       24576        7         384      8.83%
//    43       4096        8192        2           0     15.60%
//    44       4864       24576        5         256     16.65%
//    45       5376       16384        3         256     10.92%
//    46       6144       24576        4           0     12.48%
//    47       6528       32768        5         128      6.23%
//    48       6784       40960        6         256      4.36%
//    49       6912       49152        7         768      3.37%
//    50       8192        8192        1           0     15.61%
//    51       9472       57344        6         512     14.28%
//    52       9728       49152        5         512      3.64%
//    53      10240       40960        4           0      4.99%
//    54      10880       32768        3         128      6.24%
//    55      12288       24576        2           0     11.45%
//    56      13568       40960        3         256      9.99%
//    57      14336       57344        4           0      5.35%
//    58      16384       16384        1           0     12.49%
//    59      18432       73728        4           0     11.11%
//    60      19072       57344        3         128      3.57%
//    61      20480       40960        2           0      6.87%
//    62      21760       65536        3         256      6.25%
//    63      24576       24576        1           0     11.45%
//    64      27264       81920        3         128     10.00%
//    65      28672       57344        2           0      4.91%
//    66      32768       32768        1           0     12.50%
```

**bytes/obj** 表示 mspan 内部的 object 的 size，代码内通过 ***class_to_size*** 数组映射

```
bytes_obj = class_to_size[class]
```

**bytes/span** 表示 mspan 的 size，都是 8KB (Page) 的倍数，代码内通过 ***class_to_allocnpages*** 数组映射对应的 Page 数量。

```
bytes_span = class_to_allocnpages[class] * pageSize
```

**objects** 表示 mspan 内部可以存放多少 object

```
objects = bytes_span / bytes_obj
```

**tail waste** 表示 存放所有的 object 后，至少剩余的空间。

```
tail_waste = bytes_span - objects * bytes_obj
```

**max waste** 表示 存放所有的 object 后，最多剩余的空间。因为 object 不会把整个 size 填满，例如 9KB 大小的 object，会分配 class 2 对应的 16KB，就会剩余 7KB。

```
max_waste = (bytes_span - pre_bytes_span - 1) * objects + tail_waste / bytes_span
// pre_bytes_span 是前一个 class 的 bytes_span
```

class 0 是特殊的，所有大于 32KB 的对象会分配到 class 0


## 内存分配组件

Go 整体将内存分配分为三层。分别是 **mcache** **mcentral** **mheap**

### mcache

为每个工作线程（在GMP调度模型中对应Processor）分配的私有内存。

通过 **mcache** 分配无需上锁

代码里的 **mcache** 定义：

```
// runtime/mcache.go
type mcache struct {
    ... // 省略了部分内容。
    alloc [numSpanClasses]*mspan // numSpanClasses = _NumSizeClasses(67) << 1
}
```

**mcache** 内部是 class 对应的 mspan 链表。

每种规格的 **mspan** 有两列，分别分配给 *内部有指针的对象* 和 *内部无指针的对象*，这是为了方便 GC。


### mcentral

**mcentral** 为 **mcache** 提供切分好的 **mspan**。当 **mcache** 空间不足时，会向 **mcentral** 申请资源。

**mcentral** 是所有线程共享的，需要加锁访问。

```
// runtime/mcentral.go

type mcentral struct {
	lock      mutex
	spanclass spanClass
	nonempty  mSpanList // list of spans with a free object, ie a nonempty free list
	empty     mSpanList // list of spans with no free objects (or cached in an mcache)
    ...
}
```

如上所示，**mcentral** 内部包含一个互斥锁。

此外，**mcentral** 包含两个 span 链表。

* nonempty: 有空闲对象（nonempty表示空闲对象not empty），当 mcache 向 mcentral 申请空间时，会从中获取 span，并将其移入 mcentral.empty
* empty: 无空闲对象，表示已经被使用了。span 被回收时，移入 mcentral.nonempty

### mheap

当 **mcentral** 资源不足，会向 **mheap** 申请。

如果 **mheap** 资源不足，会向 **OS** 申请。

大对象(>32KB) 也会直接从 **mheap** 上分配。

具体构成在上面的基本概念里已经介绍过了


## 对象分配

总共分成三种对象

* tiny 对象 (<16B)，用 mcache 的 tiny 分配器，会在同一个 object 上分配多个 tiny 对象。
* 小对象 (>=16B && <=32KB)，用 mcache 中，合适 class 的 mspan 分割成的 object 分配。
* 大对象 (>32KB)，直接用 mheap 分配。

## 参考
1. https://www.cnblogs.com/zpcoding/p/13259943.html#_label1_2
2. https://www.jianshu.com/p/7405b4e11ee2

## END
