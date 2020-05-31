## 基本概念
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

#### [调试](https://xiedaimala.com/tasks/016e61fe-4ce8-4987-9d52-68c203a66ae2/video_tutorials/7dc543f8-1522-4615-809e-a4134aa29663)
* 作用:查看一个对象的属性和方法
```
node --inspect-brk index.js
```
---

## Node.js 的 HTTP 模块
#### [request 事件可能被触发多次](https://nodejs.org/docs/latest-v13.x/api/http.html#http_event_request)
> Emitted each time there is a request. There may be multiple requests per connection (in the case of HTTP Keep-Alive connections).
```
const http = require('http')
const server = http.createServer()
server.on('request', (request, response) => {
    console.log(response)
})
server.listen(8888)
```
---
## Node.js 的事件机制
#### [emitter.emit](https://nodejs.org/docs/latest-v13.x/api/events.html#events_emitter_emit_eventname_args)
> 是同步函数，不是异步的。Synchronously calls each of the listeners registered for the event named eventName, in the order they were registered, passing the supplied arguments to each.
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
#### one 和 emit 的具体机制 
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
> 取消订阅，必须传入回调函数名（即引用）
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
结果报错：throw new ERR_INVALID_ARG_TYPE('listener', 'Function', listener);
> 函数名代表 on 时的内存引用，所以 off 的第二个参数如果是箭头函数，则取消订阅失败
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
---
## Node.js 的模块机制
#### 缓存
```
// module1.js
console.log(1);
module.exports=()=>{console.log('module1 function')};
console.log(2);

// module2.js
console.log('before require');
const module_fn=require('./module1');
module_fn();
console.log('after require');
require('./module1');
module_fn();

node module2.js
```
执行结果
```
before require
1
2
module1 function
after require
module1 function
```
> 当 module1 第一次被 require 时，会执行 module1.js 代码，而第二次被 require 时，不会再执行 module1.js 代码

