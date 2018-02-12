
### PythonCPU诊断优化

> 无论用什么语言来实现需求，最后基本都会需要对代码运行时间进行优化，降低cpu占用率

这里主要介绍两个profile工具：

1. cProfile：用cProfile生成程序运行时的profile文件，用`python pyprof2calltree.py -i file`生成文件供QCacheGrind读取；然后用qcachegrind.exe查看，可以看到图像化的代码情况，面积表示cpu占比，可以很方便的找到cpu占比高的代码来进行优化。
2. vmprof：vmprof支持pypy，先用vmprof生成profile文件（`pypy -m vmprof -o log.jit example.py`），用`vmprofshow log.jit`来查看代码占用cpu情况；

cprofile图形化结果更加强大，但是不支持pypy。
