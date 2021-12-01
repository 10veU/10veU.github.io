---
title: Java NIO
date: 2021-11-26 14:10:57
categories: 
    - Java
    - NIO
cover: /img/post_cover/NIO.jpg
tags:
    - Java
    - NIO
---
# Java NIO
## 1. Java NIO概述
Java NIO(New IO Non Blocking IO)是从`java1.4`版本开始引入的一个新的IO API,可以**替代标准的Java IO API**。`NIO`与原来的`IO`有同样的作用和目的，但是使用的方式完全不同，`NIO`支持面向缓冲区的、基于通道的IO操作。`NIO`将以更加高效的方式进行文件的读写操作。
### IO VS NIO
![IO](IO.png)  
![NO](NIO.png)  

| IO                    | NIO                           |
|-----------------------|-------------------------------|
| 面向流（Stream Oriented）            | 面向缓冲区（Buffer Oriented） |
| 阻塞IO（Blocking IO） | 非阻塞IO（Non Bloking IO）    |
| 无                    | 选择器（Selectors）           |

## 2. 通道（Channel）和缓冲区（Buffer）
通道(`Channel`)表示打开到IO设备（例如：文件、套接字）的连接。若需要使用NIO系统，需要获取用于连接IO设备的通道以及用于容纳数据的缓冲区。然后操作缓冲区，对数据进行处理。  
简而言之，`Channel`负责传输，`Buffer`负责存储。  
### 2.1 缓冲区（Buffer）
在Java NIO中负责数据的存储，缓冲区就是数组，用于存储不同数据类型的数据。
#### 2.1.1 缓冲区基本操作  

根据数据类型的而不同（`boolean`除外），提供了相应类型的缓冲区  
- `ByteBuffer`  
- `CharBuffer`  
- `ShortBuffer`  
- `IntBuffer`  
- `LongBuffer`  
- `FloatBuffer`  
- `DoubleBuffer`  

上述缓冲区的管理方式几乎一致，通过`allocate()`获取缓冲区。  

缓冲区存取数据的两个核心方法  
- `put()` 存入数据到缓冲区
- `get()` 获取缓冲区中的数据

缓冲区中的四个核心属性
> Invariants: mark <= position <= limit <= capacity  

- `private int mark = -1;` 标记，表示记录当前position的位置，可以通过rset()恢复到mark的位置
- `private int position = 0;` 位置， 表示缓冲区中正在操作数据的位置。
- `private int limit;`  界限，缓冲区中可以操作数据的大小。（limit后面的数据是不能进行操作的）
- `private int capacity;`  容量，缓冲器中最大存储数据的容量。一旦声明，无法改变。  

