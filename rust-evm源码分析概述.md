# 1 rust-evm的整体结构
rust-evm主要由几大组件组成，分别如下：
- machine：执行指令的具体的虚拟机实体，其代码主要在core文件夹中。
- runtime：包装了machine，实现具体的evm执行功能，其中evm外部指令的执行动作在这部分代码中实现，evm内部指令执行动作在core中实现，代码在runtime目录中。
- gasometer：记录合约在evm执行过程中的gas使用情况的实体，代码在gasometer中。
- stack executor：提供合约创建和合约调用相关的接口，由链上调用；当有合约创建或者是调用时，在处理函数中创建下面的gasometer和runtime对象，然后完成具体的执行动作，代码主要在src/executor/stack/executor.rs中。

上面我们没有讲到src/executor/stack/memory.rs中的MemoryStackAccount, MemoryStackState, MemoryStackSubstate等，是因为memory中的内容相当于是一个链在内存中的模拟，在该工程中主要是用来支持benches中的测试，在实际的链需要移植rust-evm时，这部分内容的实现是和链相关。

# 2 执行过程
下面我们就结合合约创建和调用来简单看下几大组件的作用。

## 2.1 合约创建


## 2.2 合约执行

