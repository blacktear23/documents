# 一个 Clang Bug 引发的血案

## 案发现场

最近在做 TiKV 相关的开发和测试，在改了一些 TiKV 的代码之后，在自己的 Mac 笔记本上用 `make release` 编译了一下 TiKV 然后准备挂上自己的数据做一下测试。然后 Segment Fault 发生了。

```
...
2017/12/18 10:50:48.657 tikv-server.rs:146: [WARN] environment variable `TZ` is missing, use `/etc/localtime`
2017/12/18 10:50:48.667 util.rs:426: [INFO] connect to PD leader "http://127.0.0.1:2379"
2017/12/18 10:50:48.667 util.rs:357: [INFO] All PD endpoints are consistent: ["127.0.0.1:2379"]
2017/12/18 10:50:48.667 tikv-server.rs:525: [INFO] connect to PD cluster 6498555866359339957
2017/12/18 10:50:48.716 mod.rs:497: [INFO] storage RaftKv started.
2017/12/18 10:50:48.720 mod.rs:308: [INFO] starting working thread: store address resolve worker
2017/12/18 10:50:48.721 server.rs:94: [INFO] listening on 127.0.0.1:20160
2017/12/18 10:50:48.723 node.rs:329: [INFO] start raft store 1 thread
[1]    86379 segmentation fault (core dumped)  ./tikv-server -C tikv.toml
```

想起了之前 PingCAP 同学说过，TiKV，PD 和 TiDB 版本不同有可能会导致 RPC 协议数据格式的版本不同，然后会导致 Segment Fault。好吧，先把 PD 和 TiDB 都升级到最新的代码试试。

经过大约3分钟的编译，重新启动。然后 TiKV 仍旧 Segment Fault。好吧，看来是个大新闻。联系到了 PingCAP 的同学并向他们报了障。几分钟之后，PingCAP 那边的同学也反馈遇到了类似的问题，不过 Linux 下 build 的版本并没有遇到任何问题。

## 挖地三尺

遇到这种 Segment Fault 的问题一般来讲应该是遇到了一个隐藏比较深的 Bug，根据以前的经验，Segment Fault 多半是访问到了不该访问的内存。但鉴于 MacOS 并没有类似 Linux 的 syslog 中记录 Segment Fault 原因的日志，只好自己启动 lldb 来看看堆栈了。

经过一番查找，找到了引起 Segment Fault 的线程的案发现场：

```
* thread #19, stop reason = signal SIGSTOP
  * frame #0: 0x000000010d5a94d4 tikv-server`rocksdb::Version::AddIteratorsForLevel(rocksdb::ReadOptions const&, rocksdb::EnvOptions const&, rocksdb::MergeIteratorBuilder*, int, rocksdb::RangeDelAggregator*) + 692
    frame #1: 0x000000010d5a91f8 tikv-server`rocksdb::Version::AddIterators(rocksdb::ReadOptions const&, rocksdb::EnvOptions const&, rocksdb::MergeIteratorBuilder*, rocksdb::RangeDelAggregator*) + 72
    frame #2: 0x000000010d4a9b49 tikv-server`rocksdb::DBImpl::NewInternalIterator(rocksdb::ReadOptions const&, rocksdb::ColumnFamilyData*, rocksdb::SuperVersion*, rocksdb::Arena*, rocksdb::RangeDelAggregator*) + 249
    frame #3: 0x000000010d4ad85f tikv-server`rocksdb::DBImpl::NewIterator(rocksdb::ReadOptions const&, rocksdb::ColumnFamilyHandle*) + 495
    frame #4: 0x000000010d81de94 tikv-server`::crocksdb_create_iterator_cf(db=0x00007f8d0c674860, options=<unavailable>, column_family=0x00007f8d0c662d90) at c.cc:926 [opt]
    frame #5: 0x000000010cec6eb4 tikv-server`tikv::raftstore::store::engine::{{impl}}::new_iterator_cf at rocksdb.rs:213 [opt]
    frame #6: 0x000000010cec6e4d tikv-server`tikv::raftstore::store::engine::{{impl}}::new_iterator_cf(self=0x000000010ee98080, cf=<unavailable>, iter_opt=<unavailable>) at engine.rs:357 [opt]