代码示例
```java
public class BufferTest {

    @Test
    public void test2(){
        String s = "abcde";
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        buffer.put(s.getBytes());
        buffer.flip();
        byte[] dst = new byte[buffer.limit()];
        buffer.get(dst,0,2);
        System.out.println(new String(dst,0,2));
        System.out.println("position = " + buffer.position());

        // mark()标记
        buffer.mark();

        buffer.get(dst,2,2);
        System.out.println(new String(dst,2,2));
        System.out.println("position = " + buffer.position());
        // 判断缓冲区是否有可操作的数据
        if(buffer.hasRemaining()){
            System.out.println(buffer.remaining());
        }
        // reset() 恢复到mark的位置
        buffer.reset();
        System.out.println("position = " + buffer.position());

        // 判断缓冲区是否有可操作的数据
        if(buffer.hasRemaining()){
            System.out.println(buffer.remaining());
        }
    }

    @Test
    public void test1(){
        // 分配一个指定大小的缓冲区
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        System.out.println("---------- allocate() ----------");
        System.out.println("mark = " + byteBuffer.mark());
        System.out.println("position = " + byteBuffer.position());
        System.out.println("limit = " + byteBuffer.limit());
        System.out.println("capacity = " + byteBuffer.capacity());
        // 使用put()将数据存入到缓冲区
        String str= "ABCDE";
        byteBuffer.put(str.getBytes());
        System.out.println("---------- put() ----------");
        System.out.println("mark = " + byteBuffer.mark());
        System.out.println("position = " + byteBuffer.position());
        System.out.println("limit = " + byteBuffer.limit());
        System.out.println("capacity = " + byteBuffer.capacity());
        // 切换数据模式
        byteBuffer.flip();
        System.out.println("---------- flip() ----------");
        System.out.println("mark = " + byteBuffer.mark());
        System.out.println("position = " + byteBuffer.position());
        System.out.println("limit = " + byteBuffer.limit());
        System.out.println("capacity = " + byteBuffer.capacity());
        // 使用get()方法读取缓冲区中的方法
        byte[] dst = new byte[byteBuffer.limit()];
        byteBuffer.get(dst);
        System.out.println(new String(dst,0,dst.length));
        System.out.println("---------- get() ----------");
        System.out.println("mark = " + byteBuffer.mark());
        System.out.println("position = " + byteBuffer.position());
        System.out.println("limit = " + byteBuffer.limit());
        System.out.println("capacity = " + byteBuffer.capacity());

        // rewind() 可重复读
        byteBuffer.rewind();
        System.out.println("---------- rewind() ----------");
        System.out.println("mark = " + byteBuffer.mark());
        System.out.println("position = " + byteBuffer.position());
        System.out.println("limit = " + byteBuffer.limit());
        System.out.println("capacity = " + byteBuffer.capacity());

        // 清空缓冲区，但是缓冲区中的数据依然存在，但是处于“被遗忘状态”
        byteBuffer.clear();
        System.out.println("---------- clear() ----------");
        System.out.println("mark = " + byteBuffer.mark());
        System.out.println("position = " + byteBuffer.position());
        System.out.println("limit = " + byteBuffer.limit());
        System.out.println("capacity = " + byteBuffer.capacity());

        byteBuffer.get(dst);
        System.out.println(new String(dst,0,dst.length));
    }
}

```

#### 2.1.2 直接缓冲区 VS 非直接缓冲区 
> **非直接缓冲区**，通过`allocate()`方法非直接缓冲区，将缓冲区建立在`JVM`的内存中。  
> **直接缓冲区**，通过`allocateDirect()`方法分配直接缓冲区，将缓冲区建立在物理内存中。可以提高效率。 

![直接缓冲区](直接缓冲区.jpeg)  
![非直接缓冲区](非直接缓冲区.jpeg)
```java
public class BufferTest {

    @Test
    public void test3(){
        ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024);
        System.out.println(byteBuffer.isDirect());
    }

}

```
### 2.2 通道（Channel）
通道（`Channel`）由`java.nio.channels`包定义的。`Channel`表示IO源与目标打开的连接。`Channel`类似于传统的“流”。只不过`Channel`本身不能直接访问数据，`Channel`只能与`Buffer`进行交互。
#### 2.2.1 Channel的原理与获取
应用程序与磁盘之间的数据写入或者读出，都需要由用户地址空间和内存地址空间之间来回复制数据，内存地址空间中的数据通过操作系统层面的IO接口，完成与磁盘的数据存取。在应用程序调用这些系统IO接口时，由CPU完成一系列调度、任务分配，早先这些IO接口都是由CPU独立负责。所以当发生大规模读写请求时，CPU的占用率很高。  
![Channel](Channel_01.png)  
之后，操作系统为了避免CPU完全被各种IO接口调用占用，引入了DMA（直接存储器存储）。当应用程序对操作系统发出一个读写请求时，会由DMA先向CPU申请权限，申请到权限之后，内存地址空间与磁盘之间的IO操作就全由DMA来负责操作。这样，在读写请求的过程中，CPU不需要再参与，CPU去做其他事情。当然，DMA来独立完成数据在磁盘与内存空间中的来去，需要借助于DMA总线。但是当DMA总线过多时，大量的IO操作也会造成总线冲突，即也会影响最终的读写性能。  
![Channel](Channel_02.png)  
为了避免DMA总线冲突对性能的影响，后来便有了通道的方式。通道，它是一个完全独立的处理器。CPU是中央处理器，通道本身也是一个处理器，专门负责IO操作。既然是处理器，通道有自己的IO命令，与CPU无关。它更适用于大型的IO操作，性能更高。  
![Channel](Channel_03.png)
##### 总结
- 直接存储器DMA有独立总线。
- 但在大量数据面前，可能会存在总线冲突，还是需要CPU来处理。
- 通道是一个独立的处理器
- DMA方式还是需要向CPU申请DMA总线的。
- 通道有自己的处理器，适合与大量IO请求的场景，数据传输直接通过通道进行传输，不再需要请求CPU  

