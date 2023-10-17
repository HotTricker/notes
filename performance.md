# 性能调优

## 指标

性能用对cpu、内存、带宽、磁盘等核心资源的占用情况及稳定性来衡量

除了资源层面，还有应用自身的指标：异常率、响应时间、吞吐量

## 优化方案

不要反向优化（优化某个方面但其他方面下降），不要过度优化（优化空间小产出低）

- 优化代码
  - 提高复用性，减少重复代码开发
  - 池化，对象池减少内存分配、协程池避免无限创建协程打满内存
  - 并行化
  - 异步化
  - 使用时间复杂度更低的算法
- 使用设计模式
- 空间换时间/时间换空间的抉择
- 使用更好的第三方库

## benchmark

- testing内置了对基准测试的支持，默认是在1s内重复运行测试代码，输出执行次数和内存分配结果
  - 基准测试采用* testing.B
  - go test默认排除基准测试，要运行的话使用-bench=. 要排除测试可以用-run=^$提供一个不匹配任何东西的正则表达式
  - 可以指定用几个内核运行，可以指定基准测试时间，指定count多次运行，benchstat可以查看均值和样本偏差以及对比两次的结果
  - 可以用resetTimer忽略之前累积的时间
  - 可以指定-benchmem查看内存分配的统计信息

## pprof

- cpu、内存分析
- 块分析，类似cpu，可以查看被阻塞（使用通道或者锁）的goroutine，确定并发瓶颈，应该放在cpu和内存分析之后
- pprof.StartCpuProfile() defer pprof.StopCpuProfile()
- defer profile.Start(profile.ProfilePath(".")).Stop()
- 火焰图横条越长，资源消耗、占用越多
- top n：flat表示当前函数运行耗时不包含内部调用，sum表示当前行加上之前所有行运行耗时，cum表示当前函数加上内部调用运行耗时
- trace
  - 获得gouroutine的一些信息（调度、网络、系统调用、垃圾回收等）
  - 可以诊断并行度不足、gc导致延迟大、gouroutine阻塞等产生的问题
  - defer profile.Start(profile.TraceProfile, profile.ProfilePath(".")).Stop()
  - view trace
    - 采样状态：标识了goroutines(goroutine的数量)、heap(某个时间点上Go应用heap分配情况，包括已经分配的Allocated和下一次GC的目标值NextGC)和threads(Go应用启动的线程数量情况，包括running和insyscall)，可以点击某个时间点查看
    - P视角区，看到每个P上的所有事件
    - 事件区详情
      - Title：事件的可读名称；
      - Start：事件的开始时间，相对于时间线上的起始时间；
      - Wall Duration：这个事件的持续时间，这里表示的是G1在P4上此次持续执行的时间；
      - Start Stack Trace：当P4开始执行G1时G1的调用栈；
      - End Stack Trace：当P4结束执行G1时G1的调用栈；
      - Incoming flow：触发P4执行G1的事件；
      - Outgoing flow：触发G1结束在P4上执行的事件；
      - Preceding events：与G1这个goroutine相关的之前的所有的事件；
      - Follwing events：与G1这个goroutine相关的之后的所有的事件
      - All connected：与G1这个goroutine相关的所有事件。
  - Goroutine analysis从G视角看执行途径

## 逸出分析

- `go -build -gcflags=-m main.go`打印编译器的逃逸分析
  - -m print optimization decisions 打印优化决定，即是否有变量逃逸等
  - -l disable inlining 关闭内联，内联可能影响一些行为，关闭后简化问题分析，两个或三个是更激进的内联策略。-l -N禁用所有优化
  - -s打印汇编

## 内联

- 函数调用有固定的开销：堆栈和抢占检查，内联是避免这些成本的经典优化方法

## 死代码消除

- 有些代码永远无法访问，不需要在最终的二进制文件中编译/优化/发布

## 边界检查消除

- 数组可以在编译时检查，切片必须在运行时完成

## gogc

可以调整GOGC环境变量，大于100时堆快速增长，减小gc压力；小于100时增长缓慢，增加gc压力

## 减少allocation

- func (r *Reader) Read() ([]byte, error)
- func (r *Reader) Read(buf []byte) (int, error)

第一个Read方法总是分配一个缓冲区，给GC施加压力。第二个填充它被给予的缓冲区。

## 一些技巧

- 字符串连接尽量用+而不是fmt.Sprintf
- 长度已知尽量分配足够的长度
- 遍历复杂结构体切片[]struct{}用下标好于用元素，简单切片或者结构体指针则无所谓
- 使用对象池（实现一个New就可以，用get、put来使用）、协程池
- 用unsafe避开内存拷贝，尽量避免系统调用
