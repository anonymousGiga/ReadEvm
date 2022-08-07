我们在前面说过runtime属于rust-evm中一个非常重要的组件，本篇文章就主要来分析分析其源码（runtime目录下）。

# 1 数据结构
runtime的定义如下：
```
pub struct Runtime<'config> {
	machine: Machine,
	status: Result<(), ExitReason>,
	return_data_buffer: Vec<u8>,
	context: Context,
	_config: &'config Config,
}
```
各字段的含义如下：
* machine，machine的实现在我们之前分析的过的代码core中，machine取操作码，然后通过stack和memory配合执行指令。不过machine定义的执行动作中，具体的执行的实现时，内部指令的执行实现在core中定义，而外部指令的实现则是在runtime中定义。调用外部指令时，通过trap的方式来调用runtime中定义的实现。
* status，记录执行的结果，如果是结束则记录结束的原因。
* return_data_buffer，记录指令执行结束的返回数据。
* context，记录runtime执行的上下文，主要包括执行的地址、调用者、涉及的value。
* config，记录创建runtime时的配置，在rust-evm的runtime中没有使用。

# 2 功能函数
runtime中主要定义了以下函数：
```
pub fn new(
		code: Rc<Vec<u8>>,
		data: Rc<Vec<u8>>,
		context: Context,
		config: &'config Config,
	) -> Self;
  
pub fn machine(&self) -> &Machine;
pub fn context(&self) -> &Context;
pub fn step<'a, H: Handler>(
		&'a mut self,
		handler: &mut H,
	) -> Result<(), Capture<ExitReason, Resolve<'a, 'config, H>>>;
 pub fn run<'a, H: Handler>(
		&'a mut self,
		handler: &mut H,
	) -> Capture<ExitReason, Resolve<'a, 'config, H>>;
```
这里我们重点分析下step函数，run函数是循环调用step函数，其它函数则从字面意思就很容易看明白其实现的功能。
step函数中只有一行代码：
```
	step!(self, handler, return Err; Ok)
```
通过宏step！执行，传入的参数是runtime对象本身、实现了trait Handler的trait对象handler、返回类型Err、Ok。下面我们来一一介绍。

## 2.1 trait Handler
Handler trait定义了runtime执行外部指令时需要的和链相关的操作函数，换句话说，runtime执行具体的外部指令时，需要调用到这些函数。该trait的定义代码在文件runtime/src/handler.rs中。

## 2.2 step宏
下面我们就逐行分析一下step宏的实现：
```
macro_rules! step {
	( $self:expr, $handler:expr, $return:tt $($err:path)?; $($ok:path)? ) => ({
    
    
		if let Some((opcode, stack)) = $self.machine.inspect() {
                        // event宏是用来打开trace开关后，调试使用，正常是空白，所以我们可以当做event！宏都是空白，是废代码，没有意义。
			event!(Step {
				context: &$self.context,
				opcode,
				position: $self.machine.position(),
				stack,
				memory: $self.machine.memory()
			});

			//调用handler的代码，对要执行的指令做验证，主要是验证gas是否超出，如果超过，则返回失败。
			//详细的pre_validate的执行我们可以等到后面分析外部传入的具体的该函数时来看。
			match $handler.pre_validate(&$self.context, opcode, stack) {
				Ok(()) => (),
				Err(e) => {
					$self.machine.exit(e.clone().into());
					$self.status = Err(e.into());
				},
			}
		}
                
		//匹配status（上面的pre_validate如果执行失败会exit，self.status中存着对应的状态），如果失败则返回，终止当前指令的执行。
		match &$self.status {
			Ok(()) => (),
			Err(e) => {
				#[allow(unused_parens)]
				$return $($err)*(Capture::Exit(e.clone()))
			},
		}
                
		//调用machine的step函数执行，在machine的step函数中主要执行的是内部指令，如果遇到外部执行会直接返回trap的error。
		let result = $self.machine.step();
		
		// trace代码，当成无意义代码
		event!(StepResult {
			result: &result,
			return_value: &$self.machine.return_value(),
		});
		
		//下面主要上面machine执行后的结果进行处理，对于内部指令，返回的应该是Ok和Err(Capture::Exit(e))，对于外部指令，则会返回Err(Capture::Trap(opcode))
		match result {
			Ok(()) => $($ok)?(()),
			Err(Capture::Exit(e)) => {
				$self.status = Err(e.clone());
				#[allow(unused_parens)]
				$return $($err)*(Capture::Exit(e))
			},
			//匹配此项说明是外部指令，在此处执行
			Err(Capture::Trap(opcode)) => {
			        // 下面都是执行外部指令
				match eval::eval($self, opcode, $handler) {
					eval::Control::Continue => $($ok)?(()),
					eval::Control::CallInterrupt(interrupt) => {
						let resolve = ResolveCall::new($self);
						#[allow(unused_parens)]
						$return $($err)*(Capture::Trap(Resolve::Call(interrupt, resolve)))
					},
					eval::Control::CreateInterrupt(interrupt) => {
						let resolve = ResolveCreate::new($self);
						#[allow(unused_parens)]
						$return $($err)*(Capture::Trap(Resolve::Create(interrupt, resolve)))
					},
					eval::Control::Exit(exit) => {
						$self.machine.exit(exit.clone().into());
						$self.status = Err(exit.clone());
						#[allow(unused_parens)]
						$return $($err)*(Capture::Exit(exit))
					},
				}
			},
		}
	});
}
```




