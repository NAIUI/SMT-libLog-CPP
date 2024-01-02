# 高性能日志库

**为什么要实现非阻塞的日志库**

1. 如果是同步日志，那么每次产生日志信息时，就需要将这条日志信息完全写入磁盘后才会执行后续程序。而磁盘 IO 是比较耗时的操作，如果有大批量的日志信息需要写入就会阻塞网络库的工作。
2. 如果是异步日志，那么写日志消息只需要将日志的信息先进行存储，当累计到一定量或者经过一定时间时再将这些日志信息批量写入磁盘。而这个写入过程靠后台线程去执行，不会影响处理事件的其他线程。

**如何实现高性能的日志**

* 批量写入：攒够数据，一次写入，glog / muduo
* 唤醒机制：通知唤醒 `notify` + 超时唤醒 `wait_timeout`
* 锁的粒度：刷新磁盘时，日志接口不会阻塞。这是通过双队列实现的，前台队列实现日志接口，后台队列实现刷新磁盘。
* 内存分配：移动语义，避免深拷贝；双缓冲，前台后台都设有。

## 1. 日志写入逻辑

`fwrite` 函数原型

功能：向 buffer 中, 写入 count 个大小为 size 的对象到指定的流 stream。返回已写入对象的数量.

```c
 int fwrite(const void *buffer, size_t size, size_t count, FILE *stream )
```

`fwrite` 与 `write` 的区别

* fwrite 有缓存，write 没有缓存
* fwrite 是库函数，每次将数据写入用户缓冲区。等缓冲区满了，一次写入磁盘。或调用 fflush 刷新缓冲区；write 是系统调用，每次将数据写到磁盘，涉及用户态和内核态的切换。
* 批量写入的时候：应当减少调用 `write` 和 `fsync`，避免系统开销

可以用 `setbuf`设置用户缓冲区的大小

```c
 void setbuf(FILE *stream, char *buf);
```

fwrite 配合以下结合，将日志消息写入日志文件。

* `fflush`: 内部触发 `write` 把用户缓冲区数据刷新到内核缓冲区中
  int fflush( FILE *stream )
* `fsync`: 把内核缓冲区数据刷新到磁盘
  int fsync(int fd);

```C++
 #include <sstream>
 #include <unistd.h>
 
 using namespace std;
 
 int main () {
     FILE *file = fopen("0-fwrite_test.log", "wt");
     // char buffer[BUFSIZ];
     // setbuf(file, NULL);
 
     for (int i = 0; i < 10; ++i) {
         ostringstream oss;
         oss << "No." << i << "Root Error Message!\n";
         fwrite(oss.str().c_str(), 1, oss.str().size(), file);
         fflush(file);
     }
     // // 把数据刷到磁盘里
     // int fd = fileno(file); //把文件流指针转换成文件描述符
     // fsync(fd);
     // fclose(file);
 }
```

## 2. 日志一般框架

### 2.1 日志级别

日志输出级别应当在运行时可调，在必要的时候可以临时在线调整日志的输出级别。对于 muduo 库来说，调整日志的输出级别不需要重新编译，也不需要重启进程，只需要调用 `muduo::Logger::setLoglevel()`就能即时生效。

### 2.2 日志格式

muduo 库默认日志的格式是固定的，不需要运行时配置，这样可以节省每条日志解析格式字符串的开销，如果需要调整消息格式，直接修改代码重新编译即可。其格式包括

```text
 日期 时间 微秒 线程 级别 正文 源文件名:行号
```

日志消息格式的要点如下

* 尽量每条日志占一行。这样很容易用 awk, sed, grep 等命令工具快速联机分析日志。
* 时间戳精确到微秒。每条消息通过 `gettimeofday(2)`获得当前时间，该函数不是系统调用，因此不存在性能损失。
* 始终使用 GMT 时区。对于分布式系统而言，不需要本地时区转换。
* 打印线程 id。便于分析多线程程序的时序，也可以检测死锁。
* 打印日志级别。在线查错的时候优先查找 `ERROR` 日志，加速定位问题。
* 打印源文件名和行号。定位位置。

### 2.3 日志输出

本地文件

### 2.4 日志回滚

muduo 库的日志回滚没有采用文件改名，dmesg.log 始终是最新日志，便于编写及时解析日志的脚本

在性能方面，统计文件大小时， muduo 库在应用层增加当前写入日志的大小参数，无疑是一种巧妙的做法。

## 3. 日志库分析

### 3.1 异步日志机制

异步日志，用一个线程负责收集日志消息，并写入日志文件。其他业务线程只管往这个日志线程发送日志消息。

框架有一个日志缓冲队列来将日志前端的数据发送到后端（日志线程），这是一个典型的生产者与消费者模型。生产者前端不是将日志消息逐条分别传送给后端，而是将多条日志信息缓存拼成1个大的 buffer 发送给后端。同时，对于消费者后端线程来说，并不是每条日志消息写入都将其唤醒，而是当前端写满1个 buffer 的时候，才唤醒后端日志线程批量写入磁盘文件。