#### 2.2.2 Channel的基本操作  
##### 通道的主要实现类
```java
java.nio.channels.Channel接口
    |-- FileChannel 用于本地文件数据传输
  	|-- SocketChannel 用于网络，TCP
  	|-- ServerSocketChannel 用于网络，TCP
  	|-- DatagramChannel 用于网络，UDP
```
##### 获取通道
1. Java针对支持通道的类提供了`getChannel()`方法
本地IO
- FileInputStream/FileOutputStream
- RandomAccessFile
网络IO
- Socket
- ServerSocket
- DatagramSocket

```java
/**
     * 利用通道完成文件的复制(非直接缓冲区)
     */
    @Test
    public void test1(){
        FileInputStream fileInputStream = null;
        FileOutputStream fileOutputStream = null;
        FileChannel inChannel = null;
        FileChannel outChannel = null;
        try {
            fileInputStream = new FileInputStream("classpath://../resource/channel/1.png");
            fileOutputStream = new FileOutputStream("classpath://../resource/channel/2.png");
            // 1. 获取通道
            inChannel = fileInputStream.getChannel();
            outChannel = fileOutputStream.getChannel();
            // 2. 分配指定大小的缓冲区
            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
            // 3. 将通道中的数据放入缓冲区
            while(inChannel.read(byteBuffer) != -1){
                byteBuffer.flip(); // 切换到读取数据模式
                // 4. 将缓冲区的数据写入到通道中
                outChannel.write(byteBuffer);
                byteBuffer.clear();
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (outChannel != null){
                try {
                    outChannel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            if (inChannel !=null){
                try {
                    inChannel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            if (fileOutputStream != null){
                try {
                    fileOutputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if(fileInputStream != null){
                try {
                    fileInputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
```

2. 在JDK1.7中的NIO.2针对各个通道童工了静态方法`open()`  

```java
/**
     * 利用通道完成文件的复制（直接缓冲区，内存映射文件）
     */
    @Test
    public void test2() throws IOException {
        FileChannel inChannel = FileChannel.open(Paths.get("D:\\idea_projects\\java-example\\java-chapter-NIO\\resource\\channel\\1.png"), StandardOpenOption.READ);
        // 注意：StandardOpenOption的CREATE_NEW代表如果已存在则创建失败；CREATE代表如果已存在则覆盖
        // FileChannel outChannel = FileChannel.open(Paths.get("classpath://../resource/channel/3.png"), StandardOpenOption.WRITE, StandardOpenOption.CREATE_NEW);
        //注意：因为下面从通道得到的映射文件缓冲区的映射模式是读写模式，而这个outChannel只有写的打开选项，所以是不够，还要加入读配置。
        FileChannel outChannel = FileChannel.open(Paths.get("D:\\idea_projects\\java-example\\java-chapter-NIO\\resource\\channel\\3.png"), StandardOpenOption.WRITE, StandardOpenOption.READ,StandardOpenOption.CREATE_NEW);

        // 内存映射文件
        //这种利用通道通过映射文件建立直接缓冲区的方式和用缓冲区allocateDirect(int)的方式，两者的原理是一模一样的！
        //只是申请直接缓冲区的方式不同。
        //申请的空间都在物理内存中。
        //注意：申请直接缓冲区，仅仅适用于ByteBuffer缓冲区类型，其他缓冲区类型不支持。
        //与之前的通过流获得的通道不同，这种通过映射文件的方式是直接把数据通过映射文件放到物理内存中，还需要通道进行传输吗？是不是就不用了吧。我现在只需要直接向直接缓冲区中放就可以了，不需要通道。
        //所以与之前相比，获取通道的操作都省去了，直接操作缓冲区即可。
        MappedByteBuffer inMappedByteBuffer = inChannel.map(FileChannel.MapMode.READ_ONLY, 0, inChannel.size());
        MappedByteBuffer outMappedByteBuffer = outChannel.map(FileChannel.MapMode.READ_WRITE, 0, inChannel.size());

        // 直接使用缓冲区进行数据的读写操作
        byte[] dst = new byte[inMappedByteBuffer.limit()];
        inMappedByteBuffer.get(dst);
        outMappedByteBuffer.put(dst);

        // 关闭通道
        inChannel.close();
        outChannel.close();
    }
```

