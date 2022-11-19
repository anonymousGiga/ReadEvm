最近确实太忙没有来得及写笔记了，实在抱歉。今天忙里偷闲来聊一聊一条rust实现的非evm链如何集成rust-evm以支持solidity合约的运行。
前面我们写了几篇文章简单阅读了rust-evm的源码，除了executor部分，我们基本上都解读过。

最近确实太忙没有来得及写笔记了，实在抱歉。今天忙里偷闲来聊一聊一条rust实现的非evm链如何集成rust-evm以支持solidity合约的运行。前面我们写了几篇文章简单阅读了rust-evm的源码，除了executor部分，我们基本上都解读过。

关于集成rust-evm主要有以下几点：

1、实现trait StackState

2、实现trait Backend和 trait ApplyBackend

3、实现Creat、Create2、Call函数