因此，后端日志线程的唤醒有两个条件

* buffer 写满唤醒：批量写入写满1个 buffer 后，唤醒后端日志线程，减少线程被唤醒的频率，降低系统开销。
* 超时被唤醒：为了及时将日志消息写入文件，防止系统故障导致内存中的日志消息丢失，超过规定的时间阈值，即使 buffer 未满，也会立即将 buffer 中的数据写入。

### 3.2 双缓冲机制

muduo 库采用的是双缓冲机制（Google C++日志库也是如此）。其基本思路是准备两个缓冲区，bufferA 和 bufferB。前端负责向 A 中写入日志消息，后端负责将 B 中的数据写入文件。当 bufferA 写满后，交换 A 和 B，此时让后端将 A 的数据写入文件，前端向 B 中写入新的日志消息，如此往复。这样在追加日志消息的时候不必等待磁盘 IO 操作，同时也避免了每条新日志消息都触发唤醒后端日志线程。

在源码 AsyncLogging.h 中，缓冲区 A 和 B，实际上采用的是缓冲区队列，分别对应前台日志缓冲队列 buffers 和后台日志缓冲队列 buffersToWrite。交换 A 和 B 时，只需要交换其指针的指向即可。

另外，采用队列可以提升缓冲区的容错性。考虑下面这种情况：前端写满 buffer，触发交换，后端还未将数据完全写入磁盘，此时前端又写满交换后的 buffer 了，再次触发交换机制，无 buffer 可用，造成阻塞。

### **3.3 前端日志写入**

`AsyncLogging::append()`：前端生成一条日志消息。

前端准备一个前台缓冲区队列 `buffers_`和两个 buffer。前台缓冲队列 `buffers_`用来存放积攒的日志消息。两个 buffer，一个是当前缓冲区 `currentBuffer`，追加的日志消息存放于此；另一个作为当前缓冲区的备份，即预备缓冲区 `nextBuffer`，减少内存的开销。

函数执行逻辑如下：

判断当前缓冲区 `currentBuffer_`是否已经写满

* 若当前缓冲区未满，追加日志消息到当前缓冲，这是最常见的情况
* 若当前缓冲区写满，首先，把它移入前台缓冲队列 `buffers_`。其次，尝试把预备缓冲区 `nextBuffer_`移用为当前缓冲，若失败则创建新的缓冲区作为当前缓冲。最后，追加日志消息并唤醒后端日志线程开始写入日志数据。

```c
 void AsyncLogging::append(const char* logline, int len) {
   // 多线程加锁，线程安全
   MutexLockGuard lock(mutex_);
 
   // 判断当前缓冲是否已经写满，批量数据的积攒
   // 1、当前缓冲未满，还能写入数据
   if (currentBuffer_->avail() > len) {
     // 追加日消息到当前缓冲
     currentBuffer_->append(logline, len); 
   }
 
   // 2、当前缓冲写满，两件事
   // 其一，将写满的当前缓冲的日志消息写入前台缓冲队列 buffers
   // 其二，追加日志消息到当前缓冲，唤醒后台日志落盘线程
   else {
     // 其一、当前缓冲移入(move)前台缓冲队列 buffers
     buffers_.push_back(std::move(currentBuffer_));
 
     // 判断预备缓冲是否写满
     // 2.1、预备缓冲未满，复用该缓冲区
     if (nextBuffer_) {
       // 预备缓冲移用为当前缓冲
       currentBuffer_ = std::move(nextBuffer_); 
     }
     // 2.2、预备缓冲写满（极少发生），重新分配缓冲
     // 原因：前端写入速度太快，一下子把两块缓冲都用完了
     else {
       // 重新分配 buffer，用作当前缓冲
       currentBuffer_.reset(new Buffer); 
     }
   
     // 其二、追加日志信息到当前缓冲，唤醒日志落盘线程
     currentBuffer_->append(logline, len);  
     cond_.notify(); 
   }
 }
```

### **3.4 后端日志落盘**

