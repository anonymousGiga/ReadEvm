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
* data: 程序执行时的输入数据；
* code: evm即将执行的指令；
* position: 记录执行指令的位置（就是上面code数组中的位置），就类似于pc计数器；
* return_range: 
* valids: 
* memory: evm的内存；
* stack: evm的栈；

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
所有的Opcode定义在opcode.rs中，分为内部操作码和外部操作码。

这里需要解释下什么是内部操作码，什么是外部操作码。

* 内部操作码：简单的说就是一些算术运算指令和加载、跳转指令。从rust-evm代码的角度来说就是core/evm自身就能实现的指令，其执行在core/src/eval中实现。
* 外部操作码：这里指address、sha256、create、call等和合约执行相关的指令。从rust-evm代码的角度来说，其执行在runtime/src/eval中实现。

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

从上面的代码可以看出，最终是调用到step函数。

下面我们重点看看step函数，其实现如下：
```
pub fn step(&mut self) -> Result<(), Capture<ExitReason, Trap>> {
		let position = *self
			.position
			.as_ref()
			.map_err(|reason| Capture::Exit(reason.clone()))?;

		match self.code.get(position).map(|v| Opcode(*v)) {
			Some(opcode) => match eval(self, opcode, position) {
				Control::Continue(p) => {
					self.position = Ok(position + p);
					Ok(())
				}
				Control::Exit(e) => {
					self.position = Err(e.clone());
					Err(Capture::Exit(e))
				}
				Control::Jump(p) => {
					self.position = Ok(p);
					Ok(())
				}
				Control::Trap(opcode) => {
					self.position = Ok(position + 1);
					Err(Capture::Trap(opcode))
				}
			},
			None => {
				self.position = Err(ExitSucceed::Stopped.into());
				Err(Capture::Exit(ExitSucceed::Stopped.into()))
			}
		}
	}
```
在该函数中，首先会通过根据position指示的位置从code数组中取出即将执行的操作码，然后将操作码输入到eval(self, opcode, position)函数执行（在core/src/eval/mod.rs）中。对于内部操作码（上面1.3提到过），起执行的返回结果一般是Control::Continue/Control::Exit/Control::Jump；**而对于内部操作码，返回的将是Control::Trap，然后会返回Err(Capture::Trap(opcode))，这是因为外部操作码的执行和链的状态相关，其执行函数在runtime/src/eval/mod.rs中实现。**

