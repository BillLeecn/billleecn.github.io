---
layout: post
title: C 风格的 struct 初始化
tags:
    - C
    - struct
    - initialization
comments: on
---

自己学的是 C++, 从来没用过 C 风格的大括号初始化。然后，今天参加深圳某公司的校招笔试，题目全是用的大括号初始化，直接跪了……

直接上测试代码：

{% highlight c %}
#include <stddef.h>
#include <stdio.h>

typedef struct{
    char    *a;
    int     b;
    int     c;
} StructA;

#define FORMAT_STRUCTA  "{{ "{%" }}p, %d, %d}\n"

int main(){
    StructA sa = {0};
    StructA sb = {NULL, 1};
    StructA sc = {NULL, 1, 2};
    StructA sd = {
        .a = NULL,
        .b = 1,
        .c = 2
    };
    StructA se;
    printf(FORMAT_STRUCTA, sa.a, sa.b, sa.c);
    printf(FORMAT_STRUCTA, sb.a, sb.b, sb.c);
    printf(FORMAT_STRUCTA, sc.a, sc.b, sc.c);
    printf(FORMAT_STRUCTA, sd.a, sd.b, sd.c);
    printf(FORMAT_STRUCTA, se.a, se.b, se.c);

    char a1[3] = {1};
    char a2[3] = {1, 2};
    char a3[3] = {1, 2, 3};

    printf("%hhu, %hhu, %hhu\n", a1[0], a1[1], a1[2]);
    printf("%hhu, %hhu, %hhu\n", a2[0], a2[1], a2[2]);
    printf("%hhu, %hhu, %hhu\n", a3[0], a3[1], a3[2]);
    return 0;
}
{% endhighlight %}

编译参数：`gcc -ansi -Wall -Werror`, 结果：

```
{0x0, 0, 0}
{0x0, 1, 0}
{0x0, 1, 2}
{0x0, 1, 2}
{0x0, -2147048434, 1}
1, 0, 0
1, 2, 0
1, 2, 3
```

结论：

**无论是数组还是 struct, 在初始化列表不完全的时候，空缺的部分以 0 填充。**
