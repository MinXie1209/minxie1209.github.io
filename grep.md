## grep性能分析

先来看一组数据,基于点评定时任务服务 进行日志搜索分析 

- 拉取all-2022-10-31.log 的日志进行搜索 这个日志的大小为**2.9G**

performance # ll -sh
total 4.8G
2.9G -rw-r--r-- 1 root root 2.9G 10月 31 23:59 all-2022-10-31.log
1.9G -rw-r--r-- 1 root root 1.9G 11月  1 15:35 all.log
 92K -rw-r--r-- 1 root root  90K 10月 31 23:03 error-2022-10-31.log
 64K -rw-r--r-- 1 root root  61K 11月  1 15:03 error.log



- grep -c : 耗时**9.950s** ,扫描出匹配的数据量为**1113484**

performance # time grep -c "log" all-2022-10-31.log 
1113484

real   0m9.950s
user   0m2.986s
sys   0m2.279s



- 当只扫描匹配的前**100000**条, grep -c : 耗时**0.771s**

performance # time grep -c  -m 100000 "log" all-2022-10-31.log 
100000

real    0m0.771s
user    0m0.226s
sys     0m0.175s



- 当只扫描匹配的前**10000**条, grep -c : 耗时**0.033s**

performance # time grep -c  -m 10000 "log" all-2022-10-31.log 
10000

real    0m0.033s
user    0m0.004s
sys     0m0.011s



- 当只扫描匹配的前**1000**条, grep -c : 耗时**0.003s**

performance # time grep -c  -m 1000 "log" all-2022-10-31.log 
1000

real    0m0.003s
user    0m0.003s
sys     0m0.000s



- 当只扫描匹配的前**100**条, grep -c : 耗时**0.004s**

performance # time grep -c  -m 100 "log" all-2022-10-31.log 
100

real    0m0.004s
user    0m0.000s
sys     0m0.004s



- 当只扫描匹配的前**10**条, grep -c : 耗时**0.003s**

performance # time grep -c  -m 10 "log" all-2022-10-31.log 
10

real    0m0.003s
user    0m0.000s
sys     0m0.003s



- 当只扫描匹配的前**1**条, grep -c : 耗时**0.002s**

performance # time grep -c  -m 1 "log" all-2022-10-31.log 
1

real    0m0.002s
user    0m0.001s
sys     0m0.001s



> 所以当我们需要知道日志里面有没有匹配的文本, 这时候为了保证性能,只需要扫描出匹配文本的1行记录就好,因为这时候性能是最高的
>