...
```

这下子好了，至少找到引起 Segment Fault 的位置了：`rocksdb`。随即上 github 上查看了 TiKV 跟 `rust-rocksdb` 库相关的提交。总共找到了 3 个 PR 跟升级 `rust-rocksdb` 有关。接下来就可以做一些验证，找到引入问题的那个提交。

## 山穷水尽

找到了 3 个可以的嫌疑人（PR），那么挨个 Revert 然后 `make release` 最终定位到了最早的那个 PR 就是引入问题的罪魁祸首。

在拿到了这个证据之后，随机就跟 PingCAP 的同学进行讨论。发现这个 PR 的代码跟 RocksDB 的官方代码有细微的不同。但是他们也是在改进之后也没解决这个问题。然后 按照 PingCAP 的同学说用 `make unportable_release` build 确实没出现 Segment Fault 的问题。

看来这个问题真是难缠。难道这两个 build 模式又有什么不同么？抱着这个问题开始翻阅 TiKV 和 rust-rocksdb 两个项目的编译脚本。最终差别还是找到了：

* make release 会在 cc 后面加入 `-msse42` 参数
* make unportable_release 会在 cc 后面加入 `-march=native` 同时没有 `-msse42` 参数

这下明确了一点，这个 Segment Fault 大概率跟 SSE 指令相关的代码有关。根据这个线索，找到了 RocksDB 中的 `util/crc32c.cc` 文件。行数不多，但使用的跟 SSE 相关的函数经过排查发现是系统默认提供的。再一次无功而返。

这时想起了 lldb 中的反汇编工具，看看到底是死在哪里了：

```
...
    0x10d5a94b0 <+656>:  leaq   0x874889(%rip), %rcx      ; vtable for rocksdb::(anonymous namespace)::LevelFileIteratorState + 16
    0x10d5a94b7 <+663>:  movq   %rcx, (%r12)
    0x10d5a94bb <+667>:  movq   %rax, 0x10(%r12)
    0x10d5a94c0 <+672>:  movq   0x2d(%rbx), %rax
    0x10d5a94c4 <+676>:  movq   %rax, 0x4d(%r12)
    0x10d5a94c9 <+681>:  movaps (%rbx), %xmm0
    0x10d5a94cc <+684>:  movaps 0x10(%rbx), %xmm1
    0x10d5a94d0 <+688>:  movaps 0x20(%rbx), %xmm2
->  0x10d5a94d4 <+692>:  movaps %xmm2, 0x40(%r12)
    0x10d5a94da <+698>:  movaps %xmm1, 0x30(%r12)
    0x10d5a94e0 <+704>:  movaps %xmm0, 0x20(%r12)
    0x10d5a94e6 <+710>:  movq   0x60(%rbx), %rdi
    0x10d5a94ea <+714>:  testq  %rdi, %rdi
    0x10d5a94ed <+717>:  je     0x10d5a9514               ; <+756>
...
```

箭头指向了一个 `movaps` 指令，这是什么鬼...

## 柳暗花明

一路排查到这里，到头来还是什么有价值的东西都没找出来。在仔细整理了思路之后发现整个案件还有一个证据没有找到，Segment Fault 的实际原因。既然没法从 syslog 中找到对应的日志，那么在 lldb 中运行一下呢？

```
2017/12/18 11:12:59.553 tikv-server.rs:146: [WARN] environment variable `TZ` is missing, use `/etc/localtime`
2017/12/18 11:12:59.560 util.rs:426: [INFO] connect to PD leader "http://127.0.0.1:2379"
2017/12/18 11:12:59.560 util.rs:357: [INFO] All PD endpoints are consistent: ["127.0.0.1:2379"]
2017/12/18 11:12:59.560 tikv-server.rs:525: [INFO] connect to PD cluster 6498555866359339957
2017/12/18 11:12:59.624 mod.rs:497: [INFO] storage RaftKv started.
2017/12/18 11:12:59.629 mod.rs:308: [INFO] starting working thread: store address resolve worker
2017/12/18 11:12:59.629 server.rs:94: [INFO] listening on 127.0.0.1:20160
2017/12/18 11:12:59.632 node.rs:329: [INFO] start raft store 1 thread
Process 86787 stopped
* thread #2, name = 'raftstore-1', stop reason = EXC_BAD_ACCESS (code=EXC_I386_GPFLT)
    frame #0: 0x0000000100b164d4 tikv-server`rocksdb::Version::AddIteratorsForLevel(rocksdb::ReadOptions const&, rocksdb::EnvOptions const&, rocksdb::MergeIteratorBuilder*, int, rocksdb::RangeDelAggregator*) + 692