3. 在JDK1.7中的NIO.2的Files工具类的`newByteChannel()`  
##### 通道之间数据传输
- transTo()
- transFrom()  

```java
/**
     * 通道之间数据传输（直接缓冲区）
     */
    @Test
    public void test3() throws IOException {

        FileChannel inChannel = FileChannel.open(Paths.get("D:\\idea_projects\\java-example\\java-chapter-NIO\\resource\\channel\\1.png"), StandardOpenOption.READ);
        FileChannel outChannel = FileChannel.open(Paths.get("D:\\idea_projects\\java-example\\java-chapter-NIO\\resource\\channel\\4.png"), StandardOpenOption.WRITE, StandardOpenOption.READ,StandardOpenOption.CREATE);

        // inChannel.transferTo(0,inChannel.size(),outChannel);
        outChannel.transferFrom(inChannel,0,inChannel.size());
        inChannel.close();
        outChannel.close();
    }
```
##### 分散（Scatter）与聚集(Gather)
> 分散读取（`Scatter Reads`）,将通道中数据分散到多个缓冲区中  
> 聚集写入（`Gather Writes`）,将多个缓冲区中的数据聚集到通道中
```java
/**
     * 分散和聚集
     * @throws IOException
     */
    @Test
    public void test4() throws IOException {
       RandomAccessFile randomAccessFile = new RandomAccessFile("classpath://../resource/channel/1.txt","rw");
        // 1. 获取通道
        FileChannel channel = randomAccessFile.getChannel();
        // 2. 分配指定大小的缓冲区
        ByteBuffer byteBuffer1 = ByteBuffer.allocate(100);
        ByteBuffer byteBuffer2 = ByteBuffer.allocate(1024);
        // 3. 分散读取
        ByteBuffer[] byteBuffers = {byteBuffer1, byteBuffer2};
        channel.read(byteBuffers);

        for (ByteBuffer byteBuffer:byteBuffers) {
            byteBuffer.flip();
        }
        System.out.println(new String(byteBuffers[0].array(),0,byteBuffers[0].limit()));
        System.out.println("--------------------");
        System.out.println(new String(byteBuffers[1].array(),0,byteBuffers[1].limit()));
        
        // 4. 聚集写入
        RandomAccessFile randomAccessFile1 = new RandomAccessFile("classpath://../resource/channel/2.txt", "rw");
        FileChannel channel1 = randomAccessFile1.getChannel();
        channel1.write(byteBuffers);
    }
```
##### 字符集（Charset）

- 编码，字符串->字节数组
- 解码，字节数组->字符串

```java
/**
     * 字符集
     */
    @Test
    public void test6() throws CharacterCodingException {
        Charset gbkCharset = Charset.forName("GBK");
        // 获取编码器
        CharsetEncoder gbkEncoder = gbkCharset.newEncoder();
        // 获取解码器
        CharsetDecoder gbkDecoder = gbkCharset.newDecoder();

        CharBuffer charBuffer = CharBuffer.allocate(1024);
        charBuffer.put("你好，世界！");
        charBuffer.flip();
        // 编码
        ByteBuffer byteBuffer = gbkEncoder.encode(charBuffer);
        for (int i = 0; i < byteBuffer.limit(); i++) {
            System.out.println(byteBuffer.get());
        }
        // 解码
        byteBuffer.flip();
        CharBuffer charBuffer1 = gbkDecoder.decode(byteBuffer);
        System.out.println(charBuffer1.toString());
        System.out.println("------------------------");

        Charset utf8Charset = Charset.forName("UTF-8");
        // CharsetDecoder charsetDecoder = utf8Charset.newDecoder();
        //CharBuffer decode = charsetDecoder.decode(byteBuffer);
        byteBuffer.flip();
        CharBuffer decode = utf8Charset.decode(byteBuffer);
        System.out.println(decode.toString());
    }
    @Test
    public void test5(){
        Map<String, Charset> charsetMap = Charset.availableCharsets();
        Set<Map.Entry<String, Charset>> entrySet = charsetMap.entrySet();
        for (Map.Entry<String,Charset> entry: entrySet) {
            System.out.println(entry.getKey() + "=" + entry.getValue());
        }
    }

```
  

