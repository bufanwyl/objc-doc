##Bonjour

###一.Bonjour介绍
一般在进行Socket编程或者网络访问的时候，首先需要确认对方网络服务已经开启，且需要知道对方的域名或地址以及端口，然后才可以进行进一步操作。在互联网上好点，网络服务方一般常年开启，且一般IP地址是固定的，另由于DNS服务的存在，只要记住对方的域名便可以。但是在局域网，设备不一定连在上面，即使连上了，服务也不一定开了，每当设备连接到局域网的时候，IP地址一般都是动态分配的，所以情况变的复杂。Bonjour的存在便是苹果为了解决局域网设备间连接麻烦的问题。   
直白的说Bonjour就是是一种协议，使得局域网中的计算机可以方便的发布服务，发现服务和连接服务，达到零配置（[Zeroconf][link-zeroconf]）的目的。
[link-zeroconf]:http://zeroconf.org

Zeroconf Working Group指出要实现零配置网络服务的3个要求：

>+ IP地址
>+ **名字** 到 **IP地址** 的转换（即使没有DNS服务器的情况下）
>+ 发现网络中的服务

对于第一个要求相关系统和设备可以直接支持的，如动态IP地址分配。   
第二个要求则可以通过多播（UDP协议向局域网内一组机器发送数据）的方式发送 类似DNS查询的请求，开启着的网络服务收到之后便作出回应，告知自己的名字。   
第三个要求则通过DNS-SD来实现

Bonjour一般的工作模式便是：在同一个局域网中，一方开启服务，通过Bonjour接口将这个服务发布，服务搜索方在服务列表中便可以看到对应的设备的名字，选择设备便可以进行连接了。整个过程无需事先知道服务发布方的IP地址和端口号。   
我们常用的软件如iTunes的共享，keynote的remote控制或者支持Bonjour协议的打印机都可以看到Bonjour的影子。


###二.Bonjour的实现及使用
从上面的描述可以看出，Bonjour的用途便是在局域网内发布服务和搜索服务。
下面从实现层面讲解Bonjour。  

|层次|名称|
|:----|:----|
|Foundation|NSNetService/NSNetServiceBroswer|
|CoreFoundation|CFNetService/CFNetServiceBroswer|
|Low-Level Socket Based API|dns_sd.h(The DNS Service Discovery API)|
|Multicast DNS Responder|mDNSResponder (开源项目)|

一般情况下我们使用Foundation这一层接口就可以了，也是最方便的。
当然服务方在发布服务之前你得先启好网络服务，比如listening socket创建好，且开始侦听某个端口，关于socket编程的知识可以查看[Socket编程][link-socket]  

[link-socket]:Socket编程.md   

**1.发布服务**

	netService = [[[NSNetService alloc] initWithDomain:@""
	                                              type:@"_test._tcp"
	                                              name:@""
	                                              port:port] autorelease];
	if(netService != nil) {
	    [netService scheduleInRunLoop:[NSRunLoop currentRunLoop]
	                               forMode:NSRunLoopCommonModes];
	    netService.delegate = self;
	    [netService publish];
	}

**2.浏览服务**
	
+ 创建Service Broswer, 需要指定service type和domain，得和发布服务时候的type对应。还得设置delegate，然后实现其delegate方法，以便发现了服务之后进行处理以及对发现的服务进行获取IP地址和端口的结果进行处理。


		testServiceBrowser = [[NSNetServiceBrowser alloc] init];
		testServiceBrowser.delegate = self;
		[testServiceBrowser searchForServicesOfType:@"_test._tcp" inDomain:@""];


+ 实现Service Broswer 的delegate方法，处理服务增加或减少的事件


		//pragma mark NetServiceBroswer Delegate
		- (void)netServiceBrowser:(NSNetServiceBrowser*)netServiceBrowser
		           didFindService:(NSNetService*)service
	   	            moreComing:(BOOL)moreComing {
	 	   [netServiceArray addObject:service];
	 	   if (!moreComing) {
	 	       [serviceTableView reloadData];
	 	   }
		}
	
		- (void)netServiceBrowser:(NSNetServiceBrowser*)netServiceBrowser
	 	        didRemoveService:(NSNetService*)service
		               moreComing:(BOOL)moreComing {
		    [netServiceArray removeObject:service];
		    if (!moreComing) {
		        [serviceTableView reloadData];
		    }
		}
		
		
+ 连接服务

上面发现的Net Service是不带IP地址和端口信息的。   
从服务列表中选择一个已经发现的服务，进行Resolve，便可以获取服务的详细信息了。

	- (IBAction)connect:(id)sender{
	    NSUInteger selectedRow = [serviceTableView selectedRow];
	    NSNetService *selectedServiece = [netServiceArray objectAtIndex:selectedRow];
	    selectedServiece.delegate = self;
	    [selectedServiece resolveWithTimeout:5.0];
	}
	
Resolve成功

	//NSNetService Delegate
	- (void)netServiceDidResolveAddress:(NSNetService *)sender{
        NSLog(@"service ip:%@ port:%d",sender.address,sender.port);
		if ([sender getInputStream:&inputStream outputStream:&outputStream]) {
		    [outputStream scheduleInRunLoop:[NSRunLoop currentRunLoop]
		                                     forMode:NSDefaultRunLoopMode];
		    [outputStream open];
		    //发送数据
		    NSData *helloData = [@"Hello" dataUsingEncoding:NSUTF8StringEncoding];
		    [outputStream write:[helloData bytes] maxLength:[helloData length]];
		}
	}
	
上面的代码充分利用了输入输出流进行通信。如果你自己的使用socket的连接也是可以的，因为这个时候已经可以获取了对方的IP地址和端口了。