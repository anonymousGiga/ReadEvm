rust-evm/core文件夹下的代码主要是用来实现evm machine，下面我们来一一分析。

# 1 evm machine定义
machine的定义如下：
```
pub struct Machine {
	/// Program data.
	data: Rc<Vec<u8>>,
	/// Program code.
	code: Rc<Vec<u8>>,
	/// Program counter.
	position: Result<usize, ExitReason>,
	/// Return value.
	return_range: Range<U256>,
	/// Code validity maps.
	valids: Valids,
	/// Memory.
	memory: Memory,
	/// Stack.
	stack: Stack,
}
```
各部分解释如下：
* data: 程序执行时的数据；
* code: 程序执行时的code；
* position: 记录执行时的位置
* return_range: 
* valids: 
* memory: 
* stack: 

下面我们看具体的每一部分。

## 1.1 stack
stack是evm machine中很重要的数据结构，实现在stack.rs中，其定义如下：
```
pub struct Stack {
	data: Vec<H256>,
	limit: usize,
}
```
stack实现的操作如下：
```
	pub fn new(limit: usize) -> Self;
	pub fn limit(&self) -> usize;
	pub fn len(&self) -> usize;
	pub fn is_empty(&self) -> bool;
	pub fn data(&self) -> &Vec<H256>;
	pub fn pop(&mut self) -> Result<H256, ExitError>;
	pub fn push(&mut self, value: H256) -> Result<(), ExitError>;
	pub fn peek(&self, no_from_top: usize) -> Result<H256, ExitError>;
	pub fn set(&mut self, no_from_top: usize, val: H256) -> Result<(), ExitError>;
```
可见，stack就是实现了栈的正常操作。

## 1.2 Valids
Valids的作用主要用来从代码映射有效跳转目标。其实现主要valids.rs中。
Valids的定义如下：
```
pub struct Valids(Vec<bool>);
```
创建Valids的代码如下：
```
	/// Create a new valid mapping from given code bytes.
	pub fn new(code: &[u8]) -> Self {
		let mut valids: Vec<bool> = Vec::with_capacity(code.len());
		valids.resize(code.len(), false);

		let mut i = 0;
		while i < code.len() {
			let opcode = Opcode(code[i]);
			if opcode == Opcode::JUMPDEST {
				valids[i] = true;
				i += 1;
			} else if let Some(v) = opcode.is_push() {
				i += v as usize + 1;
			} else {
				i += 1;
			}
		}

		Valids(valids)
	}
```
从上面的代码中可以看出，valids实际上就是标注opcode，如果是jmpdest，则将其标注为true。

## 1.3 Opcode
所有的Opcode定义在opcode.rs中，分为内部指令和外部指令，此处不一一列出。

## 1.4 memory
在memory.rs中定义了memory的数据结构，主要用来模拟内存的实现，用于存储数据。

# 2 evm的执行

evm的执行的核心代码在core/src/lib.rs下面的run函数，其定义如下：
```
	pub fn run(&mut self) -> Capture<ExitReason, Trap> {
		loop {
			match self.step() {
				Ok(()) => (),
				Err(res) => return res,
			}
		}
	}
```

从上面的代码可以看出，最终是调用到step函数。在step函数中，会去除opcode，然后匹配table表找到对应的操作（match eval），执行指令即可。

至于具体的指令匹配后执行的代码，在eval文件夹中。
