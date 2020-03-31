#### Node.js 不是啥
* 不是 web 框架
* 不是编程语言

#### Node.js 是啥
* 是技术平台（融合多种技术）
  * V8 引擎 
  * libuv
  * C/C++ 实现的诸多库

#### Node.js bindings
> 让 JS 和 C/C++ 通信。具体表现为 JS 间接调用 C++ 的库。C++ 也可以执行 js 的回调函数

1. Node.js 对C++ 的库进行封装，比如将 http_server 封装成 http_parser_bindings.cpp 
2. Node.js 将封装结果 编译为 .node 文件
3. JS 代码可以直接 require 这个 .node 文件
4. 上述过程即为 Node.js 对一个C++ 库的 binding 过程。

#### libuv
> 跨平台的异步I/O库
