---
title: 手写http服务器（一）
date: 2019-05-29 10:44:16
categories: mytomcat
description: V1
---
>这里默认读者是连http服务器是啥一点概念都没有（但是得有网络编程基础），会从最基本的概念讲起，一步步写一个服务器，大神就不要点进来看我献丑了hhh


# 版本全览
版本一：基于bio百行代码实现http服务器
版本二：基于nio
版本三：serlevt容器、cookie、封装、日志
版本四：长连接、参考了tomcat设计
# 起源
*早在学jsp的时候，就很好奇tomcat是什么，为什么可以运行jsp、servlet？当时试图看业界盛誉的《how to tomcat work》，没看懂（但这确实是本神书）；问师兄，师兄也不太清楚。网上的博客就更不用说了。这个困惑就先被我放到一边去了。
寒假的时候用socket、多线程写了一个聊天室。突然明白，其实tomcat就像一个聊天室的服务器啊，浏览器相当于客户端，HTTP请求响应相当于聊天消息，浏览器给服务器发消息，服务器做处理。http报文只是遵守一定格式的字符串。
只不过聊天室的服务器是把收到的信息转发给其他客户端。而HTTP的服务的返回一个符合HTTP协议的消息。
最初的版本v1，只能把收到的HTTP报文打印到控制台，返回浏览器响应显示一句话。
后来发现github上比我大一两届的前辈也写过这个东西，但是他们有好多功能比如基于nio、servlet等等，像模像样。受到启发，这项目认真起来大有搞头。就有了把这个当成自己的一个项目去完善的心态。看了一些书、博客、视频。最近还发现群里的小伙伴也有在写这个的，此处点名糖糖。*
写这个服务器的好处：
* 写轮子所需几乎覆盖java所有知识点，因为即使是一个xml解析也要自己写，不像写后台一样有很多框架、工具可以用。
* 成就感强。tomcat算比较适合自己造轮子的入门中间件。曾经也有写一个Spring的想法，但看了源码之后就放弃了。Spring源码优秀，适合参读，但不太适合自己撸一个。
* 对后台开发的眼界会

# V1
## 关键字：bio、百行代码内