> 之所以这么做，是为了节省文件读取IO。第二次的 require 直接从缓存里拿数据
```
// module1.js
module.exports=()=>{console.log('module1 function')};

// module2.js
console.log(require);
const module_fn=require('./module1');
console.log(require);

node module2.js
```
执行结果
```
{cache{module2...}}

{cache{module2...,module1}} // 缓存了 module1
```	
#### exports 是指向 module.exports 的引用
```
// module1.js

// exports=module.exports; 这一句实际被执行了
exports.a=1;
console.log(module.exports);

// module2.js
const b=require('./module2');
console.log(b);

node module2.js
```
执行结果
```
{ a: 1 }
{ a: 1 }
```
#### 循环引用
[参考](https://zhuanlan.zhihu.com/p/111989060)
1. module.exports={'name':'peiqi'} 是同步执行的
2. require('./a') 时，会先检查缓存中有无 a.js 对应 module.export。若有，则直接读取 export 值。若没有，再去读取该文件（会执行其所有代码）
3. 修改某个 moduled 的 export 值，缓存中对应 的 export 也会随之更新
```
// file a.js

// 初始化时,a.js 的 module 的 export={}
module.exports={'name':'peiqi'} // a.js 的 module 的 export 值为 {'name':'peiqi'},将其放入缓存
const b = require('./b.js')
console.log(b)
module.exports.aTest = '777'
```
```
// file b.js

const a = require('./a.js') // 直接读取缓存中的 a.js 的 module 的 export 值，不执行 a.js 代码
console.log(a)
console.log(require.cache);
module.exports.bTest = '666'// 修改了 b.js 的 module 的 export 值，修改后的结果将被放入缓存
console.log(require.cache);// 缓存的 a.js 对应 module.export 已发生了变化

// 这里的 setTimeout 本身是同步执行的，其回调函数在 1s 后执行。注意应该把 a.js,b.js 合起来看成一个完整的 js 代码段
setTimeout(() => {
    console.log(require.cache);// 此时 a.js 已经解析完毕， 对应的 module.export 已发生了变化，导致缓存随之更新
    console.log(a)// 从缓存中读取 a.js 对应的 module.export
} , 1000)
```
执行结果
```
{ name: 'peiqi' }
{module-a:exports: { name: 'peiqi' },module-b:exports: {}}
{module-a:exports: { name: 'peiqi' },module-b:exports: { bTest: '666' }}
{ bTest: '666' }
{module-a:exports:{ name: 'peiqi', aTest: '777' },module-b:exports: { bTest: '666' }}
{ name: 'peiqi', aTest: '777' }
```
* 如果把上面的 module.exports.aTest = '777' 改为 module.exports = {}。执行结果为
```
{ name: 'peiqi' }
{module-a:exports: { name: 'peiqi' },module-b:exports: {}}
{module-a:exports: { name: 'peiqi' },module-b:exports: { bTest: '666' }}
{ bTest: '666' }
{module-a:exports:{},module-b:exports: { bTest: '666' }} // 缓存更新了。说明 a.js 的module.export 和缓存的 module-a:exports 始终指向同一片内存空间（应该是 node.js 做了额外的绑定）
{ name: 'peiqi'} // 虽然缓存更新了，但是由于 a 与 require.module-a:exports 已经不指向同一片内存空间。所以a的值不发生变化。
```
---

## Node.js 的异步
#### 异步
> CPU 告诉硬盘去读取数据，然后CPU执行其他计算任务，直到硬盘将数据送至内存，CPU才开始处理数据
#### IO
> 磁盘读写,网络请求,内存读写，都算IO
---

## Node.js 的 stream
#### stream 有什么用
> 不使用 stream 情况下，把 ./big_file.txt 内容 输出到 response。需要先把 ./big_file.txt 内容全部读出到内存中，在将该数据写入 response
```
const fs = require('fs')
const http = require('http')

const server = http.createServer()
server.on('request', (request, response) => {

    console.log(response.finished);

    fs.readFile('./big_file.txt', (error, data) => {

        // 内存中存有完整的 data
        console.log(data);
        if (error) throw error
        response.end(data)
        console.log('done')
    })
})
server.listen(8888)
console.log('8888')
```
> 而使用 stream,省去了“将读出数据放入内存”这一步。直接边读边写，因此大大减少了进程的内存占用
```
const http = require('http')
const fs = require('fs')
const server = http.createServer()
server.on('request', (request, response) => {

    const stream = fs.createReadStream('./big_file.txt')
    stream.pipe(response)

})

server.listen(8888)
```

#### readStream.pipe(writeStream)
> The readable.pipe() method attaches a Writable stream to the readable

> 注意 pipe 只是实现 stream 传递的一个 API。用事件机制也可以实现 stream 的传递。
```
const http = require('http')
const fs = require('fs')
const server = http.createServer()
server.on('request', (request, response) => {

    const stream =fs.createReadStream('./big_file.txt')

    stream.pipe(response)
})

server.listen(8888)
```
等价于
```
const http = require('http')
const server = http.createServer()
server.on('request', (request, response) => {

    const fs = require('fs')
    const stream =fs.createReadStream('./big_file.txt')

    // stram 一有新数据就传给 response
    stream.on('data',(chunk)=>{
        response.write(chunk)
    })

    // stream 被关闭时，也关闭 response
    stream.on('end',()=>{
        response.end()
    })
})

server.listen(8888)
```
#### ReadableStream 
	* Event: 'data',Event: 'end'
	```
	const http = require('http')
	const fs = require('fs')
	const server = http.createServer()
	server.on('request', (request, response) => {

	    const stream =
		fs.createReadStream('./big_file.txt')

	    stream.pipe(response)

	    // ReadableStream 有数据可读，则触发回调函数
	    stream.on('data', (data) => {
		console.log(`读取${data}`);
	    });
	    stream.on('end',()=>{
		console.log('read end');
	    })

	})

	server.listen(8888)
	```
#### WritableStream
	* [writable.write()](https://nodejs.org/docs/latest-v13.x/api/stream.html#stream_writable_write_chunk_encoding_callback)
	> The writable.write() method writes some data to the stream, and calls the supplied callback once the data has been fully handled.
	* [Event: 'drain'](https://nodejs.org/docs/latest-v13.x/api/stream.html#stream_event_drain)
	> If a call to stream.write(chunk) returns false, the 'drain' event will be emitted when it is appropriate to resume writing data to the stream. 及数据流出现空档（read faster than write）,此时需要触发 drain 来进行额外的操作以恢复数据流的高效运行。处理高速数据流时才需要。
	* [cork](https://nodejs.org/docs/latest-v13.x/api/stream.html#stream_writable_cork)
	> 暂时不需要了解

#### stream 的种类
1. Readable:可读
	* paused:默认状态。添加data事件监听或pipe(),resume()后变为flow态
	* flow:删除data事件监听或pause()后变为paused态
2. Writable:可写
3. Duplex:双向读写（每端都可读写，但不交叉）
4. Transform:把 sass 变成 css，把 es6 变成 es5
---
## Node.js 的 [buffer](https://nodejs.org/dist/latest-v6.x/docs/api/buffer.html#buffer_buffer)
1. 8-bit unsigned integer:8位无符号整数，范围是二进制的00000000-11111111，即十进制的0-255。
2. Buffer 类是 JavaScript 语言内置的 Uint8Array 类的子类。
3. buffer 的出现是因为早期的 Node.js 作为一门服务器语言无法处理二进制数据流，于是封装并优化了js语言的 Uint8Array API。Prior to the introduction of TypedArray in ECMAScript 2015 (ES6), the JavaScript language had no mechanism for reading or manipulating streams of binary data. 
4. buffer里的元素是二进制数
```
// Creates a Buffer containing [0x1, 0x2, 0x3].
const buf4 = Buffer.from([1, 2, 3]);
```
5. buffer 申请的内存是在 V8 的 heap 之外连续分配的一片内存空间
#### 自定义一个stream
* Writable
```
const Writable = require('stream').Writable;

const outStream = new Writable({
    write(chunk, encoding, callback) {
        console.log(chunk.toString());
        callback()
    }
});

process.stdin.pipe(outStream)
```
* Readable
```
const {Readable} = require('stream')

const inStream = new Readable()

inStream.push('ABCDEFGHIJKLM')
inStream.push('NOPQRSTUVWXYZ')

inStream.push(null) // No more data

// 以下代码等价于 inStream.pipe(process.stdout)
inStream.on('data', (chunk)=>{
    process.stdout.write(chunk)
})
```
> 调用 inStream 的 read 方法实现
```
const { Readable } = require("stream");

const inStream = new Readable({
    read(size) {
        const char = String.fromCharCode(this.currentCharCode++)
        this.push(char);
        // console.log(`推了 ${char}`)
        if (this.currentCharCode > 90) { // Z
            this.push(null);
        }
    }
})

inStream.currentCharCode = 65 // A

inStream.pipe(process.stdout)
// stdout 会不停调用 inStream 的 read。
/***
 * In flowing mode, readable.read() is called automatically until the internal buffer is fully drained.
 */
```
* transform
> 把用户输入的字符全部转换成大写后再输出（输出由用户输入决定）
```
const { Transform } = require("stream");

const upperCaseTr = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
});

process.stdin.pipe(upperCaseTr).pipe(process.stdout);
```
#### Node.js 内置的 transform 流
```
// gzip.js
const fs = require("fs");
const zlib = require("zlib");
const file = process.argv[2];
const crypto = require("crypto");


const { Transform } = require("stream");

const reportProgress = new Transform({
  transform(chunk, encoding, callback) {
    process.stdout.write(".");
    callback(null, chunk);
  }
});


fs.createReadStream(file)
  .pipe(crypto.createCipher("aes192", "123456"))
  .pipe(zlib.createGzip())
  .pipe(reportProgress)
  .pipe(fs.createWriteStream(file + ".gz"))
  .on("finish", () => console.log("Done"));
```
#### [Backpressuring in Streams](https://nodejs.org/en/docs/guides/backpressuring-in-streams/)
* 当数据的写入较慢时，会发生数据的堆积现象。
> There are instances where a Readable stream might give data to the Writable much too quickly — much more than the consumer can handle!

> When that occurs, the consumer will begin to queue all the chunks of data for later consumption. The write queue will get longer and longer, and because of this more data must be kept in memory until the entire process has completed.

> Writing to a disk is a lot slower than reading from a disk, thus, when we are trying to compress a file and write it to our hard disk, backpressure will occur because the write disk will not be able to keep up with the speed from the read.
* 数据堆积的坏处
	1. Slowing down all other current processes
	2. A very overworked garbage collector
	3. Memory exhaustion
* 在 UNIX,TCP 协议中，采用流控制来解决这个问题。而 Node.js 用 stream 来解决
* 用 stream 解决 Backpressuring 的流程
	1. pipe 函数将 backpressure closures 发送给 event triggers.
	2. data buffer has exceeded the highWaterMark or the write queue is currently busy, Writable.write() return false.
	3. backpressure system kicks in(?) the return value,导致 ReadableStream 由 flow 态变为 pause 态(可能是调用了 pause 方法)
	4. 当积压的 buffer 被清空了，就会触发 drain 事件，从而导致 ReadableStream 由 pause 态变为 flow 态(调用 resume 方法)
* lifeCycle of pipeline
```
                                                     +===================+
                         x-->  Piping functions   +-->   src.pipe(dest)  |
                         x     are set up during     |===================|
                         x     the .pipe method.     |  Event callbacks  |
  +===============+      x                           |-------------------|
  |   Your Data   |      x     They exist outside    | .on('close', cb)  |
  +=======+=======+      x     the data flow, but    | .on('data', cb)   |
          |              x     importantly attach    | .on('drain', cb)  |
          |              x     events, and their     | .on('unpipe', cb) |
+---------v---------+    x     respective callbacks. | .on('error', cb)  |
|  Readable Stream  +----+                           | .on('finish', cb) |
+-^-------^-------^-+    |                           | .on('end', cb)    |
  ^       |       ^      |                           +-------------------+
  |       |       |      |
  |       ^       |      |
  ^       ^       ^      |    +-------------------+         +=================+
  ^       |       ^      +---->  Writable Stream  +--------->  .write(chunk)  |
  |       |       |           +-------------------+         +=======+=========+
  |       |       |                                                 |
  |       ^       |                              +------------------v---------+
  ^       |       +-> if (!chunk)                |    Is this chunk too big?  |
  ^       |       |     emit .end();             |    Is the queue busy?      |
  |       |       +-> else                       +-------+----------------+---+
  |       ^       |     emit .write();                   |                |
  |       ^       ^                                   +--v---+        +---v---+
  |       |       ^-----------------------------------<  No  |        |  Yes  |
  ^       |                                           +------+        +---v---+
  ^       |                                                               |
  |       ^               emit .pause();          +=================+     |
  |       ^---------------^-----------------------+  return false;  <-----+---+
  |                                               +=================+         |
  |                                                                           |
  ^            when queue is empty     +============+                         |
  ^------------^-----------------------<  Buffering |                         |
               |                       |============|                         |
               +> emit .drain();       |  ^Buffer^  |                         |
               +> emit .resume();      +------------+                         |
                                       |  ^Buffer^  |                         |
                                       +------------+   add chunk to queue    |
                                       |            <---^---------------------<
                                       +============+
```
---
## [Node.js 的 Child Process 模块](https://nodejs.org/docs/latest-v13.x/api/child_process.html)
#### shell
* shell 不是进程，而是一个命令行环境（容器）
#### child_process.exec
> Spawns a shell then executes the command within that shell, buffering any generated output. The command string passed to the exec function is processed directly by the shell and special characters (vary based on shell) need to be dealt with accordingly:
```
const util = require('util');
const exec = util.promisify(require('child_process').exec);

async function lsExample() {
  const { stdout, stderr } = await exec('ls');
  console.log('stdout:', stdout);
  console.error('stderr:', stderr);
}
lsExample();
```
#### child_process.execFile
> The child_process.execFile() function is similar to child_process.exec() except that it does not spawn a shell by default. Rather, the specified executable file is spawned directly as a new process making it slightly more efficient than child_process.exec().The same options as child_process.exec() are supported. Since a shell is not spawned, behaviors such as I/O redirection and file globbing are not supported.
```
const { execFile } = require('child_process');
const child = execFile('node', ['--version'], (error, stdout, stderr) => {
  if (error) {
    throw error;
  }
  console.log(stdout);
});
```
#### [child_process.spawn](https://nodejs.org/docs/latestv13.x/api/child_process.html#child_process_child_process_spawn_command_args_options)
* 用法基本与 execFile 相同
* 没有回调函数，只能通过流获取事件结果（自然也没有 maxBuffer 的限制）
```
const { spawn } = require('child_process');
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.error(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});
```
#### [child_process.fork](https://nodejs.org/docs/latest-v13.x/api/child_process.html#child_process_child_process_fork_modulepath_args_options)
* 只创建 Node 进程
* 会建立 IPC channel 以实现父子进程通信[](https://nodejs.org/docs/latest-v13.x/api/child_process.html#child_process_subprocess_send_message_sendhandle_options_callback)
```
// parent.js
const cp = require('child_process');
const n = cp.fork(`${__dirname}/sub.js`);

n.on('message', (m) => {
  console.log('PARENT got message:', m);
});

// Causes the child to print: CHILD got message: { hello: 'world' }
n.send({ hello: 'world' });
```
```
// sub.js
process.on('message', (m) => {
  console.log('CHILD got message:', m);
});

// Causes the parent to print: PARENT got message: { foo: 'bar', baz: null }
process.send({ foo: 'bar', baz: NaN });
```
```
// node parent.js
```
---
## [Node.js 的 Util 模块](https://nodejs.org/api/util.html)
#### [callbackify](https://nodejs.org/api/util.html#util_util_callbackify_original)
返回回调函数
```
const util = require('util');

async function fn() {
  return 'hello world';
}
const callbackFunction = util.callbackify(fn);

callbackFunction((err, ret) => {
  if (err) throw err;
  console.log(ret);
});
```
#### promisify
同上理
---
## [Node.js 的 crypto 模块](https://nodejs.org/api/crypto.html)
#### [PBKDF2-](https://nodejs.org/api/crypto.html#crypto_crypto_pbkdf2_password_salt_iterations_keylen_digest_callback)
> Password-Based Key Derivation Function 2.
```
crypto.pbkdf2(password, salt, iterations, keylen, digest, callback)
```
* iterations
> The iterations argument must be a number set as high as possible. The higher the number of iterations, the more secure the derived key will be, but will take a longer amount of time to complete.
* salt
> The salt should be as unique as possible. It is recommended that a salt is random and at least 16 bytes long. 
* derivedKey
> 我们最终得到的加密后的秘钥
```
const crypto = require('crypto');
crypto.pbkdf2('secret', 'salt', 100000, 64, 'sha512', (err, derivedKey) => {
  if (err) throw err;
  console.log(derivedKey.toString('hex'));  // '3745e48...08d59ae'
});
```


