# java 高并发核心编程 卷一

## 二、  高并发IO的底层原理

### 2.1 io 读写的基本原理

系统内核空间 和 用户空间。 访问系统资源往往需要切换到内核空间去。

通过调用read 和write来实现对物理设备的写入和读入，但是并不是对设备的直接读写，而是缓存的复制，将用户空间的数据，复制到内核缓冲区，然后交给系统内核进行操作。

#### 2.1.1 内核缓冲区和进程缓冲区

缓冲区 目的式减少设备之间的频繁的物理交换（<font color='red'>解决生产消费速率不一致和不同步问题</font>）。

操作系统内核只有一个内核缓冲区。进程又各自独立的缓冲区。 linux中大多数读写操作都是用户缓冲区和内核缓冲区的数据交换。

### 2.2 io模型

[Operating Systems: I/O Systems (uic.edu)](https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/13_IOSystems.html)

![img](https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/images/Chapter13/13_03_Interrupt_IO.jpg)



### 2.2 四种io模型

* bio  该过程需要内核io操作彻底完成后才能返回到用户空间执行用户程序的操作指令。

* nio 用户空间程序不用等待内核io操作完成就可以立刻返回用户空间执行后续的指令。

​       看上面的流程图。 可以认为，bio是当完成设备初始化之后，让该内核级线程放弃cpu的使用权 置为阻塞状态，等数据传输完毕之后，再将内核级线程置为runable状态。

​       而nio 是初始化完毕后，并不置为阻塞状态，可以让cpu干一些其他的事情。

​    在通信的角度理解这个事情：

​    bio 是一个简单的r s。nio 是r s ，然后通过轮询的方式去获取状态。

* 多路复用

​       在nio的基础上，通过一个select / poll 系统调用，同时监听多个文件描述符。

* aio

  用户线程的线程编程被动接受者。

### 2.3 修改文件描述符

在 linux 里面修改 /etc/security/limits.conf 来处理最大文件描述符。（归系统管理员来处理）

每个操作系统的配置还可能不一样。



## 三、nio核心详解

### 3.1 java nio简介

引入了三个核心的组件

数据： buffer   流程 selector 和 channel

#### 3.1.1 nio 和 oio的对比

oio 是面向流的，nio是面向缓冲区的

oio是阻塞的，nio是非阻塞的

### 3.2  nio 类和属性

#### 3.2.1 buffer 类

ByteBuffer、CharBuffer、DoubleBuffer> FloatBuf&r、IntBuffer、LongBuflfer、ShortBufifer、 MappedByteBuffero前7种Buffer类型覆盖了能在10中传输的所有Java基本数据类型，第8种 类型是一种专门用于内存映射的ByteBuffer类型。

#### 3.2.2 buffer类的重要属性

capacity ， position， limit。

capacity  是总量大小，一般是初始化数组的大小。

position是位置信息。

mark: 标记函数。用来恢复标记的值。调用reset之后，positon 会恢复到mark的位置上。

limit是读取最大上限。position偏移的最大值。

### 3.3 nio buffer重要实现方法

静态方法实现 allocate。没啥好说的。



一般情况下 使用步骤如下：

（1） 使用创建子类实例对象的allocate。方法创建一个Buffer类的实例对象。 

（2） 调用put（）方法将数据写入缓冲区中。 

（3） 写入完成后，在开始读取数据前调用Buffer.flipO方法，将缓冲区转换为读模式。 

（4） 调用get（）方法，可以从缓冲区中读取数据。 

（5） 读取完成后，调用BufTer.clearf）方法或Buffer.compact（）方法，将缓冲区转换为写模式, 可以继续写入。



### 3.4 nio channel类



*  FileChannel：文件通道，用于文件的数据读写。 
* SocketChannel：套接字通道，用于套接字TCP连接的数据读写。 
*  ServerSocketChannel：服务器套接字通道（或服务器监听通道），允许我们监听TCP 连接请求，为每个监听到的请求创建一个SocketChannel通道。 
* Datagramchannel：数据报通道，用于UDP的数据读写。

### 3.5 nio selector

监听事件类型

*  可读：SelectionKey.OP_READ 
* 可写：SelectionKey.OP_WRITE 
*  连接：SelectionKey.OP_CONNECT 
* 接收：SelectionKey.OP_ACCEPT

