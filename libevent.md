# **libevent**

## **简述**

- 开源。精简。跨平台（Windows、Linux、maxos、unix）

- 专注于网络通信
- 特性：基于“事件”的异步通信模型。--- 回调

## **使用**

	源码包安装：参考 README、readme
	
	./configure		检查安装环境 生成 makefile
	
	make			生成 .o 和 可执行文件
	
	sudo make install	将必要的资源cp置系统指定目录
	
	进入 sample 目录，运行demo验证库安装使用情况
	
	编译使用库的 .c 时，需要加 -levent 选项
	
	库名 libevent.so --> /usr/local/lib   查看的到

## **普通event版相关函数**

**创建 event_base (乐高底座)**

```
struct event_base *event_base_new(void);

struct event_base *base = event_base_new();
```

**创建 事件evnet**

```
常规事件 event	--> event_new(); 

bufferevent --> bufferevent_socket_new();
```

**将事件 添加到 base上**

```
int event_add(struct event *ev, const struct timeval *tv)
```

**循环监听事件满足**

```
int event_base_dispatch(struct event_base *base);
event_base_dispatch(base);
```

**释放 event_base**

```
event_base_free(base);	
```

**创建事件event**

```
struct event *event_new(struct event_base *base，evutil_socket_t fd，short what，event_callback_fn cb;  void *arg);
参数
	base： event_base_new()返回值。
	fd： 绑定到 event 上的 文件描述符
	what：对应的事件（r、w、e）
		EV_READ		一次 读事件
		EV_WRTIE	一次 写事件
		EV_PERSIST	持续触发。 结合 event_base_dispatch 函数使用，生效。
	cb：一旦事件满足监听条件，回调的函数。
    typedef void (*event_callback_fn)(evutil_socket_t fd,  short,  void *)	
    arg： 回调的函数的参数。

返回值：成功创建的 event
```

**添加事件到 event_base**

```
int event_add(struct event *ev, const struct timeval *tv);

	ev: event_new() 的返回值。

	tv：NULL
```

**从event_base上摘下事件**

	int event_del(struct event *ev);
	
		ev: event_new() 的返回值。

**销毁事件**

	int event_free(struct event *ev);
	
		ev: event_new() 的返回值。

**未决和非未决**

	非未决: 没有资格被处理
	
	未决： 有资格被处理，但尚未被处理
	
	event_new --> event ---> 非未决 --> event_add --> 未决 --> dispatch() && 监听事件被触发 --> 激活态 
	
	--> 执行回调函数 --> 处理态 --> 非未决 event_add && EV_PERSIST --> 未决 --> event_del --> 非未决



## **带缓冲区的bufferevent版相关**

- 带 read/write 两个缓冲区，借助 队列 实现



**创建、销毁bufferevent**

```
struct bufferevent *ev

struct bufferevent *bufferevent_socket_new(struct event_base *base, evutil_socket_t fd, enum bufferevent_options options)
参数：
	base： event_base

	fd:	封装到bufferevent内的 fd

	options：BEV_OPT_CLOSE_ON_FREE

返回： 成功创建的 bufferevent事件对象
```

	void  bufferevent_socket_free(struct bufferevent *ev);

**给bufferevent设置回调**

```
对比event：event_new( fd, callback );event_add() -- 挂到 event_base 上
bufferevent_socket_new（fd）  bufferevent_setcb（callback）
```

