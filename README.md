[Node.js](在线运行工具)

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

#### v8
* 用于运行 js
* 垃圾回收，用于重复利用内存
* v8 是多线程的，但是执行 js 放在一个单独线程里
* 自带 eventloop,但是 Node.js 没有用它，而是基于 linuv 自己做了一个
---
#### Event
* 计时器到期
* 告诉 js 文件可以读取了/出错了
* 告诉 js 服务器响应了

#### [emitter.emit](https://nodejs.org/docs/latest-v13.x/api/events.html#events_emitter_emit_eventname_args)
* 是同步函数，不是异步的
> Synchronously calls each of the listeners registered for the event named eventName, in the order they were registered, passing the supplied arguments to each.
```
const EventEmitter = require('events');
const myEmitter = new EventEmitter();

myEmitter.on('event', ()=>{
  console.log('1');
});

console.log(2);

myEmitter.emit('event');

console.log(3);
```
执行顺序为 2 1 3

#### one 和 emit 
1. emitter.on('event',callback);emitter.emit('event')后 callback才会执行
2. 多次注册事件，会形成一个事件队列，emit 触发当前事件队列中的所有事件，不会触发 emit 后新注册的事件
```
const EventEmitter = require('events');
const myEmitter = new EventEmitter();

const fn=()=>{
	console.log(1);
	myEmitter.on('event', ()=>{
		console.log('hhh')
	});
}
myEmitter.on('event', fn);

myEmitter.emit('event');
myEmitter.emit('event');
myEmitter.emit('event');
```
结果是
```
1
1
hhh
1
hhh
hhh
```
注意
```
()=>{
		console.log('hhh')
	});
```
只有当该函数在事件队列中，且 emit('event') 时才会执行。

#### [emitter.off](https://nodejs.org/docs/latest-v13.x/api/events.html#events_emitter_off_eventname_listener)
* 取消订阅，必须传入回调函数名（即引用）
```
const EventEmitter = require('events');
const myEmitter = new EventEmitter();

const fn=()=>{
	console.log(1);
}
myEmitter.on('event', fn);

myEmitter.emit('event');
myEmitter.off('event');
myEmitter.emit('event');
```
> 结果报错：throw new ERR_INVALID_ARG_TYPE('listener', 'Function', listener);
* 函数名代表 on 时的内存引用，所以 off 的第二个参数如果是箭头函数，则取消订阅失败
```
const EventEmitter = require('events');
const myEmitter = new EventEmitter();

myEmitter.on('event', fn=()=>{
	console.log(1);
});

myEmitter.emit('event');

// 传入相同内容的箭头函数，但与 on 时的箭头函数不是同一片内存，所以取消订阅失败
myEmitter.off('event',()=>{
	console.log(1);
});
myEmitter.emit('event');
```
执行结果
```
1
1
```

#### Eventloop
[事件确定优先级后轮询](https://github.com/Hanqing1996/JavaScript-advance)

