# 阅读executor.rs

# 1 数据结构
## Accessed 
1、定义：
```
pub struct Accessed {
	pub accessed_addresses: BTreeSet<H160>,
	pub accessed_storage: BTreeSet<(H160, H256)>,
}
```
2、功能描述：
主要记录一组H160账户，用于做栈访问存储的管理。

## StackSubstateMetadata
1、定义：
```
pub struct StackSubstateMetadata<'config> {
	gasometer: Gasometer<'config>,
	is_static: bool,
	depth: Option<usize>,
	accessed: Option<Accessed>,
}
```
2、功能描述：
记录栈操作时对应的元数据，主要包括：
* gasometer，记录gas使用的具体情况；
* depth，栈的深度；
* accessed，栈访问的存储。

## StackExecutor

1、定义：
```
pub struct StackExecutor<'config, 'precompiles, S, P> {
	config: &'config Config,
	state: S,
	precompile_set: &'precompiles P,
}
```

2、功能描述：
StackExecutor是真正用来执行动作的执行器。