***[代码链接](https://github.com/wangxuanni/MyTomcat/tree/master/src/v1/BioServer.java)***


## 技术点
* BIO
* Socket
* 缓存线程池
* lambda表达式
* HTTP请求响应报文

## 5个步骤的关键代码

一、开启服务器绑定端口
`ServerSocket serverSocket = new ServerSocket(8080);`
二、一直循环等待
`while（true）{`
三、收到请求的套接字
`Socket socket = serverSocket.accept();`
四、套接字可以获得一个字节输入流，把这个输入流转成字符串，打印在控制台。
`InputStream inputStream = socket.getInputStream();`
五、创建响应报文（其实就是按照HTTP报文的格式构建的一个字符串）并返回响应
`BufferedWriter bw =new BufferedWriter(new 
OutputStreamWriter(socket.getOutputStream()));`


## 注意一些小坑
缓存字符流写入后记得要刷新！！！bw.flush();
选择缓存线程池是因为它更适合短连接。
构建响应报文用StringBuilder比String更好
响应报文正文要构建好并且要记录长度，因为响应头有一个Content-length:正文长度属性要填
记得在finally块把socket.close()关闭



# V2


## 关键字：NIO
这个版本主要从bio升级为nio。
***[代码链接](https://github.com/wangxuanni/MyTomcat/tree/master/src/v2)***
v2
最重要是弄懂nio是时候掏出我之前的bio、nio笔记了。
### bio
先讲一下bio抛砖引玉，主要是为了和nio做对比。
#### 1.概念
阻塞并同步。特点是IO两个阶段被阻塞。基于流模型。
客户端一个请求服务端就启动一个线程，线序发生请求给内核，由内核去通信，在内核准备好数据之前，线程是被挂起的，知道数据从内核复制到用户空间。调用是可靠的线性顺序
缺点：每次请求创建一个线程在销毁开销比较大，操作系统对线程的总数有限制，太多服务器可能瘫痪。可以用线程池改进。
创建bio服务器只需要一步。
#### 2.配置服务器
```
一、ServerSocket serverSocket = new ServerSocket(8080);
```
#### 3.处理请求返回响应
```
二、一直循环等待
`while（true）{
三、收到请求的套接字
`Socket socket = serverSocket.accept();`
四、用套接字得到输入输出流，处理请求响应
}
```
### nio
#### 1.概念
非阻塞并同步。可构建多路复用、同步非阻塞的io操作。特点是程序去不断询问内核是否准备好，基于buffers。
客户端请求会注册到多路复用上，单线程轮询到有io请求时，才启动一个线程进行处理，仅仅selector是阻塞的。
核心：Channels（类似流，全双工，可以读写。socketChannel、serverSocketChannel）
buffers8种基本类型都有，数据从channel读到buffer中，也可以从buffer写到channel。本质是一块方便读写数据的内存
selector多路复用器，以监听多个Channel通道感兴趣的事情。允许单线程处理多个channel。 
* OP_ACCEPT: 接收就绪
* OP_READ: 读取就绪
* OP_WRITE: 写入就绪
* OP_CONNECT: 连接就绪

一个server socket channel准备好接收新进入的连接称为“接收就绪”。某个channel成功连接到另一个服务器称为“连接就绪”。一个有数据可读的通道可以说是“读就绪”。等待写数据的通道可以说是“写就绪”。
所以只有OP_ACCEPT: 接收就绪是serviceSocketChannel使用的，其他三个都是socketChannel使用。

#### 2.配置服务器
```
     /**
         * 1. 创建Selector
         */
        Selector selector = Selector.open();
        /**
         * 2. 通过ServerSocketChannel创建channel通道
         */
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        /**
         * 3. 为channel通道绑定监听端口
         */
        serverSocketChannel.bind(new InetSocketAddress(8000));

        /**
         * 4. **设置channel为非阻塞模式**
         */
        serverSocketChannel.configureBlocking(false);
        /**
         * 5. 将channel注册到selector上，监听连接事件
         */
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        System.out.println("服务器启动成功！");
       

```
总结一下上面代码，创建多路复用器，创建ServerSocketChannel，简称ssc，ssc绑定端口并设置为非阻塞，将ssc注册到多路复用器上，监听连接事件。到这里服务器已经配置好了。

#### 3.处理请求返回响应

```
 /**
    while (true) {
            int readyChannels = selector.select();
            if (readyChannels == 0) {
                continue;
            }
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator iterator = selectionKeys.iterator();

            while (iterator.hasNext()) {

                SelectionKey selectionKey = (SelectionKey) iterator.next();

                if (selectionKey.isAcceptable()) {
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector, SelectionKey.OP_READ);

                    Response response = new Response(socketChannel);

                    response.print("<html>");
                    response.print("<head>");
                    response.print("<title>");
                    response.print("服务器响应成功");
                    response.print("</title>");
                    response.print("</head>");
                    response.print("<body>");
                    response.print("来而不往非礼也");
                    response.print("</body>");
                    response.print("</html>");

                    response.pushToBrowser(200);

                }

                if (selectionKey.isReadable()) {
                    Request request = new Request(selectionKey);
                }
                iterator.remove();
            }

        }


```
#### 4.总结

循环询问多路复用器的selectedKeys是否有值，等待连接事件
当有连接事件过来了，创建socketChannel
将socketChannel设置为非阻塞工作模式
将channel注册回selector上，监听**可读事件**
创建Response并传入socketChannel，在Response里socketChannel.write();返回响应

如果是可读事件，把selectionKey传入Request解析请求，Request类首先selectionKey里获取socketChannel（就之前接入事件创建的socketChannel），然后创建byteBuffer，用于读取客户端请求信息。

#### 5.其他
Selector、ServerSocketChannel本身为抽象类，不能直接创建,需要通过open()方法打开。
注意这段代码
```
     if (readyChannels == 0) {
                continue;}
```

最后献上教程，关于nio，[慕课有一个特别好的教程](http://www.imooc.com/learn/1118)，老师讲的很清楚。
# V3
进行封装 request接受请求并打印。
response根据传入的状态码封装固定的头信息、推送响应信息。
server类只关心内容和状态码.
