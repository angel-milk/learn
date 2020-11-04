### make new 区别 

```go
	a := new([]int)
	fmt.Println(a)	// output : &[]
	b := make([]int, 2)
	fmt.Println(b)  // [0 0]
	(*a)[0] = 2     // 指针
	b[1] = 2        // 直接使用
	fmt.Println(a) 	// panic: runtime error: index out of range
	fmt.Println(b)	// [0 2]

 ``` 
 * new(T) 返回 T 的指针 *T 并指向 T 的零值.
 * make(T) 返回的初始化的 T,只能用于 slice,map,channel,要获得一个显式的指针,使用new进行分配,或者显式地使用一个变量的地址.
 * new 函数分配内存,make函数初始化(分配内存);

### linux信号
kill -l
> 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
> 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
> 11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
> 16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
> 21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
> 26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
> 31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
> 38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
> 43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
> 48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
> 53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
> 58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
> 63) SIGRTMAX-1  64) SIGRTMAX

* HUB 1 挂起信号  没有捕获该信号，当收到该信号时，进程就会退出
* INT 2 中断信号（Ctrl+C）  中断前台进程
* QUIT 3 中断信号（Ctrl+\）  退出前台进程，同时产生core文件
* KILL 9 强制终止信号  程序结束信号，与SIGKILL不同的是，该信号可以被阻塞和终止。通常用来要示程序正常退出。执行shell命令Kill时，缺省产生这个信号。默认动作为终止进程。
* USR1 10 用户定义信号 即程序员可以在程序中定义并使用该信号 默认动作为终止进程。
* USR2 12 用户定义信号 即程序员可以在程序中定义并使用该信号 默认动作为终止进程。
* TERM 15 程序结束信号，与SIGKILL不同的是，该信号可以被阻塞和终止。通常用来要示程序正常退出。执行shell命令Kill时，缺省产生这个信号。默认动作为终止进程。
* CONT 18 如果进程已停止，则使其继续运行。默认动作为继续/忽略。
* STOP 19 停止进程的执行。信号不能被忽略，处理和阻塞。默认动作为暂停进程。


### golang 热重启
> 重启方式：
>> 前端负载均衡（nginx）保证至少有个一个服务可用，依次升级。
>> 应用热重启，更新源码和配置，升级而不停服务。

> 热重启原理
* 监听信号 usr2
* 收到重启信号时fork子进程，同时需要讲服务监听的socket文件描述符传递给子进程。
* 子进程接收并监听父进程传递的socket。
* 等待子进程启动成功后，停止父进程对新连接的接收。
* 父进程退出，重启完成。

### endless热重启
* 1 先编译新程序，执行kill -1 旧进程id
```go
func (srv *endlessServer) handleSignals() {
	...
	for {
		sig = <-srv.sigChan
		srv.signalHooks(PRE_SIGNAL, sig)
		switch sig {
		case syscall.SIGHUP:	//接收到-1信号之后，fork一个进程，并运行新编译的程序
			log.Println(pid, "Received SIGHUP. forking.")
			err := srv.fork()
			if err != nil {
				log.Println("Fork err:", err)
			}
		...
		default:
			log.Printf("Received %v: nothing i care about...\n", sig)
		}
		srv.signalHooks(POST_SIGNAL, sig)
	}
}

func (srv *endlessServer) fork() (err error) {
	...
	path := os.Args[0]	//获取当前程序的路径，在子进程执行。所以要保证新编译的程序路径和旧程序的一致。
	var args []string
	if len(os.Args) > 1 {
		args = os.Args[1:]
	}

	cmd := exec.Command(path, args...)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	cmd.ExtraFiles = files	//socket在此处传给子进程，windows系统不支持获取socket文件，所以endless无法在windows上用。windows获取socket文件时报错：file tcp [::]:9999: not supported by windows。
	cmd.Env = env	//env有一个ENDLESS_SOCKET_ORDER变量存储了socket传递的顺序（如果有多个socket）
	...

	err = cmd.Start()	//运行新程序
	if err != nil {
		log.Fatalf("Restart: Failed to launch, error: %v", err)
	}

	return
}
```
* 2 listenAndServe 新进程启动之后执行ListenAndServe方法，主要用于系统信号监听，判断自己所在进程是否时子进程，如果是，则发送中断信号给父进程，让父进程退出。最后调用serve方法为socket提供新服务。
```go
func (srv *endlessServer) ListenAndServe() (err error) {
    ...
	go srv.handleSignals()
	l, err := srv.getListener(addr)
	if err != nil {
		log.Println(err)
		return
	}
	srv.EndlessListener = newEndlessListener(l, srv)
	if srv.isChild {
		syscall.Kill(syscall.Getppid(), syscall.SIGTERM)		//给父进程发出中断信号
	}
	...
	return srv.Serve()	//为socket提供新的服务
}
```
* 3 复用socket（endless核心） 
```go
func (srv *endlessServer) getListener(laddr string) (l net.Listener, err error) {
	if srv.isChild {//如果此方法运行在子进程中，则复用socket
		var ptrOffset uint = 0
		runningServerReg.RLock()
		defer runningServerReg.RUnlock()
		if len(socketPtrOffsetMap) > 0 {
			ptrOffset = socketPtrOffsetMap[laddr]//获取和addr相对应的socket的位置
		}

		f := os.NewFile(uintptr(3+ptrOffset), "")//创建socket文件描述符
		l, err = net.FileListener(f)//创建socket文件监听器
		if err != nil {
			err = fmt.Errorf("net.FileListener error: %v", err)
			return
		}
	} else {//如果此方法不是运行在子进程中，则新建一个socket
		l, err = net.Listen("tcp", laddr)
		if err != nil {
			err = fmt.Errorf("net.Listen error: %v", err)
			return
		}
	}
	return
}
```

