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
主要记录一组H160账户，用于做权限管理
