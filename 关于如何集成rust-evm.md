最近确实太忙没有来得及写笔记了，实在抱歉。今天忙里偷闲来聊一聊一条rust实现的非evm链如何集成rust-evm以支持solidity合约的运行。
前面我们写了几篇文章简单阅读了rust-evm的源码，除了executor部分，我们基本上都解读过。

最近确实太忙没有来得及写笔记了，实在抱歉。今天忙里偷闲来聊一聊一条rust实现的非evm链如何集成rust-evm以支持solidity合约的运行。前面我们写了几篇文章简单阅读了rust-evm的源码，除了executor部分，我们基本上都解读过。

关于集成rust-evm主要有以下几点：

1、实现trait StackState
在此trait中，相关的函数主要是和链相关联，以便executor在执行时能够获取和修改链的一些状态，例如transfer就是操作链的逻辑。

2、实现trait Backend
此trait中主要是实现evm中需要读取链的相关的操作，根据函数名和链的具体情况实现逻辑即可。

3、trait ApplyBackend
此trait的函数一般是在executor执行后，然后需要修改应用于链的逻辑时，执行的函数。此trait的实现不是必须，需要看前面的StackState trait的实现，如果StackState trait的实现是直接和链的操作关联，也可以不用实现此trait（即在executor执行时就将链对应的状态修改了，frontier就是此种方式）；如果StackState的实现是以一些中间态在内存中存在（例如直接使用Rust-evm中的memoryStackState实现），则需要实现此trait，在最后executor执行完成后调用apply函数将内存中的状态修改应用到链上。


4、实现Creat、Create2、Call函数
合约创建和调用的接口，一般的实现逻辑是计算fee、创建executor并执行（executor执行时会使用到上面的那些trait，从而和链关联上）、然后调用apply将修改应用到链上。

以上几步，基本上就将rust-evm集成到自己的链上了。但是要想兼容ethereum的dapp，光完成以上工作还远远不够。以上只是能满足执行编译好的合约字节码。要兼容ethereum的dapp，则需要支持ethereum的rpc，具体实现思路就是接受ethereum的接口，然后将其翻译成自己链上对应的操作，然后再执行就好。一般的方式是实现一个gateway来支持此需求。
