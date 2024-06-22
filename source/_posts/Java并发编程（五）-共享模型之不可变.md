---
title: Java并发编程（五）-共享模型之不可变
date: 2024-06-16 17:23:18
categories: 
    - Java
    - 并发编程
cover: /img/post_cover/java_concurrency05.jpg
tags:
    - Java
    - 多线程
    - 并发编程
---
本章内容
- 不可变类使用
- 不可变类设计
- 无状态类设计
## 1. 日期转换的问题
### 问题提出
下面的代码在运行时，由于 `SimpleDateFormat` 不是线程安全的。
```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
for (int i = 0; i < 10; i++) {
 new Thread(() -> {
 try {
 log.debug("{}", sdf.parse("1951-04-21"));
 } catch (Exception e) {
 log.error("{}", e);
 }
 }).start();
}
```
有很大几率出现 `java.lang.NumberFormatException` 或者出现不正确的日期解析结果，例如
```
20:40:47 [Thread-4] c.DateFormatTest - {}
java.lang.NumberFormatException: For input string: ""
	at java.base/java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.base/java.lang.Long.parseLong(Long.java:702)
	at java.base/java.lang.Long.parseLong(Long.java:817)
	at java.base/java.text.DigitList.getLong(DigitList.java:195)
	at java.base/java.text.DecimalFormat.parse(DecimalFormat.java:2121)
	at java.base/java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1933)
	at java.base/java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1541)
	at java.base/java.text.DateFormat.parse(DateFormat.java:393)
	at com.java.demo.immutable.DateFormatTest.lambda$main$0(DateFormatTest.java:15)
	at java.base/java.lang.Thread.run(Thread.java:834)
20:40:47 [Thread-7] c.DateFormatTest - {}
java.lang.NumberFormatException: multiple points
	at java.base/jdk.internal.math.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at java.base/jdk.internal.math.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.base/java.lang.Double.parseDouble(Double.java:543)
	at java.base/java.text.DigitList.getDouble(DigitList.java:169)
	at java.base/java.text.DecimalFormat.parse(DecimalFormat.java:2126)
	at java.base/java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1933)
	at java.base/java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1541)
	at java.base/java.text.DateFormat.parse(DateFormat.java:393)
	at com.java.demo.immutable.DateFormatTest.lambda$main$0(DateFormatTest.java:15)
	at java.base/java.lang.Thread.run(Thread.java:834)
20:40:47 [Thread-6] c.DateFormatTest - {}
java.lang.NumberFormatException: multiple points
	at java.base/jdk.internal.math.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at java.base/jdk.internal.math.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.base/java.lang.Double.parseDouble(Double.java:543)
	at java.base/java.text.DigitList.getDouble(DigitList.java:169)
	at java.base/java.text.DecimalFormat.parse(DecimalFormat.java:2126)
	at java.base/java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1933)
	at java.base/java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1541)
	at java.base/java.text.DateFormat.parse(DateFormat.java:393)
	at com.java.demo.immutable.DateFormatTest.lambda$main$0(DateFormatTest.java:15)
	at java.base/java.lang.Thread.run(Thread.java:834)
20:40:47 [Thread-0] c.DateFormatTest - Sat Apr 21 00:00:00 CST 1951
20:40:47 [Thread-3] c.DateFormatTest - Sat Jan 19 00:00:00 CST 1957
20:40:47 [Thread-1] c.DateFormatTest - Sat Apr 21 00:00:00 CST 1951
20:40:47 [Thread-9] c.DateFormatTest - Mon Apr 21 00:00:00 CST 21
20:40:47 [Thread-5] c.DateFormatTest - Sat Apr 21 00:00:00 CST 1951
20:40:47 [Thread-8] c.DateFormatTest - Sat Apr 21 00:00:00 CST 1951
20:40:47 [Thread-2] c.DateFormatTest - Mon Apr 21 00:00:00 CST 2121

```
### 问题解决-同步锁
虽能解决问题，但带来的是性能上的损失，并不算很好：
```java 
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
for (int i = 0; i < 50; i++) {
 new Thread(() -> {
 synchronized (sdf) {
 try {
 log.debug("{}", sdf.parse("1951-04-21"));
 } catch (Exception e) {
 log.error("{}", e);
 }
 }
 }).start();
}
```