Socketchannel 三次握手 触发 OP_CONNECT 事件，ServerSocketChannel 监听到连接来到的时候触发OP_ACCEPT。

有数据可读的时候 触发OP_READ ， 等待数据写入OP_WRRITE

其中 fileChannel是不能被选择器监控或选择的，继承SelectableChannel 才可以。（file 本质上还是操作的缓存，所以才不不需要nio）

 SelectionKey 封装了基本的操作，并且获取selector 以及 channel。



## 四、 reactor模式



### 4.1 定义 

Reactor模式由Reactor线程、Handlers处理器两大角色组成，两大角色的职责分别如下:  

* Reactor线程的职责：负责响应10事件，并且分发到Handlers处理器。 
* Handlers处理器的职责：非阻塞的执行业务处理逻辑。



### 4.2处理模式

* Connection Per Thread模式(一个线程处理一个连接)。 消耗大量的线程资源，并且增加创建、切换、销毁的开销
* 单线程reactor 模式  简单地说，Reactor和Handlers处于一个线程中执行。 实现的时候，只用使用attach 就可以了。void attach(Object o)：将对象附加到选择键即可。
* 可以 对handler 和reactor 都扩展线程数量。

### 4.3 与其他模式对比

* 生产者 消费者模式对比

  没有使用专门的队列去存储io事件，查询到io事件之后，根据io选择键交给不同的handler来处理。

* 与观察者模式对比

  将事件保定到一个handler，每一个选择键被查询之后，所有的事件都会分发给所绑定的handler。

### 4.4 优缺点

优点：

* 相应快
* 编程相对简单，避免了复杂的线程同步，也避免了多线程个进程之间的切换开销。
* 可扩展，可以方便的增加反映器线程数量来充分利用cpu资源。

缺点

* 增加复杂度、门槛、不利于调试。
* 依赖底层多路复用支持。
* 出现长时间读写，就会影响反应器中其他通道的io处理。



## 五、 netty核心原理和基础实战

### 5.1 netty 的channel

netty 对channel 冲洗进行了封装。

• NioSocketChannel:异步非阻塞 TCP Socket 传输通道。

 • NioServerSocketChannel:异步非阻塞TCP Socket服务端监听通道。

 • NioDatagramChannel:异步非阻塞的UDP传输通道。 • NioSctpChannel:异步非阻塞Sctp传输通道。

 • NioSctpServerChannel:异步非阻塞Sctp服务端监听通道。

 • OioSocketChannel:同步阻塞式 TCP Socket 传输通道。

 • OioServerSocketChannel:同步阻塞式TCP Socket服务端监听通道。

 • OioDatagramChannel:同步阻塞式UDP传输通道。 

• OioSctpChannel:同步阻塞式Sctp传输通道。

 • OioSctpServerChannel:同步阻塞式Sctp服务端监听通道。 



### 5.2 channelOption

* SO_RCVBUF  设置接受缓冲区
* SO_SNDBUF 设置发送缓冲区
* TCP_NODELAY 开启和关闭nagle算法。
* SO_KEEPALIVE 开启tcp的心跳检测，默认tcp 心跳间隔是7200秒，两小时，默认关闭此功能。
* SO_REUSEDADDR 地址复用。
* SO_LINGER close调用后的行为。 -1 会不限事件将缓冲区数据发送到对端 ， 正整数代表这段时间内发送数据，如果没有发送完毕则发送rst包。
* SO_BACKLOG 服务器接受连接的队列长度。 三次握手排队数。如果连接频繁，服务器处理慢，可以适当增大参数。
* SO_BROADCAST tcp 传输选项，广播模式

### 5.3 channel 和pipline 类图

### 5.4 bytebuf优势

* 池化，减少内存赋值和gc。
* 支持零复制
* 不用flip
* 扩展性好
* 可以自定义缓冲区类型
* 读取和写入索引分开
* 方法链式调用
* 使用引用计数来自己控制内存。

### 5.5 池化配置

* 是否使用池化的的bytebuf 。使用io.netty.allocator.type来配置。



### 实战东西不用看，没啥东西



## 九、http原理与web服务器实战

### 没有讲啥

就是二/三层负载均衡 ，通过应答 arp搞的，换的是mac地址。

四层负载均衡 就是换了ip + 端口。 典型的就是防火墙里面的nat。

七层负载均衡 spring cloud gateway。 重试。

