> tomcat 工作模式 运行模式

**工作模式**

 独立的servlet容器，servlet容器是web服务器的一部分； 

 进程内的servlet容器，servlet容器是作为web服务器的插件和java容器的实现，web服务器插件在内部地址空间打开一个jvm使得java容器在内部得以运行。反应速度快但伸缩性不足； 

 进程外的servlet容器，servlet容器运行于web服务器之外的地址空间，并作为web服务器的插件和java容器实现的结合。反应时间不如进程内但伸缩性和稳定性比进程内优； 

**Connector  三种运行模式**

- bio(blocking I/O) 即阻塞式I/O操作，表示Tomcat使用的是传统的Java I/O操作(即java.io包及其子包)。
  一个线程处理一个请求，缺点：并发量高时，线程数较多，浪费资源。

- nio(new I/O) Java nio是一个基于缓冲区、并能提供非阻塞I/O操作的Java API，因此nio也被看成是non-blocking I/O的缩写。它拥有比传统I/O操作(bio)更好的并发运行性能。
  利用 Java 的异步请求 IO 处理，可以通过少量的线程处理大量的请求。

- apr(Apache Portable Runtime/Apache可移植运行时) Tomcat将以JNI的形式调用Apache HTTP服务器的核心动态链接库来处理文件读取或网络传输操作，从而大大地提高Tomcat对静态文件的处理性能。Tomcat apr也是在Tomcat上运行高并发应用的首选模式。

> tomcat 优化

优化要从两个方面入手， 第一是，tomcat自身的配置，另一个是tomcat所运行的jvm虚拟机的。 

**jvm虚拟机调整：修改内存等 jvm 相关配置

```
JAVA_OPTS="-server -XX:PermSize=512M -XX:MaxPermSize=1024m -Xms2048m -Xmx2048m"  
```

验证：jps 获取 java 程序进程号，jmap -heap 进程号 进行查看

**tomcat配置调整**

1、关闭 AJP

一般我们使用  Nginx+tomcat的架构，不会用到该协议可以在配置中把 AJP连接器禁用。 

```    
    <!--
    <Connector protocol="AJP/1.3"
               address="::1"
               port="8009"
               redirectPort="8443" />
    -->
```

2、 执行器（线程池） 

Executor代表了一个线程池，可以在Tomcat组件之间共享。使用线程池的好处在于减少了创建销毁线程的相关消耗，而且可以提高线程的使用效率。 

```
    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
        maxThreads="500" minSpareThreads="50" prestartminSpareThreads="true" maxQueueSize="100"/>
```

 name：线程池名称，用于 Connector中指定。 

 namePrefix：所创建的每个线程的名称前缀，一个单独的线程名称为 namePrefix+threadNumber。 

 maxThreads：池中最大线程数。 

 minSpareThreads：活跃线程数，也就是核心池线程数，这些线程不会被销毁，会一直存在。 

 maxIdleTime：线程空闲时间，超过该时间后，空闲线程会被销毁，默认值为6000（1分钟），单位毫秒。 

 maxQueueSize：在被执行前最大线程排队数目，默认为Int的最大值，也就是广义的无限。除非特殊情况，这个值不需要更改，否则会有请求不会被处理的情况发生。 

prestartminSpareThreads：启动线程池时是否启动 minSpareThreads部分线程。默认值为false，即不启动。设置成 true 才能将 minSpareThreads 的参数生效。
3、运行模式

bio： 默认的模式,性能非常低下,没有经过任何优化处理和支持。 

nio：是一个基于缓冲区、并能提供非阻塞I/O操作的Java API，因此nio也被看成是non-blocking I/O的缩写。 

apr： 安装起来最麻烦,但是从操作系统级别来解决异步的IO问题,大幅度的提高性能。此种模式下，必须要安装apr和native，直接启动就支持apr。 

 建议tomcat8以下使用nio，tomcat8及以上使用nio2。

```
<Connector executor="tomcatThreadPool"  port="8080" protocol="org.apache.coyote.http11.Http11Nio2Protocol"
               connectionTimeout="20000"
               redirectPort="8443" />
```

4、Connector

Connector是连接器，负责接收客户的请求，以及向客户端回送响应的消息。  默认情况下 Tomcat只支持200线程访问，超过这个数量的连接将被等待甚至超时放弃，所以我们需要提高这方面的处理能力。 

```
<Connector port="8080"   
          protocol="HTTP/1.1"   
          maxThreads="1000"   
          minSpareThreads="100"   
          acceptCount="1000"  
          maxConnections="1000"  
          connectionTimeout="20000"   
          maxHttpHeaderSize="8192"  
          tcpNoDelay="true"  
          compression="on"  
          compressionMinSize="2048"  
          disableUploadTimeout="true"  
          redirectPort="8443"  
          enableLookups="false"  
          URIEncoding="UTF-8"
          executor="tomcatThreadPool" />  
```

port：端口

protocol：协议

```
//NIO 可以复用同一个线程处理多个connection(多路复用).同步非阻塞IO 
protocol="org.apache.coyote.http11.Http11NioProtocol"   
//NIO2  异步非阻塞IO 
protocol="org.apache.coyote.http11.Http11Nio2Protocol" 
```

 maxThreads： 由该连接器创建的处理请求线程的最大数目，也就是可以处理的同时请求的最大数目。  如果未配置默认值为200。如果一个执行器与此连接器关联，则忽略此属性，因为该属性将被忽略，所以该连接器将使用执行器而不是一个内部线程池来执行任务。 

 minSpareThreads： 线程的最小运行数目，这些始终保持运行。如果未指定，默认值为10。 

 acceptCount：当所有可能的请求处理线程都在使用时传入连接请求的最大队列长度。如果未指定，默认值为100。一般是设置的跟 maxThreads一样或一半，此值设置的过大会导致排队的请求超时而未被处理。所以这个值应该是主要根据应用的访问峰值与平均值来权衡配置。 

 maxConnections：在任何给定的时间内，服务器将接受和处理的最大连接数。当这个数字已经达到时，服务器将接受但不处理，等待进一步连接。NIO与NIO2的默认值为10000，APR默认值为8192。 

 connectionTimeout：当请求已经被接受，但未被处理，也就是等待中的超时时间。单位为毫秒，默认值为60000。通常情况下设置为30000。 

 maxHttpHeaderSize：请求和响应的HTTP头的最大大小，以字节为单位指定。如果没有指定，这个属性被设置为8192（8 KB）。 

 tcpNoDelay：如果为true，服务器socket会设置TCP_NO_DELAY选项，在大多数情况下可以提高性能。缺省情况下设为true。 

 compression：是否启用gzip压缩，默认为关闭状态。这个参数的可接受值为“off”（不使用压缩），“on”（压缩文本数据），“force”（在所有的情况下强制压缩）。 

 compressionMinSize：如果compression=”on”，则启用此项。被压缩前数据的最小值，也就是超过这个值后才被压缩。如果没有指定，这个属性默认为“2048”（2K），单位为byte。 

 disableUploadTimeout：这个标志允许servlet Container在一个servlet执行的时候，使用一个不同的，更长的连接超时。最终的结果是给servlet更长的时间以便完成其执行，或者在数据上载的时候更长的超时时间。如果没有指定，设为false。 

 enableLookups：关闭DNS反向查询。 

 URIEncoding：URL编码字符集。 

https://blog.csdn.net/lexang1/article/details/77849485

https://zhuanlan.zhihu.com/p/73620440

https://zhuanlan.zhihu.com/p/96692243



