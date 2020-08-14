# C++ redis源码阅读之简单动态字符串sds(simple dynamic strings)

## 0.导语
因为我也没使用过redis,所以对它的特性也并不了解,因此这个剖析是直接看源码外加网络资料查询写成,再加上水平有限,错误和疏漏之处在所难免,欢迎大家的指正。

sds是redis的基础数据结构之一,根据找到的信息来看，它有以下几个特性

- 与传统的c字符串兼容

- 可动态扩展，字符串内容可修改也可以追加

- 二进制安全，可以存储任意的二进制数据，而不局限于字符

那么为了看看它是如何实现这些功能的，下面开始来分析一下代码

## 1.基本保存结构

~~redis提供了5种类型的sds,由5种不同结构体实现,分别用来存放不同大小的字符串。~~

根据后面的代码来看，这个理解并不对，应该说是提供了5种不同的header，sdshdr应该是simple dynamic strings header的意思。

分别是sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64。其中的sdshdr5是不使用的。

这几个结构体的基本结构体如下

```
//sdshdr不使用
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; 
    char buf[];
};
```

```
struct __attribute__ ((__packed__)) sdshdr8 {
    //uint8_t : unsigned char
    //sizeof(uint8_t) == 1
    uint8_t len; 
    uint8_t alloc; 
    unsigned char flags; 
    char buf[];
    // sizeof(sdshdr8) == 3
};
```
```
struct __attribute__ ((__packed__)) sdshdr16 {
	//uint16_t : unsigned short int
	//sizeof(uint16_t) == 2
    uint16_t len; 
    uint16_t alloc; 
    unsigned char flags; 
    char buf[];
    /*
    本来按照内存对齐规则,sdshdr16本来应该占5字节
    但因为使用了__attribute__ ((__packed__))声明,取消了内存对齐,所以这时候占用字节为
    sizeof(sdshdr16) == 5
    */
};
```

```
struct __attribute__ ((__packed__)) sdshdr32 {
    //uint32_t : unsigned int
    //sizeof(uint32_t) == 4
    uint32_t len; 
    uint32_t alloc; 
    unsigned char flags; 
    char buf[];
    //sizeof(sdshdr32) == 9
};
```

```
struct __attribute__ ((__packed__)) sdshdr64 {
    //uint64_t : unsigned long int
    //sizeof(uint64_t) == 8

    //buf中已占用空间的长度
    uint64_t len; 

    //注释说是去掉了header和空终止符(猜测是c字符串末尾的'\0')的部分
    uint64_t alloc; 

    //表示该字符串的类型(sdshdr5, sdshdr8, sdshdr16, sdshdr32, sdshdr64)
    unsigned char flags; 

    char buf[];
    //sizeof(sdshdr64) == 17
};
```

紧接着结构定义的就是类型定义

```
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
```

这里一共用了5个数字来定义sds的类型,1-5最多只用3bit就可以表示出来。所以这里还定义了一个SDS_TYPE_MASK,它的值为7,二进制表示为0111,只要将它与1-5
做&运算,最后就能sds对应的类型

# 2.初始化函数

如果到这里接着往下看,可以看出接下来很多函数都会传入一个sds参数,类似下面这样

`static inline size_t sdslen(const sds s)`

并且从后面的代码可以看出，这个sds类型的参数才是用来表示所谓的简单动态数组类型的字符串的，并且和前面的sdshdr\*这5个类型没有关联

那么这时候往前看，头文件的开头有一句

`typedef char *sds;`

从这里可以看出sds实际上还是一个char\*类型，也就是一个字符指针，这里就解释了sds的第一个特性：与传统的c字符串兼容，但是还有两个特

性无法解释，接着往下看。

这里找到了两个返回类型为sds的new函数，分别为

```
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}

sds sdsnewlen(const void *init, size_t initlen);
```

可以看出，sdsnew先传入一个字符串init，并返回一个将init和init大小作为参数调用sdsnewlen()的结果。

那么下面分析sdsnewlen()

```
sds sdsnewlen(const void *init, size_t initlen) {
	/* 先声明了一个空指针sh(从后面推测可能是sds header的意思)
	s* 和一个sds类型变量s(也就是char\*，c类型字符串) */

    void *sh;
    sds s;

    /* 定义了一个表示sds的5个类型的变量type，并调用[sdsReqType()](#sdsReqType)，
     * 根据传入的字符串长度判断应该使用哪个类型的sds。 */

    char type = sdsReqType(initlen);

    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */

    /* 接着往下走，因为sdshdr5已经启用，所以如果判断该使用sdshdr5，则换成sdshdr8
     * 并且如果长度为0，也就是空字符串，也使用sdshdr8。 */

    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;

    /* 根据type类型获得sdshdr的大小*/

    int hdrlen = sdsHdrSize(type);

    /* 定义一个fp指针，后面可能用来表示sdshdr里的flag*/

    unsigned char *fp; /* flags pointer. */

    /* s_malloc实际上就是z_malloc*/

    sh = s_malloc(hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);

    /* s为sh往后偏移hdrlen的地址，也就是sdshdr开始的地址*/
    s = (char*)sh+hdrlen;

    /* 因为flag是在sdshdr的最后面，所以此时fp指向了flag*/
    fp = ((unsigned char*)s)-1;

    /* 根据sdshdr的类型调用SDS_HDR_VAR()，并将sdshdr里的len和alloc设置成initlen，也就是字符串的长度
     * flag设置成sdshdr的类型*/
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }

    /* 将init的字符复制到s里*/
    if (initlen && init)
        memcpy(s, init, initlen);

    /* 为了兼容c字符串，末尾添加'\0*/
    s[initlen] = '\0';

    /* 返回s*/
    return s;
}
```
构造函数分析完，对sds大概就有点印象了，redis的字符串s本质上就是char\*类型，但是它还带了一个header，也就是

前面看到的sdshdr。为什么叫header，就是因为在内存中sdshdr是邻接在s前面，或者可以换个说法，s其实就是sdshdr最后面的

柔性数组，这样就能保证了sds的可动态扩展的特性，并且同时既能存储任意的二进制数据也能与传统的c字符串兼容——只要末尾有个'\0'就可以了。

~~下面用一张图来具体表示一下sds的内存结构,周五下班了暂时不写了~~


## sdsReqType

```
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
    return SDS_TYPE_64;
#else
    return SDS_TYPE_32;
#endif
}
```