### http
* HTTP协议是Hyper Text Transfer Protocol（超文本传输协议）缩写，是用于万维网www服务器传输超文本到本地服务器的传送协议。
* 基于TCP/IP通信协议传递数据
* 应用层面向对象协议
* 基于请求-相应模式
* 无状态保存
* 无连接（每次请求只有一个连接，服务器处理完客户请求，，收到客户应答，断开连接）


#### 请求相应步骤
+ 客户端连接到web服务器
	+ 一个HTTP客户端（浏览器），与Web服务器的HTTP端口（默认80）建立一个TCP套接字连接。
+ 发送HTTP请求
	+ 通过TCP套接字，客户端向Web服务器发送一个文本请求报文（请求行，请求头部，空行，请求数据4部分）
+ 服务器接收请求并返回HTTP相应
	+ Web服务器解析请求，定位请求资源。服务器将资源副本写到TCP套接字，由客户端读取。响应（状态行，相应头部，空行，相应数据构成）
+ 释放TCP连接
	+ 若connection模式为close，服务器主动关闭TCP连接，客户端被动关闭，释放TCP。
	+ 若connection模式为keepalive，该连接继续保持一段时间，该时间可以继续接收请求。

> 例: 浏览器输入url，按下回车流程：
* 1 浏览器向DNS服务器请求解析url中域名对应的IP地址。
* 2 解析出IP，根据ip地址和端口80，与服务器建立TCP连接。
* 3 浏览器发出读取文件(url域名后的对应文件)的HTTP请求，该请求报文作为TCP三次握手中第三个报文 的数据发送给服务器。
* 4 服务器对浏览器的请求做出相应，并把对应html文本发送给浏览器。
* 5 释放TCP连接
* 6 浏览器将接收的html文本进行展示。

#### HTTP请求方式
* 1 GET
* 2 POST
* 3 HEAD
* 4 PUT
* 5 DELETE
* 6 TRACE
* 7 OPTIONS
* 8 CONNECT

#### HTTP状态码
+ 1xx消息--请求已被服务器接收，继续处理
+ 2xx成功--请求已成功被服务器接收，并处理
+ 3xx重定向--需要进行附加操作以完成请求
+ 4xx请求错误--请求含有语法错误或无法被执行
+ 5xx服务器错误--服务器处理请求出现错误


> 常见状态码
>> + 200 ok 			//客户端请求成功
>> + 301 moved permanently 			//永久移动，请求资源重定向到心的uri
>> + 302 found 			//临时移动
>> + 400 bad request 	//客户端请求语法错误，不能被服务端理解
>> + 401 unauthorized 	//请求未经过授权
>> + 403 forbidden 		//服务器收到请求，但拒绝提供服务
>> + 404 not found 		//请求资源不存在
>> + 500 internal Server error 	//服务器错误
>> + 502 bad gateway 	//网关代理服务器执行请求时，从远端服务器接收到一个无效响应
>> + 503 server unavailable 	//服务器当前不能处理客户端请求，一段时间后恢复


#### URL
> https://weibo.com:80/12345/detail?id=1111
+ 传送协议
	+ http就是协议
+ 层级URL标记符合
	+ //,固定不变
+ 访问资源凭证信息
+ 服务器
	+ 通常是域名或ip
+ 端口号
	+ HTTP默认80
+ 路径
	+ /12345/detail,可直接找到对应资源
+ 查询
	+ GET方式，id=1111
+ 片段
	+ 通常我们说是html中的锚点。使用在客户端，不会传送给服务器。