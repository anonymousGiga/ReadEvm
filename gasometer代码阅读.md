
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