```C++
 void AsyncLogging::threadFunc() {
   latch_.countDown();
   // logFile 类负责将数据写入磁盘
   LogFile output(basename_, rollSize_, false);
 
   BufferPtr newBuffer1(new Buffer); // 用于替换前台的当前缓冲 currentbuffer
   BufferPtr newBuffer2(new Buffer); // 用于替换前台的预备缓冲 nextbuffer 
   newBuffer1->bzero();  
   newBuffer2->bzero();
 
   BufferVector buffersToWrite;  // 后台缓冲队列
   buffersToWrite.reserve(16);   // 两个不同的缓冲队列，涉及到锁的粒度问题
   
   // 异步日志开启，则循环执行
   while (running_) { 
     // <---------- 交换前台缓冲队列和后台缓冲队列 ---------->
     { // 锁的作用域，放在外面，锁的粒度就大了，日志落盘的时候都会阻塞 append
       // 1、多线程加锁，线程安全，注意锁的作用域
       MutexLockGuard lock(mutex_);
 
       // 2、判断前台缓冲队列 buffers 是否有数据可读
       // buffers 没有数据可读，休眠
       if (buffers_.empty()) {
         // 触发日志的落盘 (唤醒) 的两个条件：1.超时 or 2.被唤醒，即前台写满 buffer
	 // 如果前端缓冲区队列为空，就休眠 flushInterval_ 的时间
         cond_.waitForSeconds(flushInterval_); // 内部封装 pthread_cond_timedwait
       }
 
       // 只要触发日志落盘，不管当前的 buffer 是否写满都必须取出来，写入磁盘
       // 3、将当前缓冲区 currentbuffer 移入前台缓冲队列 buffers。
       // currentbuffer 被锁住 -> currentBuffer 被置空  
       buffers_.push_back(std::move(currentBuffer_)); 
   
       // 4、将空闲的 newbuffer1 移为当前缓冲，复用已经分配的空间
       currentBuffer_ = std::move(newBuffer1); // currentbuffer 需要内存空间
 
       // 5、核心：把前台缓冲队列的所有buffer交换（互相转移）到后台缓冲队列 
       // 这样在后续的日志落盘过程中不影响前台缓冲队列的插入
       buffersToWrite.swap(buffers_);  
 
       // 若预备缓冲为空，则将空闲的 newbuffer2 移为预备缓冲，复用已经分配的空间
       // 这样前台始终有一个预备缓冲可供调配
       if (!nextBuffer_) { 
         nextBuffer_ = std::move(newBuffer2);  
       }
     } // 注意这里加锁的粒度，日志落盘的时候不需要加锁了，主要是双队列的功劳
 
     // <-------- 日志落盘，将buffersToWrite中的所有buffer写入文件 -------->
 
     // 6、异步日志消息堆积的处理。
     // 同步日志，阻塞io，不存在堆积问题；异步日志，直接删除多余的日志，并插入提示信息
     if (buffersToWrite.size() > 25) {
       printf("Dropped\n");
       // 插入提示信息
       char buf[256];
       snprintf(buf, sizeof buf, "Dropped log messages at %s, %zd larger buffers\n",
                Timestamp::now().toFormattedString().c_str(),
                buffersToWrite.size()-2);  
       fputs(buf, stderr);
       output.append(buf, static_cast<int>(strlen(buf)));
       // 只保留2个buffer(默认4M)
       buffersToWrite.erase(buffersToWrite.begin()+2, buffersToWrite.end());   
     }
 
     // 7、循环写入 buffersToWrite 的所有 buffer
     for (const auto& buffer : buffersToWrite) {
       // 内部封装 fwrite，将 buffer中的一行日志数据，写入用户缓冲区，等待写入文件
       output.append(buffer->data(), buffer->length());  
     }
 
     // 8、刷新数据到磁盘文件？这里应该保证数据落到磁盘，但事实上并没有，需要改进 fsync
     // 内部调用flush，只能将数据刷新到内核缓冲区，不能保证数据落到磁盘（断电问题）
     output.flush();   
  
     // 9、重新填充 newBuffer1 和 newBuffer2
     // 改变后台缓冲队列的大小，始终只保存两个 buffer，多余的 buffer 被释放
     // 为什么不直接保存到当前和预备缓冲？这是因为加锁的粒度，二者需要加锁操作
     if (buffersToWrite.size() > 2) {
        // 只保留2个buffer，分别用于填充备用缓冲 newBuffer1 和 newBuffer2
       buffersToWrite.resize(2);  
     }
     // 用 buffersToWrite 内的 buffer 重新填充 newBuffer1
     if (!newBuffer1)
     {
       assert(!buffersToWrite.empty());
       // 把后端缓冲区的最后一个作为newBuffer1
       newBuffer1 = std::move(buffersToWrite.back()); 
       // 最后一个元素的拥有权已经转移到了newBuffer1中，因此弹出最后一个
       buffersToWrite.pop_back();
       // 重置newBuffer1为空闲状态（注意，这里调用的是Buffer类的reset函数而不是unique_ptr的reset函数） 
       newBuffer1->reset(); 
     }
     // 用 buffersToWrite 内的 buffer 重新填充 newBuffer2
     if (!newBuffer2) {
       assert(!buffersToWrite.empty());
       newBuffer2 = std::move(buffersToWrite.back()); // 复用 buffer
       buffersToWrite.pop_back();
       newBuffer2->reset();   // 重置指针，置空
     }
 
     // 清空 buffersToWrite
     buffersToWrite.clear();  
   }
   
   // 存在问题
   output.flush();
 }
```

# 代码分析

[跳转](./log_code.md)

# 参考

[muduo异步日志库模块的实现 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/572625895)

[muduo异步日志——core dump查找未落盘的日志 (yuque.com)](https://www.yuque.com/linuxer/xngi03/pfq9m1?)

[muduo笔记 日志库（一） - 明明1109 - 博客园 (cnblogs.com)](https://www.cnblogs.com/fortunely/p/15973948.html)