### 问题解决-不可变
如果一个对象在不能够修改其内部状态（属性），那么它就是线程安全的，因为不存在并发修改啊！这样的对象在
`Java` 中有很多，例如在 `Java 8` 后，提供了一个新的日期格式化类：
```java 
DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd");
for (int i = 0; i < 10; i++) {
 new Thread(() -> {
 LocalDate date = dtf.parse("2018-10-01", LocalDate::from);
 log.debug("{}", date);
 }).start();
}
```
可以看 `DateTimeFormatter` 的文档：
```
 @implSpec
 This class is immutable and thread-safe.

 @since 1.8
```
不可变对象，实际是另一种避免竞争的方式。

## 2. 不可变设计
另外一个 String 类也是不可变类，以他为例，说明一下不可变设计的要素
```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {

    /**
     * The value is used for character storage.
     *
     * @implNote This field is trusted by the VM, and is a subject to
     * constant folding if String instance is constant. Overwriting this
     * field after construction will cause problems.
     *
     * Additionally, it is marked with {@link Stable} to trust the contents
     * of the array. No other facility in JDK provides this functionality (yet).
     * {@link Stable} is safe here, because value is never null.
     */
    @Stable
    private final byte[] value;

    /**
     * The identifier of the encoding used to encode the bytes in
     * {@code value}. The supported values in this implementation are
     *
     * LATIN1
     * UTF16
     *
     * @implNote This field is trusted by the VM, and is a subject to
     * constant folding if String instance is constant. Overwriting this
     * field after construction will cause problems.
     */
    private final byte coder;

    /** Cache the hash code for the string */
    private int hash; 
	}
```
### 2.1 final 的使用
该类中所有的属性都是 final 的
- 属性用 final 修饰保证了该属性是只读的，不能修改
- 类用 final 修饰保证了该类中的方法不能被覆盖，防止子类无意间破坏不可变性

### 2.2 保护性拷贝
使用字符串时，也有一些修改相关的方法，比如 substring() 等，下面就是 substring() 的实现
```java
/**
     * Returns a string that is a substring of this string. The
     * substring begins with the character at the specified index and
     * extends to the end of this string. <p>
     * Examples:
     * <blockquote><pre>
     * "unhappy".substring(2) returns "happy"
     * "Harbison".substring(3) returns "bison"
     * "emptiness".substring(9) returns "" (an empty string)
     * </pre></blockquote>
     *
     * @param      beginIndex   the beginning index, inclusive.
     * @return     the specified substring.
     * @exception  IndexOutOfBoundsException  if
     *             {@code beginIndex} is negative or larger than the
     *             length of this {@code String} object.
     */
    public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = length() - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        if (beginIndex == 0) {
            return this;
        }
        return isLatin1() ? StringLatin1.newString(value, beginIndex, subLen)
                          : StringUTF16.newString(value, beginIndex, subLen);
    }
```
发现其内部就是调用 `String` 的构造方法创建一个新字符串，再看看这个构造，是否对 `final char[] value` 做出了修改：
```java
/*
	* Package private constructor. Trailing Void argument is there for
	* disambiguating it against other (public) constructors.
	*/
String(AbstractStringBuilder asb, Void sig) {
	byte[] val = asb.getValue();
	int length = asb.length();
	if (asb.isLatin1()) {
		this.coder = LATIN1;
		this.value = Arrays.copyOfRange(val, 0, length);
	} else {
		if (COMPACT_STRINGS) {
			byte[] buf = StringUTF16.compress(val, 0, length);
			if (buf != null) {
				this.coder = LATIN1;
				this.value = buf;
				return;
			}
		}
		this.coder = UTF16;
		this.value = Arrays.copyOfRange(val, 0, length << 1);
	}
}
```
结果发现也没有，构造新字符串对象时，会生成新的 `char[] value`，对内容进行复制 。这种通过创建副本对象来避
免共享的手段称之为**保护性拷贝（defensive copy）**
## 3. 享元模式
### 3.1 简介
定义 英文名称：`Flyweight pattern`. 当需要重用数量有限的同一类对象时
> wikipedia： A flyweight is an object that minimizes memory usage by sharing as much data aspossible with other similar objects  