## 3. NIO-阻塞与非阻塞
 传统的 IO 流都是阻塞式的。也就是说，当一个线程调用 `read()` 或 `write()`时，该线程被阻塞，直到有一些数据被读取或写入，该线程在此期间不能执行其他任务。  
 ![传统IO](传统IO.png)  
 因此，在完成网络通信进行 IO 操作时，由于线程会阻塞，所以服务器端必须为每个客户端都提供一个独立的线程进行处理，当服务器端需要处理大量客户端时，性能急剧下降。  
 ![传统IO多线程](传统IO多线程.png)  
 Java NIO 是非阻塞模式的。当线程从某通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。线程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出通道。因此， NIO 可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。  
 ![NIO非阻塞方式](NIO非阻塞方式.png)   
 选择器和通道的关系：通道注册到选择器上，选择器监控通道。   
 当某一个通道上，某一个事件准备就绪时，那么选择器才会将这个通道分配到服务器端一个或多个线程上，再继续运行。比如说当客户端发送一些数据给服务器端，只有当客户端的所有数据都准备就绪时，选择器才会将这个注册的通道分配到服务器端的一个或者多个线程上。那就意味这，如果客户端的线程没有将数据准备就绪时，服务器端的线程可以执行其他任务，而不必阻塞在那里。  
 ### 3.1 选择器（Selector）与通道（Channel）的关系
选择器（`Selector`） 是 `SelectableChannle` 对象的多路复用器， `Selector `可以同时监控多个 `SelectableChannel` 的 IO 状况，也就是说，利用 Selector可使一个单独的线程管理多个 `Channel`。 `Selector` 是非阻塞 IO 的核心。 
![SelectableCahnnel](SelectableChannel.png)   
**注意：** FileChannel切换为非阻塞模式！！！非阻塞模式是相对于网络IO而言的。选择器主要监控网络Channel。  （FileChannel不是可作为选择器复用的通道！FileChannel不能注册到选择器Selector！FileChannel不能切换到非阻塞模式！FileChannel不是SelectableChannel的子类！）  