tikv-server`rocksdb::Version::AddIteratorsForLevel:
->  0x100b164d4 <+692>: movaps %xmm2, 0x40(%r12)
    0x100b164da <+698>: movaps %xmm1, 0x30(%r12)
    0x100b164e0 <+704>: movaps %xmm0, 0x20(%r12)
    0x100b164e6 <+710>: movq   0x60(%rbx), %rdi
Target 0: (tikv-server) stopped.
```

好了，证据收集到了，`reason = EXC_BAD_ACCESS (code=EXC_I386_GPFLT)`。好了，这些文字还是头一次见过，那么 googl...（好吧，这个网站不存在，我用的是 bing.com！😂)

在 Stack Overflow 网站中找到一个帖子，其中有这么一句话：

>Another likely causes is unaligned access with an SSE register - in other word, reading a 16-byte SSE register from an address that isn't 16-byte aligned.

刚好在 lldb 中也看到了 `movaps %xmm2, 0x40(%r12)` 这个指令。那么可以看看是不是 `r12` 这个寄存器里面的值有问题？

```
(lldb) register read
General Purpose Registers:
       rax = 0x0000000001000000
       rbx = 0x00000001022085c0
       rcx = 0x000000010138ad40  tikv-server`vtable for rocksdb::(anonymous namespace)::LevelFileIteratorState + 16
       rdx = 0x0000000000000000
       rdi = 0x0000000104001030
       rsi = 0x0000000102879e38
       rbp = 0x000070000b3fdf30
       rsp = 0x000070000b3fded0
        r8 = 0x000000010287a7e0
        r9 = 0x0000000104001478
       r10 = 0x00000000fffff800
       r11 = 0x00000000000041c0
       r12 = 0x0000000104001478
       r13 = 0x0000000102873e00
       r14 = 0x000070000b3fdf88
       r15 = 0x0000000000000001
       rip = 0x0000000100b164d4  tikv-server`rocksdb::Version::AddIteratorsForLevel(rocksdb::ReadOptions const&, rocksdb::EnvOptions const&, rocksdb::MergeIteratorBuilder*, int, rocksdb::RangeDelAggregator*) + 692
    rflags = 0x0000000000010206
        cs = 0x000000000000002b
        fs = 0x0000000000000000
        gs = 0x0000000000000000
```

`r12 = 0x0000000104001478`，最后一字节是 `0x78`！16 字节对其的话最后一位应该是 `0` 啊！这是什么操作！

## 真相只有一个

在找到这些线索之后，跟 PingCAP 的同学也交流了一下，最后 PingCAP 的同学也确认了这个问题的锅应该是编译器来背。

**MacOS 下 Xcode 9.2 携带的 clang 4.9 会有使用 `movaps` 指令单并没做好地址对齐的问题，升级到 clang 5.0 之后指令会被替换成 `vmovups` 问题就解决了**

既然问题的原因找到了，在生产系统上跑的 TiKV 可以放心了。但是使用装了 clang 4.9 的 MacOS 的同学在开发 TiKV 的时候还是尽量使用 `make unportable_release` 来编译并测试。

剩下的就是等等新版本的 Xcode 了吧。

## 写在最后

这个 Bug 跟下来之后，发现对自身的价值观又是一次冲击。上一次对 Linux Kernel 的信赖被遇到的各种 Kernel Panic 打破了。这次又打破了我对编译器的信赖。看来这年头没有什么能 100% 信赖的东西了。

当然必须要感谢一下 PingCAP 的同学们。响应非常快，尤其是 张金鹏 同学，为了这个问题我们两个真的是懵逼了好几天，他也是各种 Rust，C++ 的手写程序来抓虫。