```
void bufferevent_setcb(struct bufferevent * bufev, bufferevent_data_cb readcb, 
					bufferevent_data_cb writecb, bufferevent_event_cb eventcb, void *cbarg );
参数
	bufev： bufferevent_socket_new() 返回值
	readcb： 设置 bufferevent 读缓冲的对应回调，read_cb{  bufferevent_read() 读数据 bufferevent_write}
	writecb： 设置 bufferevent 写缓冲的对应回调，write_cb {  } -- 给调用者，发送写成功通知，可以 NULL
	eventcb： 设置 事件回调。   也可传NULL
	events： BEV_EVENT_CONNECTED
	cbarg：	上述回调函数使用的 参数

read 回调函数类型：
	typedef void (*bufferevent_data_cb)(struct bufferevent *bev, void*ctx);

	void read_cb(struct bufferevent *bev, void *cbarg )
	{
		.....
		bufferevent_read();   --- read();
	}

bufferevent_read 函数的原型：
	size_t bufferevent_read(struct bufferevent *bev, void *buf, size_t bufsize);

bufferevent_write 函数的原型：
	int bufferevent_write(struct bufferevent *bufev, const void *data,  size_t size);

event 回调函数类型：
	typedef void (*bufferevent_event_cb)(struct bufferevent *bev,  short events, void *ctx);

    void event_cb(struct bufferevent *bev,  short events, void *ctx) {}
```



**启动、关闭 bufferevent的 缓冲区**

```
void bufferevent_enable(struct bufferevent *bufev, short events);   启动	
参数
	events： EV_READ、EV_WRITE、EV_READ|EV_WRITE

	默认、write 缓冲是 enable、read 缓冲是 disable

bufferevent_enable(evev, EV_READ);		-- 开启读缓冲
```



## **libevent实现C/S框架**

### **客户端**

#### **创建连接客户端**

	原：socket();connect();
	现：int bufferevent_socket_connect(struct bufferevent *bev, struct sockaddr *address, int addrlen);
	参数：
		bev: bufferevent 事件对象（封装了fd）
		address、len：等同于 connect() 参2和参3

#### **客户端 libevent 创建TCP连接**

```
1) 创建 event_base = event_base_new()
2) 使用 bufferevent_socket_new 创建一个用跟服务器通信的 bufferevent 事件对象
3) 使用 bufferevent_socket_connect 连接服务器
4) 使用 bufferevent_setcb 给 bufferevent 对象的read、write、event设置回调
5) 设置bufferevent 对象的读写缓冲区 enable/disable
6) 接收和发送数据，bufferevent_read 、bufferevent_write
7) 启动循环监听 event_base_dispatch()
8) 释放资源
```

### **服务器**

#### **创建监听服务器**

```
原：socket();bind();listen();accept();

现在：
struct evconnlistener * listner

struct evconnlistener *evconnlistener_new_bind(struct event_base *base, evconnlistener_cb cb, void *ptr, 
						unsigned flags, int backlog, const struct sockaddr *sa, int socklen);
参数：
    base： event_base

    cb: 回调函数。 一旦被回调，说明在其内部应该与客户端完成， 数据读写操作，进行通信

    ptr： 回调函数的参数

    flags： LEV_OPT_CLOSE_ON_FREE | LEV_OPT_REUSEABLE

    backlog： listen() 2参，监听个数，最大为128， -1 表最大值

    sa：服务器自己的地址结构体

    socklen：服务器自己的地址结构体大小

返回值：成功创建的监听器
```

#### **释放监听服务器**

	void evconnlistener_free(struct evconnlistener *lev);

#### **服务器端 libevent 创建TCP连接**

```
1）创建event_base = event_base_new()
2）使用evconnlistener_new_bind 创建监听服务器,设置其回调函数listner_cb,当有客户端成功连接时,这个回调函数会被调用
3）封装 listner_cb() 在函数内部。完成与客户端通信
4）创建bufferevent事件对象，bufferevent_socket_new()
5）使用bufferevent_setcb() 函数给 bufferevent的 read、write、event 设置回调函数
6）当监听的 事件满足时，read_cb会被调用， 在其内部 bufferevent_read();读
7）设置读缓冲、写缓冲的 使能状态 enable、disable
8）启动循环 event_base_dispath()
9）释放连接
```



## **相关代码**

**使用libevent 实现管道的读写**

​	/code/event_pipe

**带buffer的 bufferevent实现 C/S模型**

​	/code/bufferevent_cs

**B/S模型：bs-epoll-http-server**

​	/code/epoll_http_server

**B/S模型：bs-libevent-http-server**

​	/code/libevent_http_server

