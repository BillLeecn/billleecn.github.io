---
layout: post
title: 内存寻址与字节对齐 
tags:
    - C
    - struct
    - alignment
    - memory
comments: true
---

在 C 语言，编译器给 struct 的内存会有字节对齐的要求。字节对齐的目的是为了提高访问速度，这和 CPU 与内存之间的通信方式有关。

以  x86 架构为例，系统的数据总线宽度为 32 bit. 实际上，内存是按字 (word) 访问的，一个字的宽度和数据总线宽度相同。内存可以抽象为一个这样的设备：

| 端口 | 宽度 |  方向 | 说明 |
| ------ | --------| ------- | ------ |
| clk     | 1 | in | 时钟信号 |
| addr | 34 | in | 地址 |
| write | 1 | in | 是否写入 |
| en | 1 | in | 是否输出 |
| data | 32 | in/out | 数据接口 |

内存根据 addr 端口输入的地址，读取或改写对应地址处的字。这里的地址，是按字为单位计算的。

但是在 x86 指令集（以及 C 语言）中，地址是按字节计算的。这实际上是 CPU 中的硬件抽象了对内存的访问，但内存的硬件限制，就是每个内存周期只能访问一个字，所以就产生了字节对齐的问题。

以一个 32 bit 的 `int a` 为例，当 `a`  在内存中的位置刚好对应一个内存字, 即 `&a % 4 == 0` 时，CPU 可以用一个内存周期来访问这个 int. 如果这个 int 和一个内存字不对齐，那么 CPU 就需要用两个内存周期，读取两个字，从中取出这个 int 的内容。（以上讨论忽略 cache 的存在）

为了达到最高的访问效率，编译器会在 struct 中添加 padding 来满足对齐要求。在 x86 架构上，对齐要求是 4 字节。在这种情况下，对于小于 4 字节的数据类型，按数据类型本身的宽度对齐；对于大于 4 字节的数据类型，按照 4 字节对齐。例如：

{% highlight c %}
#include <stdio.h>

struct S{           /* 偏移地址 */
    char a;         /* 0x00 */
    /*  1 byte padding  */
    short b;        /* 0x02 */
    long long c;    /* 0x04 */
    int d;          /* 0x0C */
    char e;         /* 0x10 */
    /*  3 byte padding  */
};

int main(){
    struct S data[2];
    printf("0x%X\n", (unsigned long)(&data[0].b) - (unsigned long)data);
    printf("0x%X\n", (unsigned long)(&data[0].c) - (unsigned long)data);
    printf("0x%X\n", (unsigned long)(&data[0].d) - (unsigned long)data);
    printf("0x%X\n", (unsigned long)(&data[0].e) - (unsigned long)data);
    printf("0x%X\n", sizeof(struct S));
    printf("0x%X\n", (unsigned long)(data + 1) - (unsigned long)data);
    return 0;
}
{% endhighlight %}

在 x86 架构下编译运行，应该输出 

```
0x2
0x4
0xC
0x10
0x14
0x14
```