出自 `Gang of Four` design patterns  
归类 Structual patterns  

### 3.2 体现
### 3.2.1 包装类
在 `JDK` 中 `Boolean`，`Byte`，`Short`，`Integer`，`Long`，`Character` 等包装类提供了 `valueOf` 方法，例如 `Long` 的
valueOf 会缓存 `-128~127` 之间的 `Long` 对象，在这个范围之间会重用对象，大于这个范围，才会新建 `Long` 对
象：
```java
public static Long valueOf(long l) {
 final int offset = 128;
 if (l >= -128 && l <= 127) { // will cache
 return LongCache.cache[(int)l + offset];
 }
 return new Long(l);
}
```
> 注意：
Byte, Short, Long 缓存的范围都是 -128-127
Character 缓存的范围是 0-127
Integer的默认范围是 -128-127
最小值不能变
但最大值可以通过调整虚拟机参数 ` 
-Djava.lang.Integer.IntegerCache.high` 来改变
Boolean 缓存了 TRUE 和 FALSE

#### 3.2.2 String 串池
#### 3.2.3 BigDecimal/BigInteger
单个的方法都是线程安全的，但是多个方法组合在一起，并并不是原子性的。

## 4. 自定义连接池
例如：一个线上商城应用，QPS 达到数千，如果每次都重新创建和关闭数据库连接，性能会受到极大影响。 这时
预先创建好一批连接，放入连接池。一次请求到达后，从连接池获取连接，使用完毕后再还回连接池，这样既节约
了连接的创建和关闭时间，也实现了连接的重用，不至于让庞大的连接数压垮数据库。
```java

package com.java.demo.immutable;

import lombok.extern.slf4j.Slf4j;

import java.sql.*;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.Executor;
import java.util.concurrent.atomic.AtomicIntegerArray;

@Slf4j(topic = "c.Pool")
public class Pool {

    // 连接池大小
    private final int poolSize;


    // 连接池对象数组
    private Connection[] connections;


    // 链接状态数组 0-空闲， 1-使用
    private AtomicIntegerArray states;


    public Pool(int poolSize) {
        this.poolSize = poolSize;
        this.connections = new Connection[poolSize];
        this.states = new AtomicIntegerArray(new int[poolSize]);
        for (int i = 0; i < poolSize; i++) {
            connections[i] = new MockConnection("连接" + (i + 1));
        }
    }

    /**
     * 使用连接
     * @return
     */
    public Connection getConnection(){
        while(true){
            for (int i = 0; i < poolSize; i++) {
                // 获取空闲连接
                if(states.get(i) == 0 && states.compareAndSet(i, 0, 1)){
                    log.debug("get connection:{}", connections[i]);
                    return connections[i];
                }
            }
            // 没有空闲连接，当前线程进入等待
            synchronized (this){
                try {
                    log.debug("wait...");
                    this.wait();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

	/**
     * 释放连接
     * @param connection
     */
    public void releaseConnection(Connection connection){
        for (int i = 0; i < poolSize; i++) {
            if (connections[i] == connection){
                states.set(i, 0);
                synchronized (this){
                    log.debug("release connection:{}", connection);
                    this.notifyAll();
                }
                break;
            }
        }
    }
}

class MockConnection implements Connection{

    private String name;

    public MockConnection(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "MockConnection{" +
                "name='" + name + '\'' +
                '}';
    }

    // 实现接口方法略
}
```
测试代码如下：
```java
package com.java.demo.immutable;

import java.sql.Connection;
import java.util.Random;

public class PoolTest {
    public static void main(String[] args) {
        Pool pool = new Pool(2);
        for (int i = 0; i < 5; i++) {
            new Thread(()->{
                Connection connection = pool.getConnection();
                try {
                    Thread.sleep(new Random().nextInt(1000));
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                pool.releaseConnection(connection);
            }, "线程" + (i+1)).start();
        }
    }
}

```
执行结果如下：
```bash
18:45:06 [线程5] c.Pool - wait...
18:45:06 [线程4] c.Pool - wait...
18:45:06 [线程3] c.Pool - wait...
18:45:06 [线程2] c.Pool - get connection:MockConnection{name='连接2'}
18:45:06 [线程1] c.Pool - get connection:MockConnection{name='连接1'}
18:45:06 [线程2] c.Pool - release connection:MockConnection{name='连接2'}
18:45:06 [线程3] c.Pool - wait...
18:45:06 [线程5] c.Pool - get connection:MockConnection{name='连接2'}
18:45:06 [线程4] c.Pool - wait...
18:45:06 [线程5] c.Pool - release connection:MockConnection{name='连接2'}
18:45:06 [线程3] c.Pool - get connection:MockConnection{name='连接2'}
18:45:06 [线程4] c.Pool - wait...
18:45:07 [线程3] c.Pool - release connection:MockConnection{name='连接2'}
18:45:07 [线程4] c.Pool - get connection:MockConnection{name='连接2'}
18:45:07 [线程1] c.Pool - release connection:MockConnection{name='连接1'}
18:45:07 [线程4] c.Pool - release connection:MockConnection{name='连接2'}
```

