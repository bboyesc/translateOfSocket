# 网络编程主题

## 简介
<br />
>重要：这是一个为后面内容的准备性的文档。准确的说它会讨论一些技术，但不是最终的。苹果官方提供这个文档的目的是为了让你熟悉这个模块的接口。这里提供的信息可能会改变，这个文档里提到的接口信息会在最终的文档里面确定。想要了解这个文档的更新过程，点击这里[Apple Developer website](http://developer.apple.com/)。进入相关话题的reference library,并且打开它了解。
<br />

这个文档专门讲关于网络模块的相关知识。至于在这个文档里面提到的一些其他话题，我们假设你已经熟悉一些基本的网络概念。<br />
>重要：大部分开发人员不需要读这个文档，并且大部分网络应用不会需要这个文档里面的知识(估计意思是大部分app用http)。在你读这个文档之前，你需要提前读[Networking Overview](https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/Introduction/Introduction.html#//apple_ref/doc/uid/TP40010220)以便你更能理解这个文档里面的内容。
<br />

### 如何使用这个文档
这个文档包含如下几节：<br />
使用套接字(Socket)和套接字stream---描述了如何使用套接字和stream来做底层网络编程，从POSIX标准层到Foundation框架层。这个部分解释了现有的最好解决方法来写客户端和服务端。
<br />
域名系统(DNS)解析主机---解释了域名解析系统如何工作，以及如何避开域名解析系统的陷阱。
<br />
传输层安全(TLS)的正确实现---描述了如何安全的操作传输层从而避免让你的软件有严重的安全问题。这个部分包括TCP steam(使用CFStream或者NSStream API)和URL请求(使用NSURLConnection API)。
<br />

每个部分都是针对要用到相应知识来编码的开发者。
<br />
### 先决条件<br />
这个文档假设你已经度过下面的文档或者理解文档里面的相应的知识：
<br />
>[Networking Overview](https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/Introduction/Introduction.html#//apple_ref/doc/uid/TP40010220)-讲一个互联网软件工作的基本流程，如何避免一些常见的错误。
<br />
>[Networking Concepts](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/NetworkingConcepts/Introduction/Introduction.html#//apple_ref/doc/uid/TP40012487)-描述了以套接字为基础的网络操作和基本概念。
<br />
<br />




# 使用套接字和套接字流(stream)<br />
这部分讲如何用套接字和套接字stream编程，从POSIX协议层到Foundation框架层。
<br />
>重要：这部分将描述如何全面使用套接字连接，有些应用用更高一级的API更好比如NSURLConnection。想了解NSURLConnection，读[Networking Overview](https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/Introduction/Introduction.html#//apple_ref/doc/uid/TP40010220) .<br />这个部分讲一些在Cocoa或者Core Foundation之外的协议,而且这些协议是你必须支持的。
<br />

在大多数网络层中，软件被分为两类：客户端和服务端。在高级网络层(high-level API)中，这个区分更明显了。大多数用高级网络层的都是客户端。然而，在低级网络层中，这个界限一般很模糊。
<br />
套接字和数据流(stream)编程一般分为下面的两类：<br />
. 基于分组的通信(Packet-based communication)：程序一般一次操作一个数据包，监听到数据包并发送一个数据包作为回应。而且很有可能每个部分都会处理接收到或者发送的数据。网络通讯协议是一致的。
<br />
. 基于流的客户端(Stream-based clients):程序通过TCP来接收或者发送连续的数据流。对于这种模式，客户端和服务端的界限更明显。客户端和服务端对数据的处理很相似，但是他们构建通讯信道(communication channel)的方式截然不同。
<br />
这个部分分为如下几个部分：<br />
>选择一个API簇(API Family)---描述如何决定使用哪个API簇。<br />
>写一个基于TCP的客户端---描述如何构建优雅地TCP连接到服务器的服务。<br />
>写一个基于TCP的服务端---描述当我们写服务器的事后,如何监听到来的TCP连接。<br />
>工作基于分组的通信的套接字---描述如何在非TCP协议上工作，比如UDP。
<br />

## 选择一个API簇(API Family)<br />

对于基于套接字的链接，你选择那个API是根据你是发送一个链接到其他主机还是接受来自其他主机的链接。同时也决定于你是否使用TCP或者其他协议，下面是你做决定的时候需要考虑的一些因素：
<br />
.在OS X中，如果你有需要与非MAC平台的系统通讯，你可以用POSIX C网络编程API，这样你就可以在不同平台共用你的网络模块。如果你的程序是基于Core Foundation或者Cocoa (Foundation)的运行时循环(run loop)，你也可以使用Core Foundation的 CFStream API来集成POSIX标准的网络模块。另一方面，如果你用GCD，你可以添加一个套接字作为调度源(dispatch source). 
<br />
.在iOS中，POSIX标准没有被支持，因为这个标准不支持移动蜂窝网络(cellular radio)或者指定的VPN。通常情况下，你需要从平常的数据处理行数中分开网络模块并且用更高一级的API重写网络模块的代码。
<br />

>注意：如果你用POSIX标准的网络模块代码，你需要注意到POSIX标准的网络API不是协议无关的(protocol-agnostic)(你必须处理IPV4和IPV6的不同)。这是一个IP网络层的一个API而一个名字相关(connect-by-name)API，这意味着你必须做一些额外的工作，如果你想实现一些相同的内部链接特性和更高一级的API的健壮性问题。在你决定使用POSIX标准的网络模块以前，你需要读[Avoid Resolving DNS Names Before Connecting to a Host](https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/CommonPitfalls/CommonPitfalls.html#//apple_ref/doc/uid/TP40010220-CH4-SW20)
<br />

.对于监听一个端口的守护进程、服务、非TCP的链接，使用POSIX标准或者Core Foundation(CFSocket)的网络模块API。
<br />
.对于用Objective-C写的客户端，用它的Foundation框架的API。Foundation框架定义了针对URL链接、套接字流、网络服务和其他网络任务的高级接口。在iOS和OS X中，这个框架同时也是最基础的、UI无关的Objective-C框架，提供运行时循环的各种操作、字符串处理、集合类、文件访问等等。
<br />
. 如果客户端是C语言写的。用Core Foundation的网络API，这个框架和 CFNetwork框架是两个基于C语言的框架。他们一起定义了一些Foundation框架用到的结构体和函数。
<br />

>注意：在OS X中，CFNetwork框架是Core Services框架的子框架；在iOS中，CFNetwork框架是一个独立的一级框架。
<br />

### 写一个基于TCP的客户端
<br />
你编写友好的网络连接接口的方式是取决于你选择的编程语言、连接的类型(TCP,UDP等等)、是否要和其他平台分享相同的代码。
<br />
.在Objective-C中用NSStream来建立连接。
<br />
如果你要连接到一个主机，新建一个CFHost对象(不是NSHost-他们不能无缝桥接(toll-free bridged)),用[CFStreamCreatePairWithSocketToHost](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFStreamConstants/index.html#//apple_ref/c/func/CFStreamCreatePairWithSocketToHost)和[CFStreamCreatePairWithSocketToCFHost](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFSocketStreamRef/index.html#//apple_ref/c/func/CFStreamCreatePairWithSocketToCFHost)来开始一个套接字连接到指定主机和端口，并且关联两个CFStream对象。你也可以用NSStream对象。
<br />
你也可以用 CFStreamCreatePairWithSocketToNetService 函数来连接到一个Bonjour service(好像用于苹果设备之间的通讯)。读[ Discovering and Advertising Network Services](https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/Discovering,Browsing,AndAdvertisingNetworkServices/Discovering,Browsing,AndAdvertisingNetworkServices.html#//apple_ref/doc/uid/TP40010220-CH9)来获取更多信息。
<br />
.用POSIX标准如果需要支持跨平台。
<br />
如果你写的代码只用与苹果平台，name你要避免使用POSIX标准的代码，因为他们和高层的协议比起来，他们更难以操作。然而，如果你写的网络模块代码必须在不同平台分享的话，你可以用POSIX标准的网络API从而你能复用同样的代码。
<br />
千万别在主线程使用POSIX协议网络的同步API。如果你要用这种API，你必须在一个单独的线程里。
<br />
>注意：POSIX协议网络不支持iOS平台的蜂窝网络，因为这个原因，iOS平台不需要POSIX标准的网络API。

<br />
下面将讲NSStream的用法，除非特别标明，CFStream API一般都有一个同样名字的函数,而且实现也相似。
<br />
要了解更多POSIX标准的套接字API，读UNIX Socket FAQ在[ http://developerweb.net/](http://developerweb.net/) .
<br />

#### 建立一个链接
<br />
通常情况下，建立一个TCP链接到其他主机用的是数据流。Streams自动处理TCP链接面临的挑战。比如，数据流提供了通过主机名(hostname)链接的能力，在iOS中，数据流会自动唤醒设备得蜂窝模式或者指定的VPN当需要的时候(不像CFSocket 或者 BSD Socket)。相比于底层协议，Streams更相似Cocoa框架的接口，和Cocoa的文件操作API有很大的相似性。
<br />
你获取或者发送数据流取决于你是否要被链接或者主动链接到一个主机。
<br />
.如果你已经知道了一个主机的DNS名字或者IP地址，使用 Core Foundation来读取或者发送数据流通过 CFStreamCreatePairWithSocketToHost函数。你可以充分利用CFStream和NSStream的无缝转换。把CFReadStreamRef和CFWriteStreamRef对象转换为NSInputStream和 NSOutputStream对象。
<br />
.如果你想通过CFNetServiceBrowser对象来链接到一个主机，你可以通过CFStreamCreatePairWithSocketToNetService函数来接收或者发送数据。读取[Discovering and Advertising Network Services](https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/Discovering,Browsing,AndAdvertisingNetworkServices/Discovering,Browsing,AndAdvertisingNetworkServices.html#//apple_ref/doc/uid/TP40010220-CH9)
<br />
当你得到输出路和输入流以后，在没有用arc的情况下你必须马上占有他们，把他们引用到一个NSInputStream和NSOutputStream对象，并且设置他们的代理对象(这个对象需要实现NSStreamDelegate协议)。通过调用open方法在当前运行时循环上执行他们。
<br />
>注意：如果你需要同时处理一个以上链接，你需要区分是那个输入流和输出流关联。最直接的方式你新疆一个链接对象同时拥有输入流和输出流的引用，并且把这个对象设置为他们的代理对象。

<br />
#### 事件处理
当NSOutputStream对象的代理方法stream:handleEvent:被调用了，并且设置streamEvent参数的值为NSStreamEventHasSpaceAvailable，最后调用 write:maxLength:来发送数据。write:maxLength:方法要么返回发送的数据的长度，要么返回一个负数表示失败。如果这个方法返回的数据的长度小于你尝试发送的数据的长度，你必须把没有发送的那部分数据发送出去通过调用NSStreamEventHasSpaceAvailable事件。如果发生错误，你需要调用 streamError方法来确认是哪里发生了错误。
<br />
当NSInputStream对象的代理方法stream:handleEvent:被调用了，并且设置streamEvent参数的值为NSStreamEventHasBytesAvailable。你可以通过 read:maxLength:方法来读取接收到得数据。这个方法返回接收到得数据的长度或者返回一个负数表示接收失败。
<br />
如果接收到的数据的长度小于你需要的长度，你必须持有数据并且等待直到你收到所有的数据。如果发生错误，你需要调用 streamError方法来确认是哪里发生了错误。
<br />
如果链接的另一端中断了链接：
<br />
&nbsp;&nbsp;你的链接代理方法stream:handleEvent:被调用。并且streamEvent参数设置为NSStreamEventHasBytesAvailable。如果你去读取接收到的数据，你会发现长度为零。
<br />
&nbsp;&nbsp;你的代理方法stream:handleEvent: 会被调用。并且streamEvent参数被设置为 NSStreamEventEndEncountered。
<br />
当上面两个事件的其中一个发生了，代理方法需要处理链接操作结束工作和清理工作。
<br />
#### 结束链接
<br />
当结束一个链接的时候，我们首先要把它从当前运行时循环移除，设置链接的代理为nil(代理对象并没有被retain)。通过close方法关闭与链接关联的两个数据流，最后在释放者两个数据流对象(如果你没有使用ARC)或者把他们设置为nil。这就是通常的关闭链接的方式。然而，如果有下面两种情况你需要手动关闭链接：
<br />
&nbsp;&nbsp;对于一个数据流，如果你通过 setProperty:forKey: 方法设置 kCFStreamPropertyShouldCloseNativeSocket属性值为kCFBooleanFalse。
<br />
&nbsp;&nbsp;如果你通过CFStreamCreatePairWithSocket方法创建以BSD套接字为基础的数据流。一般情况下，数据流是在一个系统套接字(native socket)的基础上创建的，并且关闭的时候不会关闭底层的套接字。但是，你也可以设置自动关闭底层套接字通过设置kCFStreamPropertyShouldCloseNativeSocket属性值为kCFBooleanTrue。
<br />
#### 更多信息
要了解更多, 读[Stream Programming Guide]和[Using NSStreams For A TCP Connection Without NSHost]中的[Setting Up Socket Streams]部分, 或者下载[SimpleNetworkStreams](https://developer.apple.com/library/ios/samplecode/SimpleNetworkStreams/Introduction/Intro.html#//apple_ref/doc/uid/DTS40008979)工程来看看.

<br />
### 写一个基于TCP的服务器
未完待续。。。。。


国内ios界最大的行动已经开始了。全面开始翻译ios开发者文档。
欢迎加入，小伙伴们。有兴趣的，想要参加的赶紧来吧。
https://github.com/wang820203420/IOS-Developer-library-Chinese/tree/master