### 3.2 网络NIO示例（阻塞式 TCP协议）
#### 阻塞IO模式：客户端向服务端发送文件
```java
package com.java.demo.selector;

import org.junit.Test;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;

/**
 * 一、使用NIO完成网络通信的三个核心概念
 * 1. 通道（Channel）,负责连接
 *      java.nio.channels.Channel接口
 *          |-- SelectableChannel
 *              |--SocketChannel          TCP
 *              |-- ServerSocketChannel   TCP
 *              |-- DatagramChannel       UDP
 *
 *              |--Pipe.SinkChannel
 *              |--Pipe.SourceChannel
 *    注意：FileChannel切换为非阻塞模式！！！非阻塞模式是相对于网络IO而言的。选择器主要监控网络Channel。
 * 2. 缓冲区（Buffer）,负责数据的存取
 * 3. 选择器（Selector）,是SelectableChannel的多路复用器。用于监控SelectableChannel的IO状况
 */
public class BlockingNIOTest {
    /**
     * 客户端
     */
    @Test
    public void client() throws IOException {
        // 1. 获取通道
        SocketChannel socketChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 8888));
        // 2. 分配指定大小的缓冲区
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        // 3. 从本地读取文件并发送到服务器
        FileChannel inChannel = FileChannel.open(Paths.get("D:\\idea_projects\\java-example\\java-chapter-NIO\\resource\\channel\\1.png"), StandardOpenOption.READ);
        while (inChannel.read(byteBuffer)!=-1){
            byteBuffer.flip();
            socketChannel.write(byteBuffer);
            byteBuffer.clear();
        }
        // 4. 关闭通道
        inChannel.close();
        socketChannel.close();
    }

    /**
     * 服务端
     */
    @Test
    public void server() throws IOException {
        // 1. 获取通道
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        // 2. 绑定连接
        serverSocketChannel.bind(new InetSocketAddress(8888));
        // 3.获取客户端的连接
        SocketChannel socketChannel = serverSocketChannel.accept();
        // 4. 接收客户端传输的数据，并保存在本地
        FileChannel outChannel = FileChannel.open(Paths.get("D:\\idea_projects\\java-example\\java-chapter-NIO\\resource\\channel\\5.png"), StandardOpenOption.WRITE,StandardOpenOption.CREATE);
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        while(socketChannel.read(byteBuffer)!=-1){
            byteBuffer.flip();
            outChannel.write(byteBuffer);
            byteBuffer.clear();
        }
        // 5.关闭通道
        socketChannel.close();
        outChannel.close();
        serverSocketChannel.close();
    }
}

```
#### 阻塞IO模式：服务端向客户端发送反馈信息
```java
package com.java.demo.selector;

import org.junit.Test;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;

public class BlockingNIOTest1 {

    @Test
    public void client() throws IOException {
        SocketChannel socketChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 8888));
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        FileChannel inFileChannel = FileChannel.open(Paths.get("D:\\idea_projects\\java-example\\java-chapter-NIO\\resource\\channel\\1.png"), StandardOpenOption.READ);
        while(inFileChannel.read(byteBuffer) != -1){
            byteBuffer.flip();
            socketChannel.write(byteBuffer);
            byteBuffer.clear();
        }

        //在阻塞IO下，如果关闭socketChannel，那么服务端不知道客户端是否已经把所有数据发完，所以会一直阻塞。
        socketChannel.shutdownOutput();
        //另一种方法就是把这个线程切换成非阻塞模式

        //接收服务端反馈
        int len = 0;
        while((len = socketChannel.read(byteBuffer)) !=-1){
            byteBuffer.flip();
            System.out.println(new String(byteBuffer.array(),0,len));
        }

        inFileChannel.close();
        socketChannel.close();
    }

    @Test
    public void server() throws IOException {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(8888));
        SocketChannel socketChannel = serverSocketChannel.accept();
        FileChannel outFileChannel = FileChannel.open(Paths.get("D:\\idea_projects\\java-example\\java-chapter-NIO\\resource\\channel\\6.png"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        while(socketChannel.read(byteBuffer) != -1){
            byteBuffer.flip();
            outFileChannel.write(byteBuffer);
            byteBuffer.clear();
        }

        //发送反馈给客户端
        byteBuffer.put("服务端接收数据成功！".getBytes());
        byteBuffer.flip();
        socketChannel.write(byteBuffer);

        outFileChannel.close();
        socketChannel.close();
        serverSocketChannel.close();
    }

}

```

 ### 3.3 网络NIO示例（非阻塞式 TCP协议）
 #### 非阻塞IO模式：客户端向服务端发送数据
 ```java
package com.java.demo.selector;

import org.junit.Test;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Date;
import java.util.Iterator;
import java.util.Scanner;

public class NonBlockingNIOTest {

    @Test
    public void client() throws IOException {
        // 1. 获取通道
        SocketChannel socketChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 8888));
        // 2. 切换到非阻塞模式
        socketChannel.configureBlocking(false);
        // 3. 分配缓冲区
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        // 4. 发送数据给服务端
        Scanner scanner = new Scanner(System.in);
        while(scanner.hasNext()){
            String inputStr=scanner.next();
            byteBuffer.put((new Date().toString() + "\n" + inputStr).getBytes());
            byteBuffer.flip();
            socketChannel.write(byteBuffer);
            byteBuffer.clear();
        }
        scanner.close();
//        byteBuffer.put(new Date().toString().getBytes());
//        byteBuffer.flip();
//        socketChannel.write(byteBuffer);
//        byteBuffer.clear();

        // 5. 关闭通道
        socketChannel.close();
    }

    @Test
    public void server() throws IOException {
        // 1. 获取通道
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        // 2. 切换为非阻塞模式
        serverSocketChannel.configureBlocking(false);
        // 3. 绑定连接
        serverSocketChannel.bind(new InetSocketAddress(8888));
        // 4. 获取选择器
        Selector selector = Selector.open();
        //5、将通道注册到选择器上(第二个选项参数叫做选择键，用于告诉选择器需要监控这个通道的什么状态或者说什么事件（读、写、连接、接受）)
        //选择键是整型值，如果需要监控该通道的多个状态或事件，可以将多个选择键用位运算符“或”“|”来连接
        //这里服务端首先要监听客户端的接受状态
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        // 6. 轮询式的获取选择器上已经"准备就绪"的事件
        while (selector.select() > 0){
            // 7. 获取当前选择器中所有注册的选择键
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            // 8. 迭代获取”准备就绪”的事件
            while(iterator.hasNext()){
                SelectionKey selectionKey = iterator.next();
                // 9. 判断具体是什么事件住呢被就绪
                if (selectionKey.isAcceptable()){
                    // 10. 若“接受就绪”,获取客户端连接
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    // 11. 切换为非阻塞模式
                    socketChannel.configureBlocking(false);
                    // 12. 将该通道注册到选择器
                    socketChannel.register(selector,SelectionKey.OP_READ);
                }else if(selectionKey.isReadable()){
                    // 13. 获取当前选择器上“读就绪”状态的通道
                    SocketChannel socketChannel = (SocketChannel)selectionKey.channel();
                    // 14. 读取数据
                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                    int length = 0;
                    while((length = socketChannel.read(byteBuffer))>0){
                        byteBuffer.flip();
                        System.out.println(new String(byteBuffer.array(),0,length));
                        byteBuffer.clear();
                    }
                }
                // 15. 取消选择键SelectionKey
                // 注意：SelectionKey使用完之后，一定要取消掉！！否则一直有效，如一个通道已经连接完成accept，如果不取消，下次还有这个连接完成。
                iterator.remove();
            }
        }
    }
}

```
### 3.4 网络NIO示例（非阻塞式 UDP协议）
```java
package com.java.demo.selector;

import org.junit.Test;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.DatagramChannel;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.util.ArrayList;
import java.util.Date;
import java.util.Iterator;
import java.util.Scanner;

public class NonBlockingNIOTest1 {


    @Test
    public void send() throws IOException {
        // 1. 获取通道
        DatagramChannel datagramChannel = DatagramChannel.open();
        // 2. 切换为非阻塞模式
        datagramChannel.configureBlocking(false);
        // 3. 分配指定大小的缓冲区
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        // 4. 发送数据
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNext()){
            String inputStr = scanner.next();
            byteBuffer.put((new Date().toString() + "\n" +inputStr).getBytes());
            byteBuffer.flip();
            datagramChannel.send(byteBuffer,new InetSocketAddress("127.0.0.1",8888));
            byteBuffer.clear();
        }
        scanner.close();
        datagramChannel.close();
    }

    @Test
    public void receive() throws IOException {
        // 1. 获取通道
        DatagramChannel datagramChannel = DatagramChannel.open();
        // 2. 设置为非阻塞模式
        datagramChannel.configureBlocking(false);
        // 3. 绑定连接
        datagramChannel.bind(new InetSocketAddress(8888));
        // 4. 获取选择器
        Selector selector = Selector.open();
        // 5.将通道注册到选择器上
        datagramChannel.register(selector, SelectionKey.OP_READ);
        while (selector.select()>0){
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()){
                SelectionKey selectionKey = iterator.next();
                if(selectionKey.isReadable()){
                    // 6. 分配指定大小的缓冲区
                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                    datagramChannel.receive(byteBuffer);
                    System.out.println(new String(byteBuffer.array(),0,byteBuffer.limit()));
                    byteBuffer.clear();
                }
                iterator.remove();
            }
        }
        datagramChannel.close();
    }
}

```

## 4. NIO-管道（Pipe）  
Java NIO 管道是2个线程之间的单向数据连接。Pipe有一个source通道和一个sink通道。数据会被写到sink通道，从source通道读取。  
![pipe](pipe.png)  

代码示例
```java
package com.java.demo.pipe;

import org.junit.Test;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.Pipe;

public class PipeTest {

    @Test
    public void test() throws IOException {
        // 1. 获取管道
        Pipe pipe = Pipe.open();
        // 2. 将缓冲区数据写入到管道
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        byteBuffer.put("通过单向管道发送数据".getBytes());
        Pipe.SinkChannel sinkChannel = pipe.sink(); // Pipe.SinkChannel是Pipe的内部类
        byteBuffer.flip();
        sinkChannel.write(byteBuffer);
        byteBuffer.clear();

        //3、读取缓冲区中的数据（可以是另一个线程）
        Pipe.SourceChannel sourceChannel = pipe.source();//Pipe.SourceChannel是Pipe的内部类
        int len = sourceChannel.read(byteBuffer);
        byteBuffer.flip();
        System.out.println(new String(byteBuffer.array(),0,len));

        // 关闭通道
        sourceChannel.close();
        sinkChannel.close();
    }
}

```