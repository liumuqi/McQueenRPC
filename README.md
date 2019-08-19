McQueenRPC（代码在github上）每秒支撑的请求数上升了好几倍，测试结果的演变为：

37k –> 56k –> 65k –> 88k –> 93k –> 143k –> 148k –> 153k –> 160k –> 163k –> 168k

以上测试结果为在100并发、100 request byte、100 response byte以及单连接下的背景下得出的，在这篇blog中来分享下这个框架所做的一些优化动作，希望能给编写rpc框架或使用netty的同学们一点点帮助，也希望得到高手们更多的指点。



1、37k –> 56k

由于目前大部分的NIO框架采用的均为1个socket io线程处理多个连接的io事件，如果io线程时间占用太长的话，就会导致收到的响应处理的比较慢的现象，这步优化就是针对反序列化过程占用io线程而做的，采用的方法即为在读取流时仅根据长度信息把所有的bytes都读好，然后直接作为收到的信息返回给业务线程，业务线程在进行处理前先做反序列化动作，感兴趣的同学可以看看McQueenRPC中：NettyProtocolDecoder，以及NettyServerHandler。



2、56k –> 65k

在测试的过程中，发现YGC的耗时比较长，在咨询了sun的人后告诉我主要是由于有旧生代的数据结构引用了大量新生代对象造成的，经过对程序的分析，猜测是benchmark代码本身用于记录请求响应时间信息的ConcurrentLinkedQueue造成的，在某超级大牛的指示下，换成了在每个线程中用数组的方式，按区间来记录响应时间信息等，感兴趣的同学可以看看McQueenRPC中：SimpleProcessorBenchmarkClientRunnable



3、65k –> 88k

在某超级大牛的分析下，发现目前的情况下io线程的上下文切换还是比较频繁，导致io线程处理效率不够高，默认情况下，NIO框架多数采用的均为接到一个包后，将这个包交由反序列化的处理器进行处理，对于包中有多个请求信息或响应信息的情况，则采用一个一个通知的方式，而rpc框架在接到一个请求或响应对象时的做法通常是唤醒等待的业务线程，因此对于一个包中有多个请求或响应的状况就会导致io线程需要多次唤醒业务线程，这个地方改造的方法是nio框架一次性的将包中所有的请求或响应对象通知给业务线程，然后由业务线程pipeline的去唤醒其他的业务线程，感兴趣的同学可以看看McQueenRPC中：NettyProtocolDecoder，以及NettyClientHandler。



4、88k –> 93k

这步没什么可说的，只是多支持了hessian序列化，然后这个结果是用hessian序列化测试得出的，注意的是hessian不要使用3.1.x或3.2.x版本，这两个系列的版本性能极差，建议使用hessian 4.0.x版本。



5、93k –> 143k

在到达93k时，看到测试的结果中有不少请求的响应时间会超过10ms，于是用btrace一步一步跟踪查找是什么地方会出现这种现象，后来发现是由于之前在确认写入os send buffer时采用的是await的方式，而这会导致需要等待io线程来唤醒，增加了线程上下文切换以及io线程的负担，但这个地方又不能不做处理，后来就改造成仅基于listener的方式进行处理，如写入失败会直接创建一个响应对象，这次改造后效果非常明显，感兴趣的同学可以看看McQueenRPC中：NettyClient。



6、143k –> 148k

这步是在@killme2008 的指点下，将tcpNoDelay设置为了false，但这个设置不适用于低压力的情况，在10个线程的情况下tps由3w降到了2k，因此tcpNoDelay这个建议还是设置成true。



7、148k –> 153k

这步没什么可说的，只是多支持了protobuf序列化，这个结果是用protobuf序列化测试得出的，之前还测试过下@bnu_chenshuo 写的一个protorpc，是基于protobuf的rpc加上netty实现的，结果也很强悍，测试出来是149k，看来protobuf的rpc也是有不少值得学习的地方的。



8、153k –> 160k

server接到消息的处理过程也修改成类似client的pipeline机制，同时将之前获取协议处理器和序列化/反序列化的处理器的地方由map改成了数组，具体可以参考NettyServerHandler、ProtocolFactory和Codecs。



9、160k –> 163k

Grizzly的leader对grizzly部分的代码做了很多的修改，结果创造了目前rpc benchmark的最高纪录：163k。



10、163k –> 168k

@minzhou 对McQueenRPC的代码进行了优化，将之前在Decoder中做的构造String对象的部分挪到了业务线程中，于是TPS也有了一定的上升，感谢。



上面就是到目前为止的所有优化动作，其中的很多我估计高手的话是不会犯错的，我走的弯路多了些，总结来说rpc框架的优化动作为：

1、尽可能减少io线程的占用时间，把能做的事都挪到别的线程里去做；

2、尽可能减少线程上下文切换；

3、尽可能使用高效的序列化/反序列化；
