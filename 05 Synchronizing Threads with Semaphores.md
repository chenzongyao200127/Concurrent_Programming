# Synchronizing Threads with Semaphores
# 用信号量同步线程

共享变量是十分方便，但是它们也引入了同步错误（synchronization error）的可能性。
考虑图 12-16 中的程序 badcnt.c，它创建了两个线程，每个线程都对共享计数变量 cnt 加 1。

~~~c
/* WARNING: This code is buggy! */
#include "csapp.h"

void *thread(void *vargp); /* Thread routine prototype */

/* Global shared variable */
volatile long cnt = 0; /* Counter */

int main(int argc, char **argv)
{
    long niters;
    pthread_t tid1, tid2;

    /* Check input argument */
    if (argc != 2) {
        printf("usage: %s <niters>\n", argv[0]);
        exit(0);
    }
    niters = atoi(argv[1]);

    /* Create threads and wait for them to finish */
    Pthread_create(&tid1, NULL, thread, &niters);
    Pthread_create(&tid2, NULL, thread, &niters);
    Pthread_join(tid1, NULL);
    Pthread_join(tid2, NULL);

    /* Check result */
    if (cnt != (2 * niters))
        printf("BOOM! cnt=%ld\n", cnt);
    else
        printf("OK cnt=%ld\n", cnt);
    exit(0);
}

/* Thread routine */
void *thread(void *vargp)
{
    long i, niters = *((long *)vargp);

    for (i = 0; i < niters; i++)
        cnt++;

    return NULL;
}
~~~

因为每个线程都对计数器增加了 niters 次，我们预计它的最终值是 2 × niters。这看上去简单而直接。
然而，当在 Linux 系统上运行 badcnt.c 时，我们不仅得到错误的答案，而且每次得到的答案都还不相同！

~~~shell
czy@czy-307-thinkcentre-m720q-n000 ~/new_space/Concurrent_Programming/05
☺  ./badcnt 1000000                                                                                                           master ✗
BOOM! cnt=1004913

czy@czy-307-thinkcentre-m720q-n000 ~/new_space/Concurrent_Programming/05
☺  ./badcnt 1000000                                                                                                           master ✗
BOOM! cnt=1429824
~~~













----------------------------------------------------------------------------------------------------
C语言中的 `atoi()` 函数用于将字符串转换为整数。这个函数定义在 `<stdlib.h>` 头文件中。`atoi` 是 “ASCII to Integer” 的缩写。它的原型如下：

```c
int atoi(const char *str);
```

### 参数
- `str`：指向以 null 结尾的字符串的指针，该字符串表示要转换为整数的数字。

### 功能
- `atoi()` 函数扫描参数 `str` 指向的字符串，跳过前面的空白字符（例如空格，tab缩进等），直到第一个非空白字符。
- 从这个字符开始，函数将连续的数字（0-9）转换为整数。转换过程一直进行到遇到非数字字符或字符串结束的 null 字符。
- 如果字符串的第一个非空字符不是有效的整数或者字符串为空，或者字符串中没有其他字符，则不进行转换，并返回零。

### 返回值
- 返回转换后的整数值。如果没有执行有效的转换，则返回零。

### 注意事项
- `atoi()` 不检测溢出。当转换的值超过 `int` 类型能表示的范围时，可能会产生不确定的结果。
- 如果字符串表示的数字超过了 `int` 能表示的范围，建议使用 `strtol()` 或 `strtoll()` 函数，这些函数能提供更好的错误检测和报告。

### 示例
```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    char str1[] = "1234";
    char str2[] = "100a";  // 非数字字符会停止转换
    char str3[] = "   456"; // 前面的空格会被忽略

    int num1 = atoi(str1);
    int num2 = atoi(str2);
    int num3 = atoi(str3);

    printf("num1 = %d\n", num1);
    printf("num2 = %d\n", num2); // 由于遇到非数字字符 'a'，只会转换 '100'
    printf("num3 = %d\n", num3);

    return 0;
}
```

### 输出
```
num1 = 1234
num2 = 100
num3 = 456
```

这是 `atoi()` 函数的基本介绍和使用方法。
----------------------------------------------------------------------------------------------------