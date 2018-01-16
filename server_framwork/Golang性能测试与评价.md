
##Golang性能测试与思考

本文测试Go、Python、PyPy、C的效率，作为学习Go的参考标准。测试用例：进行(2<<25)次简单加法

>测试环境：<br> 
>系统：Windows7 专业版<br> 
>CPU：Intel(R) Core(TM) i5-4590 CPU @ 3.30GHZ 3.30GHZ, 14级流水线（Pipeline）<br> 

参考资料：
[Core微架构14级流水线](https://zh.wikipedia.org/wiki/Core%E5%BE%AE%E6%9E%B6%E6%A7%8B)

测试用例：进行(2<<25)次简单加法

```go
// golang example
package main

import "fmt"
import "time"

func main() {
    var N int = 2 << 25
    var j int = 0

    t0 := time.Now()

    for i := 1; i < (N); i++ {
        j += 1
    }

    elapsed := time.Since(t0)
    fmt.Println("N:", N, "time:", elapsed)
}
```
测试结果：N: 67108864 time: 20.5027ms


```Python
# -*- coding: utf-8 -*-
# Python and Pypy example

import time

t0 = time.clock()

N = 2 << 25
j = 0

for i in xrange(N):
    j += 1

print "N:{}, time:{:.2f}ms".format(N, (time.clock() - t0) * 1000)
```
测试结果：<br> 
Python N:67108864, time:5102.14ms<br> 
PyPy: N:67108864, time:124.16ms

```C
#include "stdafx.h"
#include<stdlib.h>
#include "test1.h"

#include<stdio.h>
#include<time.h>


#ifdef _WIN32
#include <Windows.h>
#include <sys/timeb.h>
#elif defined __linux__
#include <time.h>
#endif

 unsigned long ClockMs64()
{
#ifdef _WIN32
    struct __timeb64 timebuffer;
    _ftime64_s(&timebuffer);
    return timebuffer.time * 1000 + timebuffer.millitm;

#else
    timespec ts;
    if (-1 == clock_gettime(CLOCK_REALTIME, &ts))
    {
        return 0;
    }
    return ((unsigned long)ts.tv_sec) * 1000 + ts.tv_nsec/1000000;
#endif
}

 unsigned long Clock()
{
    return (unsigned long)ClockMs64();
}

int _tmain(int argc, _TCHAR* argv[])
{
    UINT64 N =  2 << 25;
    UINT64 j = 1;
    unsigned long start = Clock();

    for(UINT64 i=0;i<N;i++)
    {
        j += 1;
    }
    unsigned long end = Clock();
    printf("N:%I64d, time:%dms\n", N, end-start);
    return 0;
}

```
测试结果：N:67108864, time:160ms

综合以上测试结果：<br>

|        | 时间 |
| :------|:------ |
|   C    | 160ms |
|  PyPy  | 124.16ms |
| Python | 5102.14ms |
| Golang | 20.50ms|


结论：从测试结果来看，执行效率Golang > PyPy > C > Python <br>
分析：<br>
Golang是编译型语言，会将代码编译成二进制可执行机器码。我们来分析执行Golang的执行时间，我们知道一个汇编指令至少执行一个CPU周期（指令执行整数个周期，大部分指令执行一个周期，指令具体执行周期可以参考CPU手册）。也就是说只要我们知道执行整个程序的汇编指令数量，我们就能对程序运行时间做出评判。
```
测试用例循环体内执行以下操作，大致翻译成汇编指令，统计汇编指令数目
1. i < N --> 将i、N入栈，cmp后比较结果来做Jmp操作，大约5条汇编指令；
2. j += 1 --> 将j入栈，执行Inc操作，然后保存到j，大约3条汇编指令；
3. i ++ --> 将i入栈，执行Inc，保存到i，大约3条汇编；
4. 还有一个隐藏的Jump到操作1的操作， 1条汇编

故一个循环体总体汇编指令条目为(5+3+3+1)=12条
```
CPU主频为3.30GHZ，循环体汇编条目约为12条。我们以CPU满载、每条指令一个周期计算，执行`2<<25`次循环体需要用时=1000 * ((2<<25) * 12) / (3.3 \* (10**9)) = 244.03ms。
而我们测试Golang用时20.50ms，为什么Golang比前面计算的244.03ms远远要小？秘密在于CPU的指令流水线，根据资料我们知道Intel CPU是14级流水线架构，也就是说一个CPU周期最大可执行14条汇编指令。我们用单级流水线理论时间除以Golang用时：244.03ms / 20.50ms = 11.9，考虑多CPU周期汇编指令的存在和系统调度消耗，14是我们理论上限值，可见结果符合预期。

Golang在这个测试用例中比C语言更快，更是秒杀Python，超乎预期。通过计算发现Golang的计算效率接近CPU的极限，着实是一门高效的编译型语言。

另外，附上常见的Benchmark连接[The Computer Language
Benchmarks Game](http://benchmarksgame.alioth.debian.org/u64q/task-descriptions.html)，可对比各种语言的性能情况

参考资料：<br>
[Core微架构-14级流水线](https://zh.wikipedia.org/wiki/Core%E5%BE%AE%E6%9E%B6%E6%A7%8B)