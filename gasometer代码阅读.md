
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