以上实现没有考虑：
- 连接的动态增长与收缩
- 连接保活（可用性检测）
- 等待超时处理
- 分布式 hash
对于关系型数据库，有比较成熟的连接池实现，例如c3p0, druid等 对于更通用的对象池，可以考虑使用apache
commons pool，例如redis连接池可以参考jedis中关于连接池的实现

## 5. final 原理
### 5.1 设置 final 变量的原理
理解了 volatile 原理，再对比 final 的实现就比较简单
```java
public class TestFinal {
 final int a = 20;
}
```
字节码
```text
0: aload_0
1: invokespecial #1 // Method java/lang/Object."<init>":()V
4: aload_0
5: bipush 20
7: putfield #2 // Field a:I
 <-- 写屏障
10: return
```
`final` 变量的赋值也会通过 `putfield` 指令来完成，同样在这条指令之后也会加入写屏障，保证在其它线程读到它的值时不会出现为 `0` 的情况  
### 5.2 获取 final 变量的原理
通过分析下面代码的字节码了解其原理。
```java
package com.java.demo.immutable;

public class FinalTest {

    final static int A = 10;
    final static int B = Short.MAX_VALUE+1;

    final int a = 20;
    final int b = Integer.MAX_VALUE;

    final void test1(){

    }

}

class UseFinal1{
    public void test(){
        System.out.println(FinalTest.A);
        System.out.println(FinalTest.B);
        System.out.println(new FinalTest().a);
        System.out.println(new FinalTest().b);
    }
}

class UseFinal2{
    public void test(){
        System.out.println(FinalTest.A);
    }
}

```
## 6. 无状态
在 `web` 阶段学习时，设计 `Servlet` 时为了保证其线程安全，都会有这样的建议，不要为 `Servlet` 设置成员变量，这种没有任何成员变量的类是线程安全的。   
> 因为成员变量保存的数据也可以称为状态信息，因此没有成员变量就称之为**无状态**。
