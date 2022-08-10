
gasometer主要用来记录evm执行过程中gas的消耗情况，在rust-evm中，其对应的代码主要在文件夹gasometer中。

# 1 数据结构
gasometer的定义如下：
```
pub struct Gasometer<'config> {
	gas_limit: u64,
	config: &'config Config,
	inner: Result<Inner<'config>, ExitError>,
}
```
几个字段的含义如下：
* gas_limit：gas_limit的值；
* config：对应的evm对象的配置；
* inner：内部的记录gas使用情况的结构。

Inner定义如下：
```
struct Inner<'config> {
	memory_gas: u64,
	used_gas: u64,
	refunded_gas: i64,
	config: &'config Config,
}
```

# 2 相关函数
Gasometer的函数基本上就是对应gas的计算。这里重点看一下交易的gas记录，对应的函数是

```
pub fn record_transaction(&mut self, cost: TransactionCost) -> Result<(), ExitError>
```
gas的计算公式如下：
gas_used = 基础的gas（以太坊中一般是21000000）+ 非零字节长度 * 非零字节单位gas + 零字节长度 * 零字节单位gas + 地址长度 * 地址长度单位gas + 内存长度 * 内存单位gas

在rust-evm实现中，基础的gas和这几个单位gas都是配置在config中的。


