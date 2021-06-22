---
title: Python 字符串的内存管理
date: 2021-06-22T07:14:06.880Z
draft: false
featured: false
tags:
  - 字符串
  - 源码
categories:
  - python
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
[原文链接](https://rushter.com/blog/python-strings-and-memory/)

从 Python 3 开始，内置的 str 类型使用 Unicode 表示。 根据编码的不同，一个 Unicode 字符最多会占用 4 个字节，从内存的角度来看，这个代价可能有点高。

为了减少内存消耗、提高性能，Python 中的 Unicode 字符串使用了三种内部表示:

* 每个字符 1 字节(Latin-1 编码)
* 每个字符 2 字节(UCS-2 编码)
* 每个字符 4 字节(UCS-4 编码)

使用 Python 进行编程时，所有字符串的行为都是相同的，大多数时候我们并不会注意到其中差异。然而，在处理大量文本时，这种差异可能是非常明显的，有时甚至不符合预期。

为了观察内部表示的差异，使用 sys.getsizeof 函数，它以字节为单位返回对象的大小:

```python
import sys
string = 'hello'
sys.getsizeof(string)
# 54

# 1-byte encoding
sys.getsizeof(string+'!')-sys.getsizeof(string)
# 1

# 2-byte encoding
string2  = '你'
sys.getsizeof(string2+'好')-sys.getsizeof(string2)
# 2

sys.getsizeof(string2)
# 76

# 4-byte encoding
string3 = '🐍'
sys.getsizeof(string3+'💻')-sys.getsizeof(string3)
# 4

sys.getsizeof(string3)
# 80
```

正如我们所看到的，Python 根据字符串的内容，使用不同的编码。注意，Python 中的每个字符串都需要额外的49-80字节内存，其中存储补充信息，如散列、长度、字节长度、编码类型和字符串标志。 这就是为什么一个空字符串需要49字节的内存。

上述代码所演示的内容可以通过 CPython 的源码看出来

在CPython的源码中，str 被定义在unicodeobject.h中，用PyUnicodeObject来表示，PyUnicodeObject的代码如下：

```c
typedef struct {
    /* There are 4 forms of Unicode strings:

       - compact ascii:

         * structure = PyASCIIObject
         * test: PyUnicode_IS_COMPACT_ASCII(op)
         * kind = PyUnicode_1BYTE_KIND
         * compact = 1
         * ascii = 1
         * ready = 1
         * (length is the length of the utf8 and wstr strings)
         * (data starts just after the structure)
         * (since ASCII is decoded from UTF-8, the utf8 string are the data)

       - compact:

         * structure = PyCompactUnicodeObject
         * test: PyUnicode_IS_COMPACT(op) && !PyUnicode_IS_ASCII(op)
         * kind = PyUnicode_1BYTE_KIND, PyUnicode_2BYTE_KIND or
           PyUnicode_4BYTE_KIND
         * compact = 1
         * ready = 1
         * ascii = 0
         * utf8 is not shared with data
         * utf8_length = 0 if utf8 is NULL
         * wstr is shared with data and wstr_length=length
           if kind=PyUnicode_2BYTE_KIND and sizeof(wchar_t)=2
           or if kind=PyUnicode_4BYTE_KIND and sizeof(wchar_t)=4
         * wstr_length = 0 if wstr is NULL
         * (data starts just after the structure)

       - legacy string, not ready:

         * structure = PyUnicodeObject
         * test: kind == PyUnicode_WCHAR_KIND
         * length = 0 (use wstr_length)
         * hash = -1
         * kind = PyUnicode_WCHAR_KIND
         * compact = 0
         * ascii = 0
         * ready = 0
         * interned = SSTATE_NOT_INTERNED
         * wstr is not NULL
         * data.any is NULL
         * utf8 is NULL
         * utf8_length = 0

       - legacy string, ready:

         * structure = PyUnicodeObject structure
         * test: !PyUnicode_IS_COMPACT(op) && kind != PyUnicode_WCHAR_KIND
         * kind = PyUnicode_1BYTE_KIND, PyUnicode_2BYTE_KIND or
           PyUnicode_4BYTE_KIND
         * compact = 0
         * ready = 1
         * data.any is not NULL
         * utf8 is shared and utf8_length = length with data.any if ascii = 1
         * utf8_length = 0 if utf8 is NULL
         * wstr is shared with data.any and wstr_length = length
           if kind=PyUnicode_2BYTE_KIND and sizeof(wchar_t)=2
           or if kind=PyUnicode_4BYTE_KIND and sizeof(wchar_4)=4
         * wstr_length = 0 if wstr is NULL

       Compact strings use only one memory block (structure + characters),
       whereas legacy strings use one block for the structure and one block
       for characters.

       Legacy strings are created by PyUnicode_FromUnicode() and
       PyUnicode_FromStringAndSize(NULL, size) functions. They become ready
       when PyUnicode_READY() is called.

       See also _PyUnicode_CheckConsistency().
    */
    PyObject_HEAD
    Py_ssize_t length;          /* Number of code points in the string */
    Py_hash_t hash;             /* Hash value; -1 if not set */
    struct {
        /*
           SSTATE_NOT_INTERNED (0)
           SSTATE_INTERNED_MORTAL (1)
           SSTATE_INTERNED_IMMORTAL (2)

           If interned != SSTATE_NOT_INTERNED, the two references from the
           dictionary to this object are *not* counted in ob_refcnt.
         */
        unsigned int interned:2;
        /* Character size:

           - PyUnicode_WCHAR_KIND (0):

             * character type = wchar_t (16 or 32 bits, depending on the
               platform)

           - PyUnicode_1BYTE_KIND (1):

             * character type = Py_UCS1 (8 bits, unsigned)
             * all characters are in the range U+0000-U+00FF (latin1)
             * if ascii is set, all characters are in the range U+0000-U+007F
               (ASCII), otherwise at least one character is in the range
               U+0080-U+00FF

           - PyUnicode_2BYTE_KIND (2):

             * character type = Py_UCS2 (16 bits, unsigned)
             * all characters are in the range U+0000-U+FFFF (BMP)
             * at least one character is in the range U+0100-U+FFFF

           - PyUnicode_4BYTE_KIND (4):

             * character type = Py_UCS4 (32 bits, unsigned)
             * all characters are in the range U+0000-U+10FFFF
             * at least one character is in the range U+10000-U+10FFFF
         */
        unsigned int kind:3;
        /* Compact is with respect to the allocation scheme. Compact unicode
           objects only require one memory block while non-compact objects use
           one block for the PyUnicodeObject struct and another for its data
           buffer. */
        unsigned int compact:1;
        /* The string only contains characters in the range U+0000-U+007F (ASCII)
           and the kind is PyUnicode_1BYTE_KIND. If ascii is set and compact is
           set, use the PyASCIIObject structure. */
        unsigned int ascii:1;
        /* The ready flag indicates whether the object layout is initialized
           completely. This means that this is either a compact object, or
           the data pointer is filled out. The bit is redundant, and helps
           to minimize the test in PyUnicode_IS_READY(). */
        unsigned int ready:1;
        /* Padding to ensure that PyUnicode_DATA() is always aligned to
           4 bytes (see issue #19537 on m68k). */
        unsigned int :24;
    } state;
    wchar_t *wstr;              /* wchar_t representation (null-terminated) */
} PyASCIIObject;

/* Non-ASCII strings allocated through PyUnicode_New use the
   PyCompactUnicodeObject structure. state.compact is set, and the data
   immediately follow the structure. */
typedef struct {
    PyASCIIObject _base;
    Py_ssize_t utf8_length;     /* Number of bytes in utf8, excluding the
                                 * terminating \0. */
    char *utf8;                 /* UTF-8 representation (null-terminated) */
    Py_ssize_t wstr_length;     /* Number of code points in wstr, possible
                                 * surrogates count as two code points. */
} PyCompactUnicodeObject;

/* Strings allocated through PyUnicode_FromUnicode(NULL, len) use the
   PyUnicodeObject structure. The actual string data is initially in the wstr
   block, and copied into the data block using _PyUnicode_Ready. */
typedef struct {
    PyCompactUnicodeObject _base;
    union {
        void *any;
        Py_UCS1 *latin1;
        Py_UCS2 *ucs2;
        Py_UCS4 *ucs4;
    } data;                     /* Canonical, smallest-form Unicode buffer */
} PyUnicodeObject;
```

在代码PyUnicodeObject的定义中，可以清晰的看到，实际的字符串是Py_UCS1，Py_UCS2，Py_UCS4中的一种。而在 PyASCIIObject 的定义中，可以看到在state这个结构体中的kind变量，使用了3个二进制位来标识字符串中的字符是什么类型。

如果一个字符串中的所有字符都可以在 ASCII 范围内，那么它们就使用1字节的 latin-1编码进行编码。 latin-1表示前256个 Unicode 字符。它支持许多拉丁语言，如英语、瑞典语、意大利语、挪威语等大多数秀语言。 然而，它不能存储非拉丁语言，如汉语，日语，希伯来语，西里尔字母。 这是因为它们的代码点(数值索引)定义在1字节(0-255)范围之外。

大多数流行的自然语言都可以采用2字节(UCS-2)编码。 当字符串包含特殊符号、表情符号或稀有语言时，使用4字节(UCS-4)编码。 在 Unicode 标准中几乎有300个块(范围)。在 0xFFFF 块之后，可以看到什么样的符号使用4字节表示。

假设有一个 ASCII 文本，全部加载到内存中大小为 10 GB。如果在文本中插入一个的表情符号，整个字符串的大小将增加4倍！这是实践 NLP 问题时可能会遇到的一个巨大问题。

## 为什么 Python 不在内部使用 UTF-8编码

最著名和最流行的 Unicode 编码是 UTF-8，但 Python 并不使用这种方式作为内部表示。

在 UTF-8 编码中存储字符串时，根据字符所表示的字符，对每个字符进行编码时使用1-4个字节。 
这是一种高效的存储编码，但有一个明显的缺点：由于每个字符的长度不一定相同，因此不扫描整个字符串就无法通过索引随机访问单个字符。 因此，若使用 UTF-8 ，Python 执行诸如 string\[5] 之类的简单操作，需要扫描字符串，直到找到所需的字符。 固定长度的编码没有这样的问题，要通过索引 Python 定位一个字符，只需将一个索引号乘以蛋字符的长度(1、2或4字节)。

## 字符串驻留（String interning）

在python中，对只含有ASCII字母的短字符串，单字符字符串，和空字符串使用字符串驻留机制。被驻留的字符串作为一个整体被使用，也就是说，如果你有两个相同的字符串被驻留，实际上在内存中只存在一份。

```python
In [1]: a = 'hello'

In [2]: b = 'world'

In [3]: c = 'hello'

In [4]: a[4],b[1]
Out[4]: ('o', 'o')

In [5]: id(a[4]), id(b[1]), a[4] is b[1]
Out[5]: (4538234672, 4538234672, True)

In [6]: id('')
Out[6]: 4536707760

In [7]: id('')
Out[7]: 4536707760

In [8]: id(a), id(c)
Out[8]: (4572722992, 4572722992)
```

如上代码所示，两个字符串字串指向内存中的相同地址。这是因为 Python 字符串是不可变的。

在 Python 中，字符串驻留机制不仅限于字符或空字符串。 在代码编译期间，如果创建的字符串的长度不超过20个字符，字符串驻留机制也会开启。

编译期间穿件的字符串包括:

* 函数名和类名
* 变量名
* 参数名
* 常量（所有代码中定义的字符串）
* 字典的键
* 属性名

当您在 python REPL 中按回车键时，输入的语句将被编译成字节码。 这就是为什么在 REPL 中所有的短字符串都要被驻留的原因。

```python
>>> a = 'teststring'
>>> b = 'teststring'
>>> id(a), id(b), a is b
(4569487216, 4569487216, True)
>>> a = 'test'*5
>>> b = 'test'*5
>>> len(a), id(a), id(b), a is b
(20, 4569499232, 4569499232, True)
>>> a = 'test'*6
>>> b = 'test'*6
>>> len(a), id(a), id(b), a is b
(24, 4569479328, 4569479168, False)
```

字符串驻留机制节省了数以万计的重复字符串分配。 在内部，字符串驻留机制由一个全局字典来维护，字符串用作键。通过字典的包含操作检查内存 Python 中是否已经有相同的字符串。

Unicode 对象几乎有16000行 c 代码，因此有许多小的优化在本文中没有提到。如果您想更多地了解 Python 中的 Unicode，我建议您阅读关于字符串的 pep，
并检查 Unicode 对象的代码。

## 参考资料

1. https://www.cnblogs.com/malecrab/p/5300503.html
2. https://github.com/python/cpython/blob/master/Include/unicodeobject.h
3. https://jrgraphix.net/research/unicode.php