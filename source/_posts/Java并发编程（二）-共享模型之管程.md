---
title: Java并发编程（二） 共享模型之管程
date: 2022-05-31 10:55:29
categories: 
    - Java
    - 并发编程
cover: /img/post_cover/java_concurrency02.jpg
tags:
    - Java
    - 多线程
    - 并发编程
---
# Java并发编程（二） 共享模型之管程

## 1. 线程安全问题

> 两个线程对初始值为 0 的静态变量一个做自增，一个做自减，各做 5000 次，结果是 0 吗？  

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j(topic = "c.ThreadContextSwitchTest")
public class ThreadContextSwitchTest {
    private static int counter = 0;
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 5000; i++) {
                counter++;
            }
        }, "t1");

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 5000; i++) {
                counter--;
            }
        }, "t2");

        t1.start();
        t2.start();
        t1.join();
        t2.join();
        log.debug("counter = {}",counter);
    }
}

```

执行结果:

- 负数  

```cmd
10:05:34 [main] c.ThreadContextSwitchTest - counter = -867
```

- 正数  

```cmd
10:06:52 [main] c.ThreadContextSwitchTest - counter = 775
```

- 0  

```cmd
10:06:34 [main] c.ThreadContextSwitchTest - counter = 0
```

### 1.1 原因分析

以上的结果可能是正数、负数、零。为什么呢？因为 Java 中对静态变量的自增，自减并不是原子操作，要彻底理
解，必须从字节码来进行分析
例如对于 i++ 而言（i 为静态变量），实际会产生如下的 JVM 字节码指令：

```java
GETSTATIC com/java/demo/monitor/ThreadContextSwitchTest.counter : I
    ICONST_1
    IADD
    PUTSTATIC com/java/demo/monitor/ThreadContextSwitchTest.counter : I
```

而对应 i-- 也是类似：

```java
GETSTATIC com/java/demo/monitor/ThreadContextSwitchTest.counter : I
    ICONST_1
    ISUB
    PUTSTATIC com/java/demo/monitor/ThreadContextSwitchTest.counter : I
```

而 Java 的内存模型如下，完成静态变量的自增，自减需要在主存和工作内存中进行数据交换：
![线程安全问题](线程安全问题.png)  
如果是单线程以上 8 行代码是顺序执行（不会交错）没有问题：  
![正常情况](正常.png)  
但多线程下这 8 行代码可能交错运行：
出现负数的情况：  
![负数情况](负数.png)  
出现正数的情况：  
![正数情况](正数.png)

### 1.2 临界区 (Critical Section)

- 一个程序运行多个线程本身是没有问题的
- 问题出在多个线程访问**共享资源**  
  - 多个线程读**共享资源**其实也没有问题  
  - 在多个线程对**共享资源**读写操作时发生指令交错，就会出现问题
- 一段代码块内如果存在对**共享资源**的多线程读写操作，称这段代码块为**临界区**  
  例如，下面代码中的临界区  

```java
static int counter = 0;
static void increment() 
// 临界区
{ 
 counter++;
}
static void decrement() 
// 临界区
{ 
 counter--;
}
```

### 1.3 竞态条件 (Race Condition)

多个线程在**临界区**内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了**竞态条件**  

## 2. synchronized解决方案

为了避免临界区的竞态条件发生，有多种手段可以达到目的。

- 阻塞式的解决方案：synchronized，Lock
- 非阻塞式的解决方案：原子变量
  下面介绍阻塞式的解决方案：**synchronized**，来解决上述问题，即俗称的**对象锁**，它采用**互斥**的方式让同一
  时刻至多只有一个线程能持有**对象锁**，其它线程再想获取这个**对象锁**时就会**阻塞**住。这样就能保证拥有锁的线程可以安全的执行临界区内的代码，不用担心**线程上下文切换**。

> **注意**  
> 虽然 java 中互斥和同步都可以采用 synchronized 关键字来完成，但它们还是有区别的：  

- **互斥**是保证临界区的竞态条件发生，同一时刻只能有一个线程执行临界区代码
- **同步**是由于线程执行的先后、顺序不同、需要一个线程等待其它线程运行到某个点  

### 2.1 synchronized

#### 语法

```java
synchronized(对象) // 线程1， 线程2(blocked)
{
 临界区
}

```

#### 解决方案

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j(topic = "c.SynchronizedTest")
public class SynchronizedTest {
    private static int counter = 0;

    static Object lock = new Object();
    public static void main(String[] args) throws InterruptedException {

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 5000; i++) {
                synchronized (lock){
                    counter++;
                }
            }
        }, "t1");

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 5000; i++) {
                synchronized (lock){
                    counter--;
                }
            }
        }, "t2");
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        
        log.debug("counter = {}",counter);
    }
}

```

多次运行，执行结果都是0  

```cmd
12:03:43 [main] c.SynchronizedTest - counter = 0
```

用图来表示  
![图解](图解.png)  

#### 思考

`synchronized` 实际是用**对象锁**保证了**临界区内代码的原子性**，临界区内的代码对外是不可分割的，不会被线程切换所打断。  
为了加深理解，请思考下面的问题

- 如果把 synchronized(obj) 放在 for 循环的外面，如何理解？-- 原子性
- 如果 t1 synchronized(obj1) 而 t2 synchronized(obj2) 会怎样运作？-- 锁对象
- 如果 t1 synchronized(obj) 而 t2 没有加会怎么样？如何理解？-- 锁对象

#### 面向对象改进

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j(topic = "c.SynchronizedTest01")
public class SynchronizedTest01 {
    public static void main(String[] args) throws InterruptedException {
        CounterLock counterLock = new CounterLock();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 5000; i++) {
                counterLock.increment();
            }
        }, "t1");
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 5000; i++) {
                counterLock.decrement();
            }
        }, "t2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        log.debug("counter = {}",counterLock.getCounter());
    }
}
class CounterLock{

    private int counter = 0;

    public void increment(){
        synchronized (this){
            counter++;
        }
    }

    public void decrement(){
        synchronized (this){
            counter--;
        }
    }

    public int getCounter(){
        synchronized (this){
            return counter;
        }
    }
}
```

## 3.方法上的synchornized

### synchornized加在成员方法上

```java
class Test{
 public synchronized void test() {
 
 }
}
等价于
class Test{
 public void test() {
 synchronized(this) {
 
 }
 }
}
```

### synchornized加在静态方法上

```java
class Test{
 public synchronized static void test() {
 }
}
等价于
class Test{
 public static void test() {
 synchronized(Test.class) {
 
 }
 }
}

```

### 不加 synchronized 的方法

不加 synchronzied 的方法就好比不遵守规则的人，不去老实排队（好比翻窗户进去的）

### 所谓的“线程八锁”

其实就是考察 synchronized 锁住的是哪个对象

- 情况1：输出结果为`1 2或者2 1`

```java
@Slf4j(topic = "c.Number")
class Number{
    public synchronized void a() {
        log.debug("1");
    }

    public synchronized void b(){
        log.debug("2");
    }
}

public static void test01(){
        Number n1 = new Number();
        new Thread(()->{ log.debug("begin..."); n1.a();}).start();
        new Thread(()->{ log.debug("begin..."); n1.b();}).start();
    }
```

- 情况2：输出结果为`1s后1 2`或者为`2 1s后 1`

```java
@Slf4j(topic = "c.Number")
class Number{
    public synchronized void a() {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("1");
    }

    public synchronized void b(){
        log.debug("2");
    }
}
```

- 情况3：输出结果为`3 1s后 1 2`或`2 3 1s后 1`或`3 2 1s后 1`

```java
@Slf4j(topic = "c.Number")
class Number{
    public synchronized void a() {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("1");
    }

    public synchronized void b(){
        log.debug("2");
    }

    public void c(){
        log.debug("3");
    }
}

 public static void test03(){
        Number n1 = new Number();
        new Thread(()->{ log.debug("begin..."); n1.a();}).start();
        new Thread(()->{ log.debug("begin..."); n1.b();}).start();
        new Thread(()->{ log.debug("begin..."); n1.c();}).start();

    }
```

- 情况4: 输出结果为`2 1s后 1`

```java
@Slf4j(topic = "c.Number")
class Number{
    public synchronized void a() {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("1");
    }

    public synchronized void b(){
        log.debug("2");
    }
    
}

public static void test04(){
        Number n1 = new Number();
        Number n2 = new Number();
        new Thread(()->{ log.debug("begin..."); n1.a();}).start();
        new Thread(()->{ log.debug("begin..."); n2.b();}).start();

    }
```

- 情况5：输出结果为`2 1s后 1`

```java
@Slf4j(topic = "c.Number")
class Number{
    public static synchronized void a() {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("1");
    }

    public synchronized void b(){
        log.debug("2");
    }
}

public static void test05(){
        Number n1 = new Number();
        new Thread(()->{ log.debug("begin..."); n1.a();}).start();
        new Thread(()->{ log.debug("begin..."); n1.b();}).start();
    }
```

- 情况6：输出结果为`1s后 1 2`或`2 1s后 1`

```java
@Slf4j(topic = "c.Number")
class Number{
    public static synchronized void a() {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("1");
    }

    public static synchronized void b(){
        log.debug("2");
    }
}

```

- 情况7：输出结果为`2 1s后 1`

```java
@Slf4j(topic = "c.Number")
class Number{
    public static synchronized void a() {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("1");
    }

    public synchronized void b(){
        log.debug("2");
    }
}

 public static void test07(){
        Number n1 = new Number();
        Number n2 = new Number();
        new Thread(()->{ log.debug("begin..."); n1.a();}).start();
        new Thread(()->{ log.debug("begin..."); n2.b();}).start();
    }
```

- 情况8：输出结果为`1s后 1 2`或`2 1s后 1`  

```java
@Slf4j(topic = "c.Number")
class Number{
    public static synchronized void a() {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("1");
    }

    public static synchronized void b(){
        log.debug("2");
    }
}

 public static void test08(){
        Number n1 = new Number();
        Number n2 = new Number();
        new Thread(()->{ log.debug("begin..."); n1.a();}).start();
        new Thread(()->{ log.debug("begin..."); n2.b();}).start();
    }
```

## 4. 变量的线程安全分析

### 4.1 成员变量和静态变量是否线程安全？

- 如果它们没有共享，则线程安全
- 如果它们被共享了，根据它们的状态是否能够改变，又分两种情况:
  &nbsp;&nbsp;&nbsp;&nbsp;1. 如果只有读操作，则线程安全
  &nbsp;&nbsp;&nbsp;&nbsp;2. 如果有读写操作，则这段代码是临界区，需要考虑线程安全

### 4.2 局部变量是否线程安全？

- 局部变量是线程安全的
- 但局部变量引用的对象则未必  
  &nbsp;&nbsp;&nbsp;&nbsp;1. 如果该对象没有逃离方法的作用访问，它是线程安全的  
  &nbsp;&nbsp;&nbsp;&nbsp;2. 如果该对象逃离方法的作用范围，需要考虑线程安全

### 4.3 局部变量线程安全分析

```java
public static void test1() {
        int i = 10;
        i++;
    }
```

**每个线程**调用 test1() 方法时**局部变量** i，**会在每个线程的栈帧内存中被创建多份，因此不存在共享**  
test1()编译之后产生的JVM字节码指令：  

```java
0 bipush 10
2 istore_0
3 iinc 0 by 1
6 return
```

如图  
![局部变量线程安全分析](局部变量线程安全分析.png)  

#### 局部变量引用的对象线程安全分析

##### 成员变量

```java
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;

/**
 *
 */
public class LocalVariableTest03 {

    private final static int THREAD_NUM = 2;

    private final static int LOOP_NUM = 500;

    public static void main(String[] args) {
        ThreadUnSafe threadUnSafe = new ThreadUnSafe();
        for (int i = 0; i < THREAD_NUM; i++) {
            new Thread(()->{
                threadUnSafe.operation(LOOP_NUM);
            },"t"+i).start();
        }
    }
}

@Slf4j(topic = "c.ThreadUnSafe")
class ThreadUnSafe{

    private List<String> list = new ArrayList<>();

    public void operation(int loopNum){
        for (int i = 0; i < loopNum; i++) {
            add();
            remove();
        }
    }

    private void add() {
        list.add("A");
    }

    private void remove() {
        list.remove(0);
    }

}
```

分析：  

- 无论哪个线程中的 add 引用的都是同一个对象中的 list 成员变量
- add与remove的分析相同

```java
new Thread(() -> {
    list.add("1");        // 时间1. 会让内部 size ++
    list.remove(0); // 时间3. 再次 remove size-- 出现角标越界
}, "t1").start();

new Thread(() -> {
    list.add("2");        // 时间1（并发发生）. 会让内部 size ++，但由于size的操作非原子性,  size 本该是2，但结果可能出现1
    list.remove(0); // 时间2. 第一次 remove 能成功, 这时 size 已经是0
}, "t2").start();
```

##### 局部变量

> 将list修改为局部变量

```java
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;

/**
 *
 * 局部变量引用之局部变量
 */
public class LocalVariableTest04 {


    private final static int THREAD_NUM = 2;

    private final static int LOOP_NUM = 500;

    public static void main(String[] args) {
        ThreadSafe threadSafe = new ThreadSafe();
        for (int i = 0; i < THREAD_NUM; i++) {
            new Thread(()->{

                threadSafe.operation(LOOP_NUM);
            },"t"+i).start();
        }
    }
}


@Slf4j(topic = "c.ThreadSafe")
class ThreadSafe{

    public void operation(int loopNum){
        List<String> list = new ArrayList<>();
        for (int i = 0; i < loopNum; i++) {
            add(list);
            remove(list);
        }
    }

    private void add(List<String> list) {
        list.add("A");
    }

    private void remove(List<String> list) {
        list.remove(0);
    }

}

```

这样就不会出现线程不安全的问题。  
分析：  

- `list` 是局部变量，每个线程调用时会创建其不同实例，没有共享
- 而 `add` 的参数是从 `operation` 中传递过来的，与 `operation` 中引用同一个对象
- `add` 的参数分析与 `remove` 相同  

##### 方法访问修饰符带来的线程安全问题

如果把 `add` 和 `remove` 的方法修改为 `public` 会不会代理线程安全问题？

- 情况1：将`add`和`remove`的方法全部修改为`public`,其他线程调`operation()`方法
- 在 `情况1` 的基础上，为 `ThreadSafe01` 类添加子类，子类覆盖 `add` 或 `remove` 方法，即

```java
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;

/**
 * 局部变量引用之暴漏引用
 */
public class LocalVariableTest05 {

    private final static int THREAD_NUM = 2;

    private final static int LOOP_NUM = 500;

    public static void main(String[] args) {
        ThreadSafe01 threadSafe = new ThreadSafeSub();
        for (int i = 0; i < THREAD_NUM; i++) {
            new Thread(()->{

                threadSafe.operation(LOOP_NUM);
            },"t"+i).start();
        }
    }
}
@Slf4j(topic = "c.ThreadSafe01")
class ThreadSafe01{

    public void operation(int loopNum){
        List<String> list = new ArrayList<>();
        for (int i = 0; i < loopNum; i++) {
            add(list);
            remove(list);
        }
    }

    public void add(List<String> list) {
        list.add("A");
    }

    public void remove(List<String> list) {
        list.remove(0);
    }

}
class ThreadSafeSub extends ThreadSafe01{
    @Override
    public void remove(List<String> list) {
        new Thread(() -> {
            list.remove(0);
        }).start();

    }
}

```

方法改进：  

```java
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;

/**
 * 方法修饰符改进
 */
public class LocalVariableTest06 {
    private final static int THREAD_NUM = 2;

    private final static int LOOP_NUM = 500;

    public static void main(String[] args) {
        ThreadSafe02 threadSafe = new ThreadSafe02();
        for (int i = 0; i < THREAD_NUM; i++) {
            new Thread(()->{

                threadSafe.operation(LOOP_NUM);
            },"t"+i).start();
        }
    }
}

@Slf4j(topic = "c.ThreadSafe02")
class ThreadSafe02{

    public final void operation(int loopNum){
        List<String> list = new ArrayList<>();
        for (int i = 0; i < loopNum; i++) {
            add(list);
            remove(list);
        }
    }

    private void add(List<String> list) {
        list.add("A");
    }

    private void remove(List<String> list) {
        list.remove(0);
    }

}


```

> **注意：**
> 从这个例子可以看出 private 或 final 提供**安全**的意义所在，请体会开闭原则中的**闭**

### 4.4 常见线程安全类  

- String
- Integer
- StringBuffer
- Random
- Vector
- HashTable
- java.util.concurrent包下的类

这里说它们是线程安全的是指，多个线程调用它们同一个实例的某个方法时，是线程安全的。也可以理解为

```java
import lombok.extern.slf4j.Slf4j;

import java.util.Hashtable;

/**
 * 常见线程安全类举例
 */
@Slf4j(topic = "c.HashTableTest")
public class HashTableTest {
    public static void main(String[] args) throws InterruptedException {
        Hashtable<Integer,Integer> hashtable = new Hashtable<>();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 50000; i++) {
                hashtable.put(new Integer(i), i);
            }
        }, "t1");
        Thread t2 = new Thread(() -> {
            for (int i = 50000; i < 100000; i++) {
                hashtable.put(new Integer(i), i);
            }
        }, "t2");

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        log.debug("hashtable size is {}",hashtable.size());

    }
}

```

- 它们的每个方法是原子的  
- 但注意它们多个方法的组合不是原子的(见后面分析),如下：

#### 线程安全类方法的组合

```java
import lombok.extern.slf4j.Slf4j;

import java.util.Hashtable;

/**
 * 线程安全类的原子方法的组合不是原子的
 */
@Slf4j(topic = "c.HashTableTest01")
public class HashTableTest01 {

    public static void main(String[] args) throws InterruptedException {

        Hashtable<Integer,Integer> hashtable = new Hashtable<>();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                operate(i,hashtable);
            }
        }, "t1");
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                operate(i,hashtable);
            }
        }, "t2");

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        log.debug("hashtable size = {} ",hashtable.size());

    }

    private static void operate(int i,Hashtable<Integer,Integer> hashtable){
        synchronized (HashTableTest01.class){
            if (hashtable.get(i) == null) {
                hashtable.put(i, i);
            } else{
                hashtable.remove(i);
            }
        }

    }
}

```

#### 不可变类线程的安全性

String、Integer 等都是不可变类，因为其内部的状态不可以改变，因此它们的方法都是线程安全的

> 有同学或许有疑问，`String` 有 `replace`，`substring` 等方法**可以**改变值啊，那么这些方法又是如何保证线程安全的呢？
> 以`String`的`substring`方法为例

```java
 public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = value.length - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
    }
```

源码中存在一些方法，调用后可以得到改变后的值（`replace，replaceAll，toLowerCase`等）；通过源码可以知道，这些值表面上改变了，实际上方法内部创建了一个新的`String`对象并把这些对象重新赋值给当前的引用。

### 4.5 线程安全案例分析

- 案例1

```java
public class MyServlet extends HttpServlet {
 // 是否安全？ 不安全
 Map<String,Object> map = new HashMap<>();
 // 是否安全？ 不可变类安全
 String S1 = "...";
 // 是否安全？ 安全
 final String S2 = "...";
 // 是否安全？ 不安全
 Date D1 = new Date();
 // 是否安全？ 不安全 属性可以修改
 final Date D2 = new Date();
 
 public void doGet(HttpServletRequest request, HttpServletResponse response) {

 // 使用上述变量
 }
}

```

- 案例2

```java
public class MyServlet extends HttpServlet {
 // 是否安全？ 不安全 
 private UserService userService = new UserServiceImpl();
 
 public void doGet(HttpServletRequest request, HttpServletResponse response) {
 userService.update(...);
 }
}
public class UserServiceImpl implements UserService {
 // 记录调用次数
 private int count = 0;
 
 public void update() {
 // ...
 count++;
 }
}

```

- 案例3

```java
@Aspect
@Component
public class MyAspect {
 // 是否安全？不安全 单例被共享
 private long start = 0L;
 
 @Before("execution(* *(..))")
 public void before() {
 start = System.nanoTime();
 }
 
 @After("execution(* *(..))")
 public void after() {
 long end = System.nanoTime();
 System.out.println("cost time:" + (end-start));
 }
}
```

- 案例4

```java
public class MyServlet extends HttpServlet {
 // 是否安全  安全
 private UserService userService = new UserServiceImpl();
 
 public void doGet(HttpServletRequest request, HttpServletResponse response) {
 userService.update(...);
 }
}
public class UserServiceImpl implements UserService {
 // 是否安全  安全 没有成员变量
 private UserDao userDao = new UserDaoImpl();
 
 public void update() {
 userDao.update();
 }
}
public class UserDaoImpl implements UserDao { 
 public void update() {
 String sql = "update user set password = ? where username = ?";
 // 是否安全  安全 局部变量
 try (Connection conn = DriverManager.getConnection("","","")){
 // ...
 } catch (Exception e) {
 // ...
 }
 }
}

```

- 案例5

```java
public class MyServlet extends HttpServlet {
 // 是否安全 安全
 private UserService userService = new UserServiceImpl();
 
 public void doGet(HttpServletRequest request, HttpServletResponse response) {
 userService.update(...);
 }
}
public class UserServiceImpl implements UserService {
 // 是否安全 安全
 private UserDao userDao = new UserDaoImpl();
 
 public void update() {
 userDao.update();
 }
}
public class UserDaoImpl implements UserDao {
 // 是否安全 不安全
 private Connection conn = null;
 public void update() throws SQLException {
 String sql = "update user set password = ? where username = ?";
 conn = DriverManager.getConnection("","","");
 // ...
 conn.close();
 }
}

```

- 案例6

```java
public class MyServlet extends HttpServlet {
 // 是否安全 安全
 private UserService userService = new UserServiceImpl();
 
 public void doGet(HttpServletRequest request, HttpServletResponse response) {
 userService.update(...);
 }
}
public class UserServiceImpl implements UserService { 
 public void update() {
 UserDao userDao = new UserDaoImpl();
 userDao.update();
 }
}
public class UserDaoImpl implements UserDao {
 // 是否安全 安全
 private Connection = null;
 public void update() throws SQLException {
 String sql = "update user set password = ? where username = ?";
 conn = DriverManager.getConnection("","","");
 // ...
 conn.close();
 }
}
```

- 案例7

```java
public abstract class Test {
 
 public void bar() {
 // 是否安全 不安全
 SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
 foo(sdf);
  }
 
 public abstract foo(SimpleDateFormat sdf);
 
 
 public static void main(String[] args) {
 new Test().bar();
 }
}

```

其中 foo 的行为是不确定的，可能导致不安全的发生，被称之为外星方法

```java
public void foo(SimpleDateFormat sdf) {
 String dateStr = "1999-10-11 00:00:00";
 for (int i = 0; i < 20; i++) {
 new Thread(() -> {
 try {
 sdf.parse(dateStr);
 } catch (ParseException e) {
 e.printStackTrace();
 }
 }).start();
 }
}
```

请比较 JDK 中 String 类的实现

- 案例8

```java
private static Integer i = 0;
public static void main(String[] args) throws InterruptedException {
 List<Thread> list = new ArrayList<>();
 for (int j = 0; j < 2; j++) {
 Thread thread = new Thread(() -> {
 for (int k = 0; k < 5000; k++) {
 synchronized (i) {
 i++;
 }
 }
 }, "" + j);
 list.add(thread);
 }
 list.stream().forEach(t -> t.start());
 list.stream().forEach(t -> {
 try {
 t.join();
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 });
 log.debug("{}", i);
}
```

## 5. 习题

### 卖票练习

```java
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.Vector;

@Slf4j(topic = "c.ExerciseSell")
public class ExerciseSell {
    public static void main(String[] args) throws InterruptedException {
        // 模拟多人买票
        // 1. 创建售票窗口
        TicketWindow ticketWindow = new TicketWindow(1000);
        // 线程集合
        List<Thread> threadList = new ArrayList<>();
        // 记录卖出的票数
        List<Integer> amountList = new Vector<>();
        // 2. 多人去买票
        for (int i = 0; i < 2000; i++) {
            Thread thread = new Thread(() -> {
                // 买票
                int amount = ticketWindow.sell(randomAmount());
                try {
                    Thread.sleep(randomAmount());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                amountList.add(amount);
            }, "买家" + i);
            // 线程启动添加到线程集合
            threadList.add(thread);
            thread.start();
        }

        // 等待所有线程执行完毕
        for (Thread thread: threadList) {
            thread.join();
        }

        // 统计卖出的票和剩余的票
        log.debug("卖票：{} 张",amountList.stream().mapToInt(i->i).sum());
        log.debug("余票：{} 张",ticketWindow.getCount());
    }

    // Random为线程安全
    static Random random = new Random();
    // 随机1——5
    public static int randomAmount(){
        return random.nextInt(5) + 1;
    }
}

class TicketWindow{

    private int count;

    public TicketWindow(int count){
        this.count = count;
    }

    // 获取余票数量
    public int getCount(){
        return count;
    }

    // 售票
    public synchronized int sell(int amount){
        if(this.count >= amount){
            this.count -= amount;
            return amount;
        }else {
            return 0;
        }
    }
}

```

### 转账练习

```java
import lombok.extern.slf4j.Slf4j;

import java.util.Random;

@Slf4j(topic = "c.ExerciseTransfer")
public class ExerciseTransfer {
    public static void main(String[] args) throws InterruptedException {
        // 创建账户
        Account account1 = new Account(1000);
        Account account2 = new Account(1000);

        // 转账
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                account1.transfer(account2, randomAmount());
            }
        }, "account1");

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                account2.transfer(account1, randomAmount());
            }
        }, "account2");

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        // 查看转账2000次后的账户总金额
        log.debug("total = {}",account1.getMoney() + account2.getMoney());
    }

    // Random为线程安全
    static Random random = new Random();
    // 随机1——100
    public static int randomAmount(){
        return random.nextInt(100) + 1;
    }
}

// 账户
class Account{

    // 账户余额
    private int money;

    public Account(int money){
        this.money = money;
    }

    public int getMoney() {
        return money;
    }

    public void setMoney(int money) {
        this.money = money;
    }

    // 转账
    public void transfer(Account target,int amount){
        synchronized(Account.class){ // 此处不可使用this,因为涉及两个账户的成员变量money的加锁，使用this只是对当前对象加锁
            if(this.money >= amount){
                this.setMoney(this.getMoney() - amount);
                target.setMoney(target.getMoney() + amount);
            }
        }

    }
}

```

## 6. Monitor 概念

### 6.1 Java对象头

> 此部分基于`JDK8`进行描述的。   

对象在内存中的存储布局分为 3 块区域：**对象头（Header）**、**实例数据（Instance Data）**和**对齐填充（Padding）**。
使用[jol](https://openjdk.java.net/projects/code-tools/jol/)获取对象布局的总体结构

```java
import org.openjdk.jol.info.ClassLayout;

public class ObjectHeaderTest {
    public static void main(String[] args) {
        L l = new L();
        System.out.println(ClassLayout.parseInstance(l).toPrintable());

    }
}
class L{
    private boolean flag = true;

    private Integer count = 1;
}
```

结果如下：

```cmd
com.java.demo.monitor.L object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                               VALUE
      0     4           (object header)   //markword              01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4           (object header)   //markword              00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4           (object header)  //klass pointer 类元数据  43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4       int L.count                                   1
     16     1   boolean L.flag                                    true  // Instance Data 对象实际的数据
     17     7           (loss due to the next object alignment)         //Padding 对齐填充数据
Instance size: 24 bytes
Space losses: 0 bytes internal + 7 bytes external = 7 bytes total
```

- `OFFSET` 偏移地址，单位字节； 
- `SIZE` 占用的内存大小，单位字节； 
- `TYPE DESCRIPTION` 类型描述，其中`object header`为对象头；
- `VALUE` 对应内存中当前存储的值；

根据对象头的格式，前8个字节为mark word（在64位JVM下）。
由于希望用尽可能少的二进制位表示尽可能多的信息，所以设置了lock标记。该标记的值不同，整个mark word表示的含义不同。  
通过倒数三位数 我们可以判断出锁的类型

```c++
enum {  locked_value                 = 0, // 0 00 轻量级锁
         unlocked_value           = 1,// 0 01 无锁
         monitor_value            = 2,// 0 10 重量级锁
         marked_value             = 3,// 0 11 gc标志
         biased_lock_pattern      = 5 // 1 01 偏向锁
  }; 
```

HotSpot采用[Oop-Klass](https://www.zhihu.com/search?hybrid_search_extra=%7B%22sourceType%22%3A%22article%22%2C%22sourceId%22%3A%22104494807%22%7D&hybrid_search_source=Entity&q=Oop-Klass%E6%A8%A1%E5%9E%8B&search_source=Entity&type=content)模型来表示Java对象，其中Klass对应着Java对象的类型（Class），而Oop则对应着Java对象的实例（Instance）。

```C++
// hotspot/src/share/vm/oops/oop.hpp
class oopDesc {
 ...
 private:
  // 用于存储对象的运行时记录信息，如哈希值、GC分代年龄、锁状态等
  volatile markOop  _mark;
  // Klass指针的联合体，指向当前对象所属的Klass对象
  union _metadata {
    // 未采用指针压缩技术时使用
    Klass*      _klass;
    // 采用指针压缩技术时使用
    narrowKlass _compressed_klass;
  } _metadata;
 ...
}
```

- [查看源码](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/oop.hpp)  

整个对象头由两个部分组成，即：`klass pointer`和`mark word`（`_mark`和`_metadata`被称为**对象头**）。其中前者存储对象的运行时记录信息；后者是一个指针，指向当前对象所属的Klass对象。  

- 普通对象  
  ![普通对象头](普通对象头.png)  
- 数组对象  
  ![数组对象](数组对象.png)

#### klass pointer

- `klass pointer`一般占`32bit`即`4`个字节，如果你有足够的原因关闭默认的指针压缩，即启动参数加上了`-XX:-UseCompressedOops`那么它就`64bit`。  
- `klass pointer`的存储内容是一个指针，指向了其类元数据的信息，`jvm`使用该指针来确定此对象是类的哪个实例.  

![klass poniter](klassponiter.png)

#### mark word

`mark word`主要用来存储对象自身的运行时数据，如`hashcode`、`gc`分代年龄等。`mark word`的位长度为`JVM`的一个`Word`大小，也就是说`32`位`JVM`的`Mark word`为`32`位，`64`位`JVM`为`64`位。  
![32bit mark word](32bit_mark_word.png)  
![64bit_mark_word](64bit_mark_word.png) 

- `unsed`  未使用的
- `hashcode`  指`identity hashcode`。`identityHashCode`是`System`里面提供的本地方法，`identityHashCode`会返回对象的`hashCode`，而不管对象是否重写了`hashCode`方法。
- `thread`  偏向锁记录的线程标识  
- `epoch`  验证偏向锁有效性的时间戳
- `age`  分代年龄
- `biased_lock`  偏向锁标志
- `lock`  锁标志
- `pointer_to_lock_record`  轻量锁`lock record`指针  
- `pointer_to_heavyweight_monitor`  重量锁`monitor`指针


### 6.2 原理 之 Monitor(锁)

`Monitor` 被翻译为**监视器**或**管程**  
每个 `Java` 对象都可以关联一个 `Monitor` 对象，如果使用 `synchronized` 给对象上锁（重量级）之后，该对象头的
`Mark Word` 中就被设置指向 `Monitor` 对象的指针。  
`Monitor` 结构如下  
![Monitor](Monitor.png)  

- 刚开始 `Monitor` 中 `Owner` 为 `null`  
- 当 `Thread-2` 执行 `synchronized(obj)` 就会将 `Monitor` 的所有者 `Owner` 置为 `Thread-2`，`Monitor`中只能有一
  个 `Owner`  
- 在 `Thread-2` 上锁的过程中，如果 `Thread-3`，`Thread-4`，`Thread-5` 也来执行 `synchronized(obj)`，就会进入
  `EntryList BLOCKED`  
- `Thread-2` 执行完同步代码块的内容，然后唤醒 `EntryList` 中等待的线程来竞争锁，竞争的时是非公平的  
- 图中 `WaitSet` 中的 `Thread-0`，`Thread-1` 是之前获得过锁，但条件不满足进入 `WAITING` 状态的线程，后面讲
  `wait-notify` 时会分析  

> **注意**
>
> - synchronized 必须是进入同一个对象的 monitor 才有上述的效果  
> - 不加 synchronized 的对象不会关联监视器，不遵从以上规则

### 6.3 原理 之 synchronized

```java
static final Object lock = new Object();
    static int counter = 0;
    public static void main(String[] args) {
        synchronized (lock) {
            counter++;
        }
    }
```

对应字节码  

```java
public static void main(java.lang.String[]);
 descriptor: ([Ljava/lang/String;)V
 flags: ACC_PUBLIC, ACC_STATIC
 Code:
 stack=2, locals=3, args_size=1
 0: getstatic #2 // <- lock引用 （synchronized开始）
 3: dup
 4: astore_1 // lock引用 -> slot 1
 5: monitorenter // 将 lock对象 MarkWord 置为 Monitor 指针
 6: getstatic #3 // <- i
 9: iconst_1 // 准备常数 1
 10: iadd // +1
 11: putstatic #3 // -> i
 14: aload_1 // <- lock引用
 15: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList
 16: goto 24
 19: astore_2 // e -> slot 2 
 20: aload_1 // <- lock引用
 21: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList
 22: aload_2 // <- slot 2 (e)
 23: athrow // throw e
 24: return
 Exception table:
 from to target type
 6 16 19 any
 19 22 19 any
 LineNumberTable:
 line 8: 0
 line 9: 6
 line 10: 14
 line 11: 24
 LocalVariableTable:
 Start Length Slot Name Signature
 0 25 0 args [Ljava/lang/String;
 StackMapTable: number_of_entries = 2
 frame_type = 255 /* full_frame */
 offset_delta = 19
 locals = [ class "[Ljava/lang/String;", class java/lang/Object ]
 stack = [ class java/lang/Throwable ]
 frame_type = 250 /* chop */
 offset_delta = 4
```

> 注意  
> 方法级别的 synchronized 不会在字节码指令中有所体现

### 6.4 原理 之 synchronized进阶

#### 6.4.1 轻量级锁

轻量级锁的使用场景  

> 如果一个对象虽然有多线程要加锁，但加锁的时间是错开的（也就是没有竞争），那么可以
> 使用轻量级锁来优化。  

**轻量级锁**对使用者是透明的，即语法仍然是 `synchronized`。  
假设有两个方法同步块，利用同一个对象加锁

```java
static final Object obj = new Object();
    public static void method1() {
        synchronized( obj ) {
            // 同步块 A
            method2();
        }
    }
    public static void method2() {
        synchronized( obj ) {
            // 同步块 B
        }
    }
```

![轻量级锁](轻量级锁.png)

#### 6.4.2 锁膨胀

如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁（有
竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。  

```java
static Object obj = new Object();
public static void method1() {
 synchronized( obj ) {
 // 同步块
 }
}

```

![锁膨胀](锁膨胀.png)

#### 6.4.3 自旋优化

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步
块，释放了锁），这时当前线程就可以避免阻塞。

- 自旋重试成功的情况

| 线程1(core 1上)         | 对象Mark               | 线程2(core 2上)         |
| ----------------------- | ---------------------- | ----------------------- |
| -                       | 10（重量锁）           | -                       |
| 访问同步块，获取monitor | 10（重量锁）重量锁指针 | -                       |
| 成功（加锁）            | 10（重量锁）重量锁指针 | -                       |
| 执行同步块              | 10（重量锁）重量锁指针 | -                       |
| 执行同步块              | 10（重量锁）重量锁指针 | 访问同步块，获取monitor |
| 执行同步块              | 10（重量锁）重量锁指针 | 自旋重试                |
| 执行完毕                | 10（重量锁）重量锁指针 | 自旋重试                |
| 成功（解锁）            | 10（无锁）             | 自旋重试                |
| -                       | 10（重量锁）重量锁指针 | 成功（加锁）            |
| -                       | 10（重量锁）重量锁指针 | 执行同步块              |
| -                       | ...                    | ...                     |


- 自旋重试失败的情况

| 线程1(core 1上)         | 对象Mark               | 线程2(core 2上)         |
| ----------------------- | ---------------------- | ----------------------- |
| -                       | 10（重量锁）           | -                       |
| 访问同步块，获取monitor | 10（重量锁）重量锁指针 | -                       |
| 成功（加锁）            | 10（重量锁）重量锁指针 | -                       |
| 执行同步块              | 10（重量锁）重量锁指针 | -                       |
| 执行同步块              | 10（重量锁）重量锁指针 | 访问同步块，获取monitor |
| 执行同步块              | 10（重量锁）重量锁指针 | 自旋重试                |
| 执行同步块              | 10（重量锁）重量锁指针 | 自旋重试                |
| 执行同步块              | 10（重量锁）重量锁指针 | 自旋重试                |
| 执行同步块              | 10（重量锁）重量锁指针 | 阻塞                    |
| -                       | ...                    | ...                     |


**注意：**

> - 自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。  
> - 在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，总之，比较智能。  
> - Java 7 之后不能控制是否开启自旋功能

#### 6.4.4 偏向锁

轻量级锁在没有竞争时（就自己这个线程），每次重入仍然需要执行 CAS 操作。
Java 6 中引入了**偏向锁**来做进一步优化：只有第一次使用 CAS 将线程 ID 设置到对象的 Mark Word 头，之后发现这个线程 ID 是自己的就表示没有竞争，不用重新 CAS。以后只要不发生竞争，这个对象就归该线程所有。
例如：

```java
static final Object obj = new Object();
public static void m1() {
 synchronized( obj ) {
 // 同步块 A
 m2();
 }
}
public static void m2() {
 synchronized( obj ) {
 // 同步块 B
 m3();
 }
}
public static void m3() {
 synchronized( obj ) {
    // 同步块 C
 }
}
  
```

![偏向锁](偏向锁1.png) 

![偏向锁](偏向锁2.png)  

##### 偏向状态

对象头格式  
![64bit](64bit_mark_word.png)  
一个对象创建时：

- 如果开启了偏向锁（默认开启），那么对象创建后，`markword` 值为 `0x05` 即最后 `3` 位为 `101`，这时它的`thread`、`epoch`、`age` 都为 `0`   
- 偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加 `VM` 参数 `-XX:BiasedLockingStartupDelay=0` 来禁用延迟  
- 如果没有开启偏向锁，那么对象创建后，`markword` 值为 `0x01` 即最后 `3` 位为 `001`，这时它的 `hashcode`、`age` 都为 `0`，第一次用到 `hashcode` 时才会赋值

###### 测试延迟特性

测试代码运行时在添加 VM 参数`-XX:BiasedLockingStartupDelay=0` 禁用延迟

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

import java.util.concurrent.TimeUnit;

/**
 * 偏向锁-测试延迟性
 */
@Slf4j(topic="c.BiasedTest01")
public class BiasedTest01 {
    public static void main(String[] args) throws InterruptedException {
        System.out.println(ClassLayout.parseInstance(new Cat()).toPrintable());
        TimeUnit.SECONDS.sleep(5);
        System.out.println(ClassLayout.parseInstance(new Cat()).toPrintable());
    }
}
class Cat{

}

```

结果如下

```cmd
com.java.demo.theory.Cat object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

com.java.demo.theory.Cat object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

###### 测试偏向锁

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

@Slf4j(topic = "c.BiasedTest02")
public class BiasedTest02 {
    public static void main(String[] args) {
        Monkey monkey = new Monkey();
        log.debug(ClassLayout.parseInstance(monkey).toPrintable());
        synchronized (monkey){
            log.debug(ClassLayout.parseInstance(monkey).toPrintable());
        }
        log.debug(ClassLayout.parseInstance(monkey).toPrintable());
    }
}
class Monkey{

}
```

结果如下

```cmd
10:51:40 [main] c.BiasedTest02 - com.java.demo.theory.Monkey object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

10:51:40 [main] c.BiasedTest02 - com.java.demo.theory.Monkey object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 28 39 03 (00000101 00101000 00111001 00000011) (54077445)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

10:51:40 [main] c.BiasedTest02 - com.java.demo.theory.Monkey object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 28 39 03 (00000101 00101000 00111001 00000011) (54077445)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

**注意**  

> 处于偏向锁的对象解锁后，线程 id 仍存储于对象头中

###### 禁用偏向锁

在上面测试代码运行时在添加 VM 参数 `-XX:-UseBiasedLocking` 禁用偏向锁  
输出结果如下：

```cmd
10:56:07 [main] c.BiasedTest02 - com.java.demo.theory.Monkey object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

10:56:07 [main] c.BiasedTest02 - com.java.demo.theory.Monkey object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           18 f6 c4 02 (00011000 11110110 11000100 00000010) (46462488)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

10:56:07 [main] c.BiasedTest02 - com.java.demo.theory.Monkey object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

###### 测试 hashCode

正常状态对象一开始是没有 hashCode 的，第一次调用才生成

##### 撤销 - 调用对象 hashCode

调用了对象的 `hashCode`，但偏向锁的对象 `MarkWord` 中存储的是线程 `id`，如果调用 `hashCode` 会导致偏向锁被撤销  

- **轻量级锁**会在锁记录中记录 hashCode
- **重量级锁**会在 Monitor 中记录 hashCode
  在调用 `hashCode` 后使用偏向锁，记得去掉`-XX:-UseBiasedLocking`,设置为`-XX:BiasedLockingStartupDelay=0` 

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

@Slf4j(topic = "c.BiasedTest03")
public class BiasedTest03 {
    public static void main(String[] args) {
        Rabbit rabbit = new Rabbit();
        rabbit.hashCode(); // 调用hashcode禁用偏向锁
        log.debug(ClassLayout.parseInstance(rabbit).toPrintable());
        synchronized (rabbit){
            log.debug(ClassLayout.parseInstance(rabbit).toPrintable());
        }
        log.debug(ClassLayout.parseInstance(rabbit).toPrintable());
    }
}
class Rabbit{

}
```

输出结果为

```cmd
11:14:09 [main] c.BiasedTest03 - com.java.demo.theory.Rabbit object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 3f aa 4c (00000001 00111111 10101010 01001100) (1286225665)
      4     4        (object header)                           57 00 00 00 (01010111 00000000 00000000 00000000) (87)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:14:09 [main] c.BiasedTest03 - com.java.demo.theory.Rabbit object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           c8 f6 da 02 (11001000 11110110 11011010 00000010) (47904456)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:14:09 [main] c.BiasedTest03 - com.java.demo.theory.Rabbit object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 3f aa 4c (00000001 00111111 10101010 01001100) (1286225665)
      4     4        (object header)                           57 00 00 00 (01010111 00000000 00000000 00000000) (87)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

##### 撤销 - 其它线程使用对象

当有其它线程使用偏向锁对象时，会将偏向锁升级为轻量级锁    
测试代码运行时在添加 VM 参数`-XX:BiasedLockingStartupDelay=0` 禁用延迟

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

@Slf4j(topic = "c.BiasedTest04")
public class BiasedTest04 {
    public static void main(String[] args) {
        Test test = new Test();
        Thread t1 = new Thread(() -> {
            log.debug(ClassLayout.parseInstance(test).toPrintable());
            synchronized (test) {
                log.debug(ClassLayout.parseInstance(test).toPrintable());
            }
            log.debug(ClassLayout.parseInstance(test).toPrintable());

            synchronized (BiasedTest04.class){
                BiasedTest04.class.notify();
            }

        }, "t1");

        t1.start();

        Thread t2 = new Thread(() -> {

            synchronized (BiasedTest04.class){
                try {
                    BiasedTest04.class.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            log.debug(ClassLayout.parseInstance(test).toPrintable());
            synchronized (test) {
                log.debug(ClassLayout.parseInstance(test).toPrintable());
            }
            log.debug(ClassLayout.parseInstance(test).toPrintable());
        }, "t2");

        t2.start();
    }
}
class Test{

}

```

执行结果

```cmd
11:11:49 [t1] c.BiasedTest04 - com.java.demo.theory.Test object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:11:49 [t1] c.BiasedTest04 - com.java.demo.theory.Test object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 50 d3 1f (00000101 01010000 11010011 00011111) (533942277)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:11:49 [t1] c.BiasedTest04 - com.java.demo.theory.Test object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 50 d3 1f (00000101 01010000 11010011 00011111) (533942277)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:11:49 [t2] c.BiasedTest04 - com.java.demo.theory.Test object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 50 d3 1f (00000101 01010000 11010011 00011111) (533942277)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:11:49 [t2] c.BiasedTest04 - com.java.demo.theory.Test object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           30 ef 2a 20 (00110000 11101111 00101010 00100000) (539684656)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:11:49 [t2] c.BiasedTest04 - com.java.demo.theory.Test object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

##### 撤销 - 调用 wait/notify

测试代码运行时在添加 VM 参数`-XX:BiasedLockingStartupDelay=0` 禁用延迟

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

@Slf4j(topic = "c.BiasedTest05")
public class BiasedTest05 {
    public static void main(String[] args) {
        Dog dog = new Dog();
        new Thread(()->{
            log.debug(ClassLayout.parseInstance(dog).toPrintable());
            synchronized (dog){
                log.debug(ClassLayout.parseInstance(dog).toPrintable());
                try {
                    dog.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug(ClassLayout.parseInstance(dog).toPrintable());
            }
            log.debug(ClassLayout.parseInstance(dog).toPrintable());
        },"t1").start();


        new Thread(()->{
            try {
                Thread.sleep(6000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (dog) {
                log.debug("notify");
                dog.notify();
            }
        },"t2").start();
    }
}


```

运行结果

```cmd
11:47:23 [t1] c.BiasedTest05 - com.java.demo.theory.Dog object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:47:23 [t1] c.BiasedTest05 - com.java.demo.theory.Dog object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 38 c0 1f (00000101 00111000 11000000 00011111) (532690949)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:47:27 [t2] c.BiasedTest05 - notify
11:47:27 [t1] c.BiasedTest05 - com.java.demo.theory.Dog object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           4a 0e b7 1c (01001010 00001110 10110111 00011100) (481758794)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

11:47:27 [t1] c.BiasedTest05 - com.java.demo.theory.Dog object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           4a 0e b7 1c (01001010 00001110 10110111 00011100) (481758794)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a4 8b 01 f8 (10100100 10001011 00000001 11111000) (-134116444)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

```

##### 批量重偏向

如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程 `T1` 的对象仍有机会重新偏向 `T2`，重偏向会重置对象的 `Thread ID`  
当撤销偏向锁阈值超过 `20` 次后，jvm 会这样觉得，我是不是偏向错了呢，于是会在给这些对象加锁时重新偏向至加锁线程  
测试代码运行时在添加 VM 参数`-XX:BiasedLockingStartupDelay=0` 禁用延迟

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

import java.util.Vector;

/**
 * 批量重偏向
 */
@Slf4j(topic = "c.BiasedTest06")
public class BiasedTest06 {
    public static void main(String[] args) {
        Vector<Dog> list = new Vector<>();
        new Thread(()->{
            for (int i = 0; i < 30; i++) {
                Dog dog = new Dog();
                list.add(dog);
                synchronized (dog){
                    log.debug(i + "/t" + ClassLayout.parseInstance(dog).toPrintable());
                }
            }
            synchronized (list){
                list.notify();
            }
        },"t1").start();


        new Thread(()->{
            synchronized (list){
                try {
                    list.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            log.debug("===============>>> ");

            for (int i = 0; i < 30; i++) {
                Dog d = list.get(i);
                log.debug(i + "/t" + ClassLayout.parseInstance(d).toPrintable());
                synchronized (d){
                    log.debug(i + "/t" + ClassLayout.parseInstance(d).toPrintable());
                }
                log.debug(i + "/t" + ClassLayout.parseInstance(d).toPrintable());


            }
        },"t2").start();


    }
}

```

##### 批量撤销

当撤销偏向锁阈值超过 `40` 次后，`jvm` 会这样觉得，自己确实偏向错了，根本就不该偏向。于是整个类的所有对象都会变为不可偏向的，新建的对象也是不可偏向的  

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

import java.util.Vector;
import java.util.concurrent.locks.LockSupport;

/**
 * 批量撤销
 */
@Slf4j(topic = "c.BiasedTest07")
public class BiasedTest07 {
    public static void main(String[] args) throws InterruptedException {
        test4();
    }
    static Thread t1,t2,t3;

    private static void test4() throws InterruptedException {
        Vector<Dog> list = new Vector<>();
        int loopNumber = 39;
        t1 = new Thread(() -> {
            for (int i = 0; i < loopNumber; i++) {
                Dog d = new Dog();
                list.add(d);
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                }
            }
            LockSupport.unpark(t2);
        }, "t1");
        t1.start();
        t2 = new Thread(() -> {
            LockSupport.park();
            log.debug("===============> ");
            for (int i = 0; i < loopNumber; i++) {
                Dog d = list.get(i);
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                }
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
            }
            LockSupport.unpark(t3);
        }, "t2");
        t2.start();
        t3 = new Thread(() -> {
            LockSupport.park();
            log.debug("===============> ");
            for (int i = 0; i < loopNumber; i++) {
                Dog d = list.get(i);
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
                }
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintable());
            }
        }, "t3");
        t3.start();
        t3.join();
        log.debug(ClassLayout.parseInstance(new Dog()).toPrintable());
    }

}
```

#### 6.4.5 锁消除

```java
package com.java.demo.theory;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.concurrent.TimeUnit;

@Fork(1)
@BenchmarkMode(Mode.AverageTime)
@Warmup(iterations=3)
@Measurement(iterations=5)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class EliminateLocks {
    public static void main(String[] args) throws RunnerException {
        Options options = new OptionsBuilder()
                .include(EliminateLocks.class.getSimpleName())
                .build();
        new Runner(options).run();
    }
    static int x = 0;
    @Benchmark
    public void a() throws Exception {
        x++;
    }
    @Benchmark
    public void b() throws Exception {
        Object o = new Object();
        synchronized (o) {
            x++;}
    }
}
```

- 测试结果

```
Benchmark         Mode  Cnt  Score   Error  Units
EliminateLocks.a  avgt    5  2.854 ± 0.862  ns/op
EliminateLocks.b  avgt    5  2.109 ± 0.272  ns/op
```

- 添加` -XX:-EliminateLocks`参数测试结果

```
Benchmark         Mode  Cnt   Score   Error  Units
EliminateLocks.a  avgt    5   2.788 ± 0.359  ns/op
EliminateLocks.b  avgt    5  27.844 ± 6.160  ns/op
```

## 7. wait notify

### 7.1 wait notify原理

![](wait notify原理.png)

- `Owner` 线程发现条件不满足，调用 `wait` 方法，即可进入 `WaitSet` 变为 `WAITING` 状态 
- `BLOCKED` 和 `WAITING` 的线程都处于阻塞状态，不占用 `CPU` 时间片 
- `BLOCKED` 线程会在 `Owner` 线程释放锁时唤醒 
- `WAITING` 线程会在 `Owner` 线程调用 `notify` 或 `notifyAll` 时唤醒，但唤醒后并不意味者立刻获得锁，仍需进入 `EntryList` 重新竞争

### 7.2 API介绍

- `obj.wait()` 让进入 `object` 监视器的线程到 `waitSet` 等待 
- `obj.notify()`在 `object` 上正在 `waitSet` 等待的线程中挑一个唤醒 
- `obj.notifyAll()` 让 `object` 上正在 `waitSet`等待的线程全部唤醒

它们都是线程之间进行协作的手段，都属于 Object 对象的方法。必须获得此对象的锁，才能调用这几个方法

```java
package com.java.demo.theory;


import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

/**
 * wait-notify/notifyAll测试
 */
@Slf4j(topic = "c.waitNotify")
public class WaitNotifyTest {
    final static Object lock = new Object();
    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            synchronized (lock){
                log.debug("执行....");
                try {
                    lock.wait(); // 让线程在obj上一直等待下去
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("其它代码....");
            }
        },"t1").start();

        new Thread(()->{
            synchronized (lock){
                log.debug("执行....");
                try {
                    lock.wait(); // 让线程在obj上一直等待下去
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("其它代码....");
            }
        },"t1").start();

        // 主线程两秒后执行
        TimeUnit.SECONDS.sleep(2);
        log.debug("唤醒 obj 上其它线程");
        synchronized (lock) {
            //lock.notify(); // 唤醒obj上一个线程
            lock.notifyAll(); // 唤醒obj上所有等待线程
        }
    }
}
```

- `notify`的一种结果

  ```
  22:14:38 [t1] c.waitNotify - 执行....
  22:14:38 [t1] c.waitNotify - 执行....
  22:14:40 [main] c.waitNotify - 唤醒 obj 上其它线程
  22:14:40 [t1] c.waitNotify - 其它代码....
  ```

- `notifyAll`的结果

  ```
  22:12:50 [t1] c.waitNotify - 执行....
  22:12:50 [t1] c.waitNotify - 执行....
  22:12:52 [main] c.waitNotify - 唤醒 obj 上其它线程
  22:12:52 [t1] c.waitNotify - 其它代码....
  22:12:52 [t1] c.waitNotify - 其它代码....
  ```

`wait()` 方法会释放对象的锁，进入 `WaitSet` 等待区，从而让其他线程就机会获取对象的锁。无限制等待，直到 `notify` 为止 ;

`wait(long n)` 有时限的等待, 到 `n` 毫秒后结束等待，或是被 `notify`。

## 8. wait notify的正确姿势

###  sleep(long n)和wait(long n)的区别

1. `sleep`是`Thread`的方法，而`wait`是`Object`的方法
2. `sleep`不需要强制和`synchronized`配合使用，但是`wait`需要和`synchronized`一起用
3. `sleep`在睡眠的同时，不会释放对象锁的，但`wait`在等待的时候会释放对象锁

4. 它们 状态 `TIMED_WAITING`

### Step 1（Sleep方式）

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

/**
 * 使用sleep方法解决
 */
@Slf4j(topic = "c.step1")
public class WaitNotifyCorrectPostureStep1 {

    static final Object room = new Object();
    static boolean hasCigarette = false;
    static boolean hasTakeout = false;

    public static void main(String[] args) throws InterruptedException {

        new Thread(() -> {
            synchronized (room) {
                log.debug("有烟没？[{}]", hasCigarette);
                if (!hasCigarette) {
                    log.debug("没烟，先歇会！");
                    try {
                        TimeUnit.SECONDS.sleep(2);
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                log.debug("有烟没？[{}]", hasCigarette);
                if (hasCigarette) {
                    log.debug("可以开始干活了");
                }
            }
        }, "小南").start();
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                synchronized (room) {
                    log.debug("可以开始干活了");
                }
            }, "其它人").start();

        }
        TimeUnit.SECONDS.sleep(1);
        new Thread(() -> {
            // 这里能不能加 synchronized (room)？
            hasCigarette = true;
            log.debug("烟到了噢！");
        }, "送烟的").start();


    }
}

```

执行结果

```
3:33:20 [小南] c.step1 - 有烟没？[false]
23:33:20 [小南] c.step1 - 没烟，先歇会！
23:33:21 [送烟的] c.step1 - 烟到了噢！
23:33:22 [小南] c.step1 - 有烟没？[true]
23:33:22 [小南] c.step1 - 可以开始干活了
23:33:22 [其它人] c.step1 - 可以开始干活了
23:33:22 [其它人] c.step1 - 可以开始干活了
23:33:22 [其它人] c.step1 - 可以开始干活了
23:33:22 [其它人] c.step1 - 可以开始干活了
23:33:22 [其它人] c.step1 - 可以开始干活了

```

- 其它干活的线程，都要一直阻塞，效率太低 
- 小南线程必须睡足 2s 后才能醒来，就算烟提前送到，也无法立刻醒来 
- 加了 synchronized (room) 后，就好比小南在里面反锁了门睡觉，烟根本没法送进门，main 没加 synchronized 就好像 main 线程是翻窗户进来的 
- 解决方法，使用 `wait - notify` 机制

### step 2（wait-notify机制）

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.step2")
public class WaitNotifyCorrectPostureStep2 {

    static final Object room = new Object();
    static boolean hasCigarette = false;
    static boolean hasTakeout = false;

    public static void main(String[] args) throws InterruptedException {

        new Thread(() -> {
            synchronized (room) {
                log.debug("有烟没？[{}]", hasCigarette);
                if (!hasCigarette) {
                    log.debug("没烟，先歇会！");
                    try {
                        room.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                log.debug("有烟没？[{}]", hasCigarette);
                if (hasCigarette) {
                    log.debug("可以开始干活了");
                }
            }
        }, "小南").start();
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                synchronized (room) {
                    log.debug("可以开始干活了");
                }
            }, "其它人").start();

        }
        TimeUnit.SECONDS.sleep(1);
        new Thread(() -> {
            synchronized (room){
                hasCigarette = true;
                log.debug("烟到了噢！");
                room.notify();

            }
        }, "送烟的").start();


    }
}
```

执行结果

```
11:06:34 [小南] c.step2 - 有烟没？[false]
11:06:34 [小南] c.step2 - 没烟，先歇会！
11:06:34 [其它人] c.step2 - 可以开始干活了
11:06:34 [其它人] c.step2 - 可以开始干活了
11:06:34 [其它人] c.step2 - 可以开始干活了
11:06:34 [其它人] c.step2 - 可以开始干活了
11:06:34 [其它人] c.step2 - 可以开始干活了
11:06:35 [送烟的] c.step2 - 烟到了噢！
11:06:35 [小南] c.step2 - 有烟没？[true]
11:06:35 [小南] c.step2 - 可以开始干活了
```

- 解决了其它干活的线程阻塞的问题 
- 但如果有其它线程也在等待条件呢？

### Step 3

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.step3")
public class WaitNotifyCorrectPostureStep3 {

    static final Object room = new Object();
    static boolean hasCigarette = false;
    static boolean hasTakeout = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            synchronized (room){
                log.debug("有烟没？[{}]", hasCigarette);
                if (!hasCigarette) {
                    log.debug("没烟，先歇会！");
                    try {
                        room.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("有烟没？[{}]", hasCigarette);
                if (hasCigarette) {
                    log.debug("可以开始干活了");
                } else {
                    log.debug("没干成活");
                }
            }
        },"小南").start();
        new Thread(()->{
            synchronized (room) {
                Thread thread = Thread.currentThread();
                log.debug("外卖送到没？[{}]", hasTakeout);
                if (!hasTakeout) {
                    log.debug("没外卖，先歇会！");
                    try {
                        room.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("外卖送到没？[{}]", hasTakeout);
                if (hasTakeout) {
                    log.debug("可以开始干活了");
                } else {
                    log.debug("没干成活...");
                }
            }
        },"小女").start();

        TimeUnit.SECONDS.sleep(1);

        new Thread(() -> {
            synchronized (room) {
                hasTakeout = true;
                log.debug("外卖到了噢！");
                room.notify();
            }
        }, "送外卖的").start();

    }
}
```

执行结果

```
11:08:46 [小女] c.step3 - 外卖送到没？[false]
11:08:46 [小女] c.step3 - 没外卖，先歇会！
11:08:46 [小南] c.step3 - 有烟没？[false]
11:08:46 [小南] c.step3 - 没烟，先歇会！
11:08:47 [送外卖的] c.step3 - 外卖到了噢！
11:08:47 [小女] c.step3 - 外卖送到没？[true]
11:08:47 [小女] c.step3 - 可以开始干活了
```

```
11:09:09 [小南] c.step3 - 有烟没？[false]
11:09:09 [小南] c.step3 - 没烟，先歇会！
11:09:09 [小女] c.step3 - 外卖送到没？[false]
11:09:09 [小女] c.step3 - 没外卖，先歇会！
11:09:10 [送外卖的] c.step3 - 外卖到了噢！
11:09:10 [小南] c.step3 - 有烟没？[false]
11:09:10 [小南] c.step3 - 没干成活
```

- `notify` 只能随机唤醒一个 `WaitSet` 中的线程，这时如果有其它线程也在等待，那么就可能唤醒不了正确的线 程，称之为**虚假唤醒**
- 解决方法，改为 `notifyAll`

### Step 4

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.step4")
public class WaitNotifyCorrectPostureStep4 {

    static final Object room = new Object();
    static boolean hasCigarette = false;
    static boolean hasTakeout = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            synchronized (room){
                log.debug("有烟没？[{}]", hasCigarette);
                if (!hasCigarette) {
                    log.debug("没烟，先歇会！");
                    try {
                        room.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("有烟没？[{}]", hasCigarette);
                if (hasCigarette) {
                    log.debug("可以开始干活了");
                } else {
                    log.debug("没干成活");
                }
            }
        },"小南").start();
        new Thread(()->{
            synchronized (room) {
                Thread thread = Thread.currentThread();
                log.debug("外卖送到没？[{}]", hasTakeout);
                if (!hasTakeout) {
                    log.debug("没外卖，先歇会！");
                    try {
                        room.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("外卖送到没？[{}]", hasTakeout);
                if (hasTakeout) {
                    log.debug("可以开始干活了");
                } else {
                    log.debug("没干成活...");
                }
            }
        },"小女").start();

        TimeUnit.SECONDS.sleep(1);

        new Thread(() -> {
            synchronized (room) {
                hasTakeout = true;
                log.debug("外卖到了噢！");
                room.notifyAll();
            }
        }, "送外卖的").start();
    }
}

```

执行结果

```
11:16:07 [小南] c.step4 - 有烟没？[false]
11:16:07 [小南] c.step4 - 没烟，先歇会！
11:16:07 [小女] c.step4 - 外卖送到没？[false]
11:16:07 [小女] c.step4 - 没外卖，先歇会！
11:16:08 [送外卖的] c.step4 - 外卖到了噢！
11:16:08 [小女] c.step4 - 外卖送到没？[true]
11:16:08 [小女] c.step4 - 可以开始干活了
11:16:08 [小南] c.step4 - 有烟没？[false]
11:16:08 [小南] c.step4 - 没干成活
```

- 用 notifyAll 仅解决某个线程的唤醒问题，但使用 if + wait 判断仅有一次机会，一旦条件不成立，就没有重新 判断的机会了 
- 解决方法，用 while + wait，当条件不成立，再次 wait

### Step 5

```java
package com.java.demo.theory;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.step5")
public class WaitNotifyCorrectPostureStep5 {

    static final Object room = new Object();
    static boolean hasCigarette = false;
    static boolean hasTakeout = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            synchronized (room){
                log.debug("有烟没？[{}]", hasCigarette);
                while (!hasCigarette) {
                    log.debug("没烟，先歇会！");
                    try {
                        room.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("有烟没？[{}]", hasCigarette);
                if (hasCigarette) {
                    log.debug("可以开始干活了");
                } else {
                    log.debug("没干成活");
                }
            }
        },"小南").start();
        new Thread(()->{
            synchronized (room) {
                Thread thread = Thread.currentThread();
                log.debug("外卖送到没？[{}]", hasTakeout);
                if (!hasTakeout) {
                    log.debug("没外卖，先歇会！");
                    try {
                        room.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                log.debug("外卖送到没？[{}]", hasTakeout);
                if (hasTakeout) {
                    log.debug("可以开始干活了");
                } else {
                    log.debug("没干成活...");
                }
            }
        },"小女").start();

        TimeUnit.SECONDS.sleep(1);

        new Thread(() -> {
            synchronized (room) {
                hasTakeout = true;
                log.debug("外卖到了噢！");
                room.notifyAll();
            }
        }, "送外卖的").start();
    }
}

```

### Wait-Notify/NotifyAll正确姿势

```java
synchronized(lock){
    while(条件判断){
        lock.wait();
    }
    // TODO
}

// 另一个线程
synchronized(lock){
    lock.notifyAll();
}
```

### 同步模式 之 保护性暂停（Guarded Suspension）

#### 1. 定义

即 `Guarded Suspension`，用在一个线程等待另一个线程的执行结果 

要点 

- 有一个结果需要从一个线程传递到另一个线程，让他们关联同一个 `GuardedObject` 
- 如果有结果不断从一个线程到另一个线程那么可以使用消息队列（见生产者/消费者） 
- `JDK` 中，`join` 的实现、`Future` 的实现，采用的就是此模式 
- 因为要等待另一方的结果，因此归类到同步模式

![](同步模式之保护性暂停.png)

#### 2. 实现

```java
package com.java.demo.pattern.synchronous;

import lombok.extern.slf4j.Slf4j;

@Slf4j(topic = "c.gurededObject")
public class GuardedObject {

    private Object response;

    public Object get(){
        synchronized (this){
            while (this.response == null){
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
            return this.response;
        }
    }

    public void set(Object response){
        synchronized (this){
            this.response = response;
            this.notifyAll();
        }
    }
}
```

##### 应用

```java
package com.java.demo.pattern.synchronous;

import lombok.extern.slf4j.Slf4j;
import java.io.IOException;
import java.util.List;

@Slf4j(topic = "c.GuardedObjectTest")
public class GuardedObjectTest {
    public static void main(String[] args) throws IOException {
        GuardedObject guardedObject = new GuardedObject();
        new Thread(() -> {
            try {
                // 子线程执行下载
                List<String> response = Downloader.download();
                log.debug("download complete...");
                guardedObject.set(response);
            } catch (IOException | InterruptedException e) {
                e.printStackTrace();
            }
        },"downloader").start();
        log.debug("waiting...");
        // 主线程阻塞等待
        Object response = guardedObject.get();
        log.debug("get response: [{}] lines", ((List<String>) response).size());

    }
}
```

执行结果

```
19:19:58 [main] c.test - waiting...
19:20:02 [downloader] c.test - download complete...
19:20:02 [main] c.test - get response: [2] lines
```

#### 3. 带超时版的GuardedObject

```java
package com.java.demo.pattern.synchronous;

import lombok.extern.slf4j.Slf4j;

@Slf4j(topic = "c.TimeOutGuardedObject")
public class TimeOutGuardedObject {
    private Object response;

    /**
     * 获取线程执行结果
     * @param timeout 超时时间
     * @return
     */
    public Object get(long timeout){
        synchronized (this){
            // 开始时间
            long begin = System.currentTimeMillis(); // 15:00:00
            // 经历时间
            long period = 0;
            while (this.response == null){
                long waitTime = timeout - period;
                log.debug("waitTime : {}", waitTime);
                // 经历的时间大于最大等待时间，退出循环
                if (waitTime <= 0){
                    break;
                }
                try {
                    this.wait(waitTime);  // 虚假唤醒  15:00:01
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                period = System.currentTimeMillis() - begin;  // 15:00:02  1s
                log.debug("period: {}, object is null {}",
                        period, response == null);
            }
            return this.response;
        }
    }


    public void set(Object response){
        synchronized (this){
            this.response = response;
            this.notifyAll();
        }
    }
}
```

##### 测试

- 超时

  ```java
  package com.java.demo.pattern.synchronous;
  
  import lombok.extern.slf4j.Slf4j;
  
  import java.io.IOException;
  import java.util.List;
  
  @Slf4j(topic = "c.TimeOutGuardedObjectTest")
  public class TimeOutGuardedObjectTest {
  
      public static void main(String[] args) {
          TimeOutGuardedObject timeOutGuardedObject = new TimeOutGuardedObject();
          new Thread(()->{
              // 子线程执行下载
              try {
                  List<String> response = Downloader.download();
                  timeOutGuardedObject.set(response);
              } catch (IOException|InterruptedException e) {
                  throw new RuntimeException(e);
              }
              log.debug("download complete...");
  
          },"downloader").start();
          log.debug("waiting...");
          // 主线程阻塞等待
          Object response = timeOutGuardedObject.get(1000);
          log.debug("get response: [{}] lines", (response == null ? "response is null":((List<String>) response).size()));
      }
  }
  ```

  执行结果

  ```bash
  19:48:29 [main] c.TimeOutGuardedObjectTest - waiting...
  19:48:29 [main] c.TimeOutGuardedObject - waitTime : 1000
  19:48:30 [main] c.TimeOutGuardedObject - period: 1009, object is null true
  19:48:30 [main] c.TimeOutGuardedObject - waitTime : -9
  19:48:30 [main] c.TimeOutGuardedObjectTest - get response: [response is null] lines
  19:48:34 [downloader] c.TimeOutGuardedObjectTest - download complete...
  ```

- 不超时

  ```java
  package com.java.demo.pattern.synchronous;
  
  import lombok.extern.slf4j.Slf4j;
  
  import java.io.IOException;
  import java.util.List;
  
  @Slf4j(topic = "c.TimeOutGuardedObjectTest")
  public class TimeOutGuardedObjectTest {
  
      public static void main(String[] args) {
          TimeOutGuardedObject timeOutGuardedObject = new TimeOutGuardedObject();
          new Thread(()->{
              // 子线程执行下载
              try {
                  List<String> response = Downloader.download();
                  timeOutGuardedObject.set(response);
              } catch (IOException|InterruptedException e) {
                  throw new RuntimeException(e);
              }
              log.debug("download complete...");
  
          },"downloader").start();
          log.debug("waiting...");
          // 主线程阻塞等待
          Object response = timeOutGuardedObject.get(5000);
          log.debug("get response: [{}] lines", (response == null ? "response is null":((List<String>) response).size()));
      }
  }
  ```

  执行结果

  ```bash
  20:05:57 [main] c.TimeOutGuardedObjectTest - waiting...
  20:05:57 [main] c.TimeOutGuardedObject - waitTime : 5000
  20:06:01 [main] c.TimeOutGuardedObject - period: 4131, object is null false
  20:06:01 [downloader] c.TimeOutGuardedObjectTest - download complete...
  20:06:01 [main] c.TimeOutGuardedObjectTest - get response: [2] lines
  ```

##### 原理 之 join

`join`是`Thread`类的方法。

```java
    /**
     * Waits at most {@code millis} milliseconds for this thread to
     * die. A timeout of {@code 0} means to wait forever.
     *
     * <p> This implementation uses a loop of {@code this.wait} calls
     * conditioned on {@code this.isAlive}. As a thread terminates the
     * {@code this.notifyAll} method is invoked. It is recommended that
     * applications not use {@code wait}, {@code notify}, or
     * {@code notifyAll} on {@code Thread} instances.
     *
     * @param  millis
     *         the time to wait in milliseconds
     *
     * @throws  IllegalArgumentException
     *          if the value of {@code millis} is negative
     *
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

> 注意 
>
> `join` 体现的是**保护性暂停**模式，请参考之

#### 4. 多任务版 GuardedObject

![](多任务版GuardedObject.png)

图中 `Futures` 就好比居民楼一层的**信箱**（每个信箱有房间编号），左侧的` t0`，`t2`，`t4` 就好比等待邮件的居民，右 侧的 `t1`，`t3`，`t5` 就好比邮递员 如果需要在多个类之间使用 `GuardedObject` 对象，作为参数传递不是很方便，因此设计一个用来解耦的中间类， 这样不仅能够解耦【结果等待者】和【结果生产者】，还能够同时支持多个任务的管理。

##### 实现

```java
package com.java.demo.pattern.synchronous;

import lombok.extern.slf4j.Slf4j;

import java.util.Hashtable;
import java.util.Map;
import java.util.Set;

@Slf4j(topic = "c.MultitaskingGuardedObject")
public class MultitaskingGuardedObject {

    // 标识执行不同任务的GuardObject
    private int id;

    public MultitaskingGuardedObject(int id) {
        this.id = id;
    }

    public int getId() {
        return id;
    }

    // 存储线程执行任务的结果
    private Object response;

    /**
     * 获取线程执行任务的结果
     * @param timeOut 超时时间
     * @return
     */
    public Object get(long timeOut){
        synchronized (this) {
            long begin = System.currentTimeMillis();
            long period = 0;
            while(this.response == null){
                long waitTime = timeOut - period;
                if(waitTime <= 0){
                    break;
                }
                try {
                    this.wait(waitTime);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                period = System.currentTimeMillis() - begin;
            }
           return this.response;
        }
    }

    /**
     * 设置线程执行任务的返回结果
     * @param response
     */
    public void set(Object response){
        synchronized (this){
            this.response = response;
            this.notifyAll();
        }
    }
}


class GuardObjectHolder{

    private static Map<Integer, MultitaskingGuardedObject> multitaskingGuardedObjectMap = new Hashtable<>();

    private static int id = 1;

    private static synchronized int generateId(){
        return id++;
    }

    public static MultitaskingGuardedObject createMultitaskingGuardedObject(){
        MultitaskingGuardedObject multitaskingGuardedObject = new MultitaskingGuardedObject(generateId());
        multitaskingGuardedObjectMap.put(multitaskingGuardedObject.getId(),multitaskingGuardedObject);
        return multitaskingGuardedObject;
    }

    public static Set<Integer> getIds(){
        return multitaskingGuardedObjectMap.keySet();
    }

    public static MultitaskingGuardedObject getMultitaskingGuardedObject(int id){
        return multitaskingGuardedObjectMap.remove(id);
    }
}




@Slf4j(topic = "c.Sender")
class Sender extends Thread {

    private int id;

    private String message;

    public Sender(int id, String message) {
        this.id = id;
        this.message = message;
    }


    @Override
    public void run(){
        MultitaskingGuardedObject multitaskingGuardedObject = GuardObjectHolder.getMultitaskingGuardedObject(id);
        log.debug("发送消息 id = {},发送的消息内容 receiveContent = {}",id,message);
        multitaskingGuardedObject.set(message);

    }
}



@Slf4j(topic = "c.Receiver")
class Receiver extends Thread {
    @Override
    public void run(){
        MultitaskingGuardedObject multitaskingGuardedObject = GuardObjectHolder.createMultitaskingGuardedObject();
        log.debug("等待接收消息中 id = {}",multitaskingGuardedObject.getId());
        Object receiveContent = multitaskingGuardedObject.get(5000);
        log.debug("收到发送的消息 id = {},收到发送的消息内容 receiveContent = {}",multitaskingGuardedObject.getId(),receiveContent);
    }
}
```

##### 测试

```java
package com.java.demo.pattern.synchronous;

import java.util.concurrent.TimeUnit;

public class MultiGuardObjectTest {
    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 3; i++) {
            Receiver receiver = new Receiver();
            receiver.setName("接收者" + (i+1));
            receiver.start();
        }
        TimeUnit.SECONDS.sleep(1);
        for (int id: GuardObjectHolder.getIds()) {
            Sender sender = new Sender(id,"消息"+id);
            sender.setName("发送者" + id);
            sender.start();
        }

    }
}


```

测试结果

```java
13:42:51 [接收者2] c.Receiver - 等待接收消息中 id = 3
13:42:51 [接收者1] c.Receiver - 等待接收消息中 id = 1
13:42:51 [接收者3] c.Receiver - 等待接收消息中 id = 2
13:42:52 [发送者3] c.Sender - 发送消息 id = 3,发送的消息内容 receiveContent = 消息3
13:42:52 [发送者2] c.Sender - 发送消息 id = 2,发送的消息内容 receiveContent = 消息2
13:42:52 [发送者1] c.Sender - 发送消息 id = 1,发送的消息内容 receiveContent = 消息1
13:42:52 [接收者3] c.Receiver - 收到发送的消息 id = 2,收到发送的消息内容 receiveContent = 消息2
13:42:52 [接收者2] c.Receiver - 收到发送的消息 id = 3,收到发送的消息内容 receiveContent = 消息3
13:42:52 [接收者1] c.Receiver - 收到发送的消息 id = 1,收到发送的消息内容 receiveContent = 消息1
```



### 异步模式之 生产者消费者

#### 1. 定义

与前面的保护性暂停中的 `GuardObject` 不同，不需要产生结果和消费结果的线程一一对应 消费队列可以用来平衡生产和消费的线程资源 生产者仅负责产生结果数据，不关心数据该如何处理，而消费者专心处理结果数据 消息队列是有容量限制的，满时不会再加入数据，空时不会再消耗数据 `JDK` 中各种阻塞队列，采用的就是这种模式

![](生产者-消费者.png)

#### 2.实现

消息类

```java
package com.java.demo.pattern.asynchronous;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class Message {

    private int id;

    private Object message;

}
```

消息队列类

```java
package com.java.demo.pattern.asynchronous;

import lombok.extern.slf4j.Slf4j;

import java.util.LinkedList;

/**
 * 消息队列类，java线程之间通信
 */
@Slf4j(topic = "c.MessageQueue")
public class MessageQueue {

    // 存储消息内容的集合
    private LinkedList<Message> messageList = new LinkedList<>();

    // 队列容量
    private int capity;

    public MessageQueue(int capity){
        this.capity = capity;
    }

    /**
     * 存入消息
     * @param message
     */
    public void push(Message message){
        synchronized (messageList){
            // 检查队列是否已经存储满了
            while(messageList.size() == capity){
                // 满了等待消息消费之后再存入
                log.debug("队列满了，生产者线程等待中...");
                try {
                    messageList.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            // 如果消息队列存储的消息未达到最大容量则存入消息队列尾部
            messageList.addLast(message);
            log.debug("生产消息:{}",message);
            // 唤醒正在等待消费的线程
            messageList.notifyAll();
        }
    }

    /**
     * 消费消息
     * @return
     */
    public Message consume(){
        synchronized (messageList){
            while (messageList.isEmpty()){
                log.debug("队列为空，消费者线程等待中...");
                try {
                    messageList.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            // 从消息队列头部获取消息并返回
            Message message = messageList.removeFirst();
            log.debug("消费消息:{}",message);
            // 唤醒正在等待生产的线程
            messageList.notifyAll();
            return message;
        }

    }
}
```

测试类

```java
package com.java.demo.pattern.asynchronous;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.ProducerConsumerTest")
public class ProducerConsumerTest {
    public static void main(String[] args) {
        // 初始化消息队列
        MessageQueue messageQueue = new MessageQueue(2);
        // 定义生产者线程
        for (int i = 0; i < 3; i++) {
            int id = i + 1;
            new Thread(()->{
                messageQueue.push(new Message(id,"message"+id));
            },"生产者" + id).start();
        }

        // 定义消费者
        new Thread(()->{
            while (true){
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                messageQueue.consume();
            }
        },"消费者").start();
    }
}
```

测试结果

```java
14:11:27 [生产者1] c.MessageQueue - 生产消息:Message(id=1, message=message1)
14:11:27 [生产者3] c.MessageQueue - 生产消息:Message(id=3, message=message3)
14:11:27 [生产者2] c.MessageQueue - 队列满了，生产者线程等待中...
14:11:28 [消费者] c.MessageQueue - 消费消息:Message(id=1, message=message1)
14:11:28 [生产者2] c.MessageQueue - 生产消息:Message(id=2, message=message2)
14:11:29 [消费者] c.MessageQueue - 消费消息:Message(id=3, message=message3)
14:11:30 [消费者] c.MessageQueue - 消费消息:Message(id=2, message=message2)
14:11:31 [消费者] c.MessageQueue - 队列为空，消费者线程等待中...
```

## 9.park&unPark

### 9.1 基本使用

这两个方法是LockSupport类中的方法

```java
// 暂停当前线程
LockSupport.park();
// 恢复某个线程的运行
LockSupport.unpark(暂停线程对象);
```

先park再unpark

```java
package com.java.demo.monitor;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.LockSupport;

@Slf4j(topic = "c.ParkUnParkTest")
public class ParkUnParkTest {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
            log.debug("start...");
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            log.debug("park...");
            LockSupport.park();
            log.debug("resume...");
        },"t1");
        t1.start();

        TimeUnit.SECONDS.sleep(2);

        log.debug("unpark...");
        LockSupport.unpark(t1);
    }
}

```

测试结果

```java
15:02:32 [t1] c.ParkUnParkTest - start...
15:02:33 [t1] c.ParkUnParkTest - park...
15:02:34 [main] c.ParkUnParkTest - unpark...
15:02:34 [t1] c.ParkUnParkTest - resume...
```

先unpark再park

```java
package com.java.demo.monitor;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.LockSupport;

@Slf4j(topic = "c.ParkUnParkTest1")
public class ParkUnParkTest1 {
    public static void main(String[] args) throws InterruptedException {
        Thread t2 = new Thread(()->{
            log.debug("start...");
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            log.debug("park...");
            LockSupport.park();
            log.debug("resume...");

        },"t2");
        t2.start();

        TimeUnit.SECONDS.sleep(1);
        log.debug("unpark...");
        LockSupport.unpark(t2);
    }
}

```

测试结果

```java
15:18:37 [t2] c.ParkUnParkTest1 - start...
15:18:38 [main] c.ParkUnParkTest1 - unpark...
15:18:39 [t2] c.ParkUnParkTest1 - park...
15:18:39 [t2] c.ParkUnParkTest1 - resume...
```

### 9.2 特点

与`Object`的`wait & notify`相比

- `wait，notify`和`notifyAll`必须配合`Object Monitor`一起使用，而`park，unpark`不必。
- `park & unpark`是以线程为单位来**阻塞**和**唤醒**线程，而`notify`只能随机唤醒一个等待线程，`notifyAll`是唤星所有等待线程，就不那么精确。
- `park & unpark`可以先`unpark`,而`wait & notify`不能先`notify`。

### 9.3 原理 之 park unpark 原理

每个线程都有自己的一个 `Parker` 对象，由三部分组成 `_counter` ， `_cond` 和 `_mutex` 打个比喻

- 线程就像一个旅人，Parker 就像他随身携带的背包，条件变量就好比背包中的帐篷。_counter 就好比背包中 的备用干粮（0 为耗尽，1 为充足)
- 调用 park 就是要看需不需要停下来歇息 
  - 如果备用干粮耗尽，那么钻进帐篷歇息 
  - 如果备用干粮充足，那么不需停留，继续前进 
- 调用 unpark，就好比令干粮充足 
  - 如果这时线程还在帐篷，就唤醒让他继续前进 
  - 如果这时线程还在运行，那么下次他调用 park 时，仅是消耗掉备用干粮，不需停留继续前进 
    - 因为背包空间有限，多次调用 unpark 仅会补充一份备用干粮

![](park1unpark2.png)

1. 当前线程调用 Unsafe.park() 方法 
2. 检查 _counter ，本情况为 0，这时，获得 _mutex 互斥锁 
3. 线程进入 _cond 条件变量阻塞 
4. 设置 _counter = 0

![](unpark1park2.png)

1. 调用 Unsafe.unpark(Thread_0) 方法，设置 _counter 为 1 
2. 唤醒 _cond 条件变量中的 Thread_0 
3. Thread_0 恢复运行 
4. 设置 _counter 为 0

------

![](unpark_park.png)



1. 调用 Unsafe.unpark(Thread_0) 方法，设置 _counter 为 1 
2. 当前线程调用 Unsafe.park() 方法 
3. 检查 _counter ，本情况为 1，这时线程无需阻塞，继续运行 
4. 设置 _counter 为 0

## 10. 线程状态转换

![](线程状态切换.png)

> 以下情况假设有线程`Thread t`

### 情况1 `NEW ➡ RUNNABLE`

- 当调用`t.start()`方法时，由`NEW➡RUNNABLE`

### 情况2 `RUNNABLE ⬅➡ WAITING`

t线程用`synchronized(obj)`获取对象锁后

- 调用`obj.wait()`方法时，他线程从`RUNNABLE ➡WAITING`
- 调用`obj.notify()`,`obj.notifyAll()`,`t.interrupt()`时，
  - 竞争锁成功，t线程从`WAITING ➡ RUNNABLE`
  - 竞争锁失败，t线程从`WAITING ➡BLOCKED`

### 情况3 `RUNNABLE⬅➡WAITING`

- 当前线程调用`t.join()`方法时，当前线程从`RUNNABLE➡WAITING`
  - 注意是当前线程在t线程对象的监视器上等待
- t线程运行结束，或调用了当前线程的`interrupt()`时，当前线程从`WAITING➡RUNNABLE`

### 情况4 `RUNNABLE⬅➡WAITING`

- 当前线程调用 `LockSupport.park()` 方法会让当前线程从 `RUNNABLE ➡ WAITING` 调用
- `LockSupport.unpark(目标线程)` 或调用了线程 的 `interrupt()` ，会让目标线程从 `WAITING ➡ RUNNABLE`

### 情况5 `RUNNABLE⬅➡TIMED_WAITING`

- t 线程用 `synchronized(obj)` 获取了对象锁后 
  - 调用 `obj.wait(long n)` 方法时，t 线程从 `RUNNABLE ➡TIMED_WAITING` 
  - t 线程等待时间超过了 n 毫秒，或调用 `obj.notify()` ， `obj.notifyAll()` ， `t.interrupt()` 时 
    - 竞争锁成功，t 线程从 `TIMED_WAITING ➡ RUNNABLE` 
    - 竞争锁失败，t 线程从 `TIMED_WAITING ➡ BLOCKED`

### 情况6 `RUNNABLE⬅➡TIMED_WAITING`

- 当前线程调用 `t.join(long n)` 方法时，当前线程从 `RUNNABLE➡TIMED_WAITING` 
  - 注意是当前线程在t 线程对象的监视器上等待
-  当前线程等待时间超过了 n 毫秒，或t 线程运行结束，或调用了当前线程的 `interrupt()` 时，当前线程从 `TIMED_WAITING ➡ RUNNABLE`

### 情况7 `RUNNABLE⬅➡TIMED_WAITING`

- 当前线程调用 `Thread.sleep(long n)` ，当前线程从 `RUNNABLE ➡ TIMED_WAITING` 
- 当前线程等待时间超过了 n 毫秒，当前线程从 `TIMED_WAITING ➡ RUNNABLE`

### 情况8 `RUNNABLE⬅➡TIMED_WAITING`

- 当前线程调用 `LockSupport.parkNanos(long nanos)` 或 `LockSupport.parkUntil(long millis)` 时，当前线 程从 `RUNNABLE ➡ TIMED_WAITING `
- 调用 `LockSupport.unpark(目标线程)` 或调用了线程 的 `interrupt()` ，或是等待超时，会让目标线程从 `TIMED_WAITING➡RUNNABLE`

### 情况9 `RUNNABLE ⬅➡ BLOCKED`

- t 线程用 `synchronized(obj)` 获取了对象锁时如果竞争失败，从 `RUNNABLE ➡ BLOCKED` 
- 持 obj 锁线程的同步代码块执行完毕，会唤醒该对象上所有 `BLOCKED` 的线程重新竞争，如果其中 t 线程竞争 成功，从 `BLOCKED ➡ RUNNABLE` ，其它失败的线程仍然 `BLOCKED`

### 情况10  `RUNNABLE ⬅➡ TERMINATED`

当前线程所有代码运行完毕，进入 `TERMINATED`

## 11. 多把锁

一间大屋子有两个功能：睡觉、学习，互不相干。 现在小南要学习，小女要睡觉，但如果只用一间屋子（一个对象锁）的话，那么并发度很低 解决方法是准备多个房间（多个对象锁）

示例

```java
package com.java.demo.monitor;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic="c.MultiLockTest")
public class MultiLockTest {
    public static void main(String[] args) {
        BigRooms bigRooms = new BigRooms();
        new Thread(()->{
            try {
                bigRooms.sleep();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },"小南").start();

        new Thread(()->{
            try {
                bigRooms.study();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },"小女").start();
    }
}


@Slf4j(topic = "c.BigRooms")
class BigRooms{

    public void sleep() throws InterruptedException {
        synchronized (this) {
            log.debug("休息1h");
            TimeUnit.SECONDS.sleep(1);
        }
    }

    public void study() throws InterruptedException {
        synchronized (this) {
            log.debug("学习2h");
            TimeUnit.SECONDS.sleep(2);
        }
    }


}
```

执行结果

```java
11:21:11 [小南] c.BigRooms - 休息1h
11:21:12 [小女] c.BigRooms - 学习2h
```

细分锁粒度

```java
package com.java.demo.monitor;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic="c.MultiLockTest01")
public class MultiLockTest01 {
    public static void main(String[] args) {
        Rooms bigRooms = new Rooms();
        new Thread(()->{
            try {
                bigRooms.sleep();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },"小南").start();

        new Thread(()->{
            try {
                bigRooms.study();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },"小女").start();
    }
}


@Slf4j(topic = "c.BigRooms")
class Rooms{

    private final Object studyRoom = new Object();

    private final Object bedRoom = new Object();


    public void sleep() throws InterruptedException {
        synchronized (bedRoom) {
            log.debug("休息1h");
            TimeUnit.SECONDS.sleep(1);
        }
    }

    public void study() throws InterruptedException {
        synchronized (studyRoom) {
            log.debug("学习2h");
            TimeUnit.SECONDS.sleep(2);
        }
    }


}
```

执行结果

```java
11:22:12 [小南] c.BigRooms - 休息1h
11:22:12 [小女] c.BigRooms - 学习2h
```

将锁的粒度细分 

- 好处，是可以增强并发度 
- 坏处，如果一个线程需要同时获得多把锁，就容易发生死锁

## 12. 活跃性

### 死锁

有这样的情况：一个线程需要同时获取多把锁，这时就容易发生死锁 `t1` 线程 获得 `o1`对象 锁，接下来想获取 `o2`对象 的锁 `t2` 线程 获得 `o2`对象 锁，接下来想获取 `o1`对象 的锁 例：

```java
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.DeadLockTest")
public class DeadLockTest {

    public static void main(String[] args) {
        Object o1 = new Object();
        Object o2 = new Object();
        Thread t1 = new Thread(() -> {
            synchronized (o1) {
                log.debug("lock o1");
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                synchronized (o2) {
                    log.debug("lock o2");
                    log.debug("操作...");
                }
            }
        }, "t1");


        Thread t2 = new Thread(() -> {
            synchronized (o2) {
                log.debug("lock o2");
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                synchronized (o1) {
                    log.debug("lock o1");
                    log.debug("操作...");
                }
            }
        }, "t2");

        t1.start();
        t2.start();

    }
}

```

结果

```java
11:54:58 [t2] c.DeadLockTest - lock o2
11:54:58 [t1] c.DeadLockTest - lock o1
```

### 定位死锁

检测死锁可以使用 `jconsole`工具，或者使用` jps` 定位进程 `id`，再用 `jstack` 定位死锁：

```
PS D:\idea_projects\java-example> jps
295844 Launcher
7236     
65464 Jps
289576 DeadLockTest
PS D:\idea_projects\java-example> jstack 289576
2022-05-30 13:30:47
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.251-b08 mixed mode):

"DestroyJavaVM" #14 prio=5 os_prio=0 tid=0x0000000003373000 nid=0x48aac waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"t2" #13 prio=5 os_prio=0 tid=0x00000000200fb000 nid=0x31d70 waiting for monitor entry [0x000000002060e000]      
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.java.demo.monitor.DeadLockTest.lambda$main$1(DeadLockTest.java:38)
        - waiting to lock <0x000000076d0ac540> (a java.lang.Object)
        - locked <0x000000076d0ac550> (a java.lang.Object)
        at com.java.demo.monitor.DeadLockTest$$Lambda$2/1416233903.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

"t1" #12 prio=5 os_prio=0 tid=0x00000000200fa800 nid=0x41234 waiting for monitor entry [0x000000002050f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.java.demo.monitor.DeadLockTest.lambda$main$0(DeadLockTest.java:22)
        - waiting to lock <0x000000076d0ac550> (a java.lang.Object)
        - locked <0x000000076d0ac540> (a java.lang.Object)
        at com.java.demo.monitor.DeadLockTest$$Lambda$1/787387795.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

"Service Thread" #11 daemon prio=9 os_prio=0 tid=0x000000001ed09000 nid=0x44e10 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C1 CompilerThread3" #10 daemon prio=9 os_prio=2 tid=0x000000001ec6b800 nid=0x48704 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread2" #9 daemon prio=9 os_prio=2 tid=0x000000001ec5e800 nid=0x2c260 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread1" #8 daemon prio=9 os_prio=2 tid=0x000000001ec5a800 nid=0x46d84 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #7 daemon prio=9 os_prio=2 tid=0x000000001ec59000 nid=0x455fc waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Monitor Ctrl-Break" #6 daemon prio=5 os_prio=0 tid=0x000000001ec56000 nid=0x475c4 runnable [0x000000001f46e000]
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
        at java.net.SocketInputStream.read(SocketInputStream.java:171)
        at java.net.SocketInputStream.read(SocketInputStream.java:141)
        at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284)
        at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326)
        at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178)
        - locked <0x000000076c6449c0> (a java.io.InputStreamReader)
        at java.io.InputStreamReader.read(InputStreamReader.java:184)
        at java.io.BufferedReader.fill(BufferedReader.java:161)
        at java.io.BufferedReader.readLine(BufferedReader.java:324)
        - locked <0x000000076c6449c0> (a java.io.InputStreamReader)
        at java.io.BufferedReader.readLine(BufferedReader.java:389)
        at com.intellij.rt.execution.application.AppMainV2$1.run(AppMainV2.java:49)

"Attach Listener" #5 daemon prio=5 os_prio=2 tid=0x000000001eb8e000 nid=0x438e0 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" #4 daemon prio=9 os_prio=2 tid=0x000000001eb8d000 nid=0x466cc runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Finalizer" #3 daemon prio=8 os_prio=1 tid=0x000000001eb21000 nid=0x46710 in Object.wait() [0x000000001f0ff000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x000000076c388ee0> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:144)
        - locked <0x000000076c388ee0> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:165)
        at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:216)

"Reference Handler" #2 daemon prio=10 os_prio=2 tid=0x000000001eb20800 nid=0x43f74 in Object.wait() [0x000000001effe000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x000000076c386c00> (a java.lang.ref.Reference$Lock)
        at java.lang.Object.wait(Object.java:502)
        at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
        - locked <0x000000076c386c00> (a java.lang.ref.Reference$Lock)
        at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)

"VM Thread" os_prio=2 tid=0x000000001cd18800 nid=0x46ad4 runnable

"GC task thread#0 (ParallelGC)" os_prio=0 tid=0x0000000003388800 nid=0x3b034 runnable

"GC task thread#1 (ParallelGC)" os_prio=0 tid=0x000000000338a000 nid=0x47e3c runnable

"GC task thread#2 (ParallelGC)" os_prio=0 tid=0x000000000338b800 nid=0x48a60 runnable

"GC task thread#3 (ParallelGC)" os_prio=0 tid=0x000000000338d800 nid=0x47aac runnable

"GC task thread#4 (ParallelGC)" os_prio=0 tid=0x000000000338f800 nid=0x473c8 runnable

"GC task thread#5 (ParallelGC)" os_prio=0 tid=0x0000000003391800 nid=0x48494 runnable

"GC task thread#6 (ParallelGC)" os_prio=0 tid=0x0000000003395000 nid=0x46dec runnable

"GC task thread#7 (ParallelGC)" os_prio=0 tid=0x0000000003396000 nid=0x4794c runnable

"VM Periodic Task Thread" os_prio=2 tid=0x000000001ed8a000 nid=0x482b4 waiting on condition

JNI global references: 316


Found one Java-level deadlock:
=============================
"t2":
  waiting to lock monitor 0x000000001cd20e58 (object 0x000000076d0ac540, a java.lang.Object),
  which is held by "t1"
"t1":
  waiting to lock monitor 0x000000001cd23218 (object 0x000000076d0ac550, a java.lang.Object),
  which is held by "t2"

Java stack information for the threads listed above:
===================================================
"t2":
        at com.java.demo.monitor.DeadLockTest.lambda$main$1(DeadLockTest.java:38)
        - waiting to lock <0x000000076d0ac540> (a java.lang.Object)
        - locked <0x000000076d0ac550> (a java.lang.Object)
        at com.java.demo.monitor.DeadLockTest$$Lambda$2/1416233903.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)
"t1":
        at com.java.demo.monitor.DeadLockTest.lambda$main$0(DeadLockTest.java:22)
        - waiting to lock <0x000000076d0ac550> (a java.lang.Object)
        - locked <0x000000076d0ac540> (a java.lang.Object)
        at com.java.demo.monitor.DeadLockTest$$Lambda$1/787387795.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

Found 1 deadlock.

PS D:\idea_projects\java-example> 
```

### 哲学家就餐—死锁问题

![](哲学家就餐.png)



有五位哲学家，围坐在圆桌旁。 他们只做两件事，思考和吃饭，思考一会吃口饭，吃完饭后接着思考。 吃饭时要用两根筷子吃，桌上共有 5 根筷子，每位哲学家左右手边各有一根筷子。 如果筷子被身边的人拿着，自己就得等待。

```java
import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class Chopstick {

    private String name;

    @Override
    public String toString() {
        return "筷子{" + name + '}';
    }
}
```

```java



import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.Philpsopher")
public class Philosopher extends Thread{

    private Chopstick left;

    private Chopstick right;

    public Philosopher(String name,Chopstick left,Chopstick right){
        super(name);
        this.left = left;
        this.right = right;
    }

    public void eat() {
        log.debug("eating...");
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void run() {
        while (true) {
            // 获得左手筷子
            synchronized (left) {
                // 获得右手筷子
                synchronized (right) {
                    // 吃饭
                    eat();
                }
                // 放下右手筷子
            }
            // 放下左手筷子
        }
    }
}
```

```java
public class DeadLockTest {
    public static void main(String[] args) {
        Chopstick c1 = new Chopstick("1");
        Chopstick c2 = new Chopstick("2");
        Chopstick c3 = new Chopstick("3");
        Chopstick c4 = new Chopstick("4");
        Chopstick c5 = new Chopstick("5");
        new Philosopher("苏格拉底", c1, c2).start();
        new Philosopher("柏拉图", c2, c3).start();
        new Philosopher("亚里士多德", c3, c4).start();
        new Philosopher("赫拉克利特", c4, c5).start();
        new Philosopher("阿基米德", c5, c1).start();
    }
}

```

执行一小会，执行不下去了，卡住了

```java
14:17:22 [苏格拉底] c.Philpsopher - eating...
14:17:22 [亚里士多德] c.Philpsopher - eating...
14:17:23 [亚里士多德] c.Philpsopher - eating...
14:17:23 [阿基米德] c.Philpsopher - eating...
14:17:24 [阿基米德] c.Philpsopher - eating...
14:17:24 [亚里士多德] c.Philpsopher - eating...
14:17:25 [阿基米德] c.Philpsopher - eating...
```

使用 `jconsole` 检测死锁，发现

```java
名称: 阿基米德
状态: com.java.demo.monitor.deadlock.v1.Chopstick@609a727c上的BLOCKED, 拥有者: 苏格拉底
总阻止数: 4, 总等待数: 3

堆栈跟踪: 
com.java.demo.monitor.deadlock.v1.Philosopher.run(Philosopher.java:37)
   - 已锁定 com.java.demo.monitor.deadlock.v1.Chopstick@28362027  
--------------------------------------------------------------------
名称: 苏格拉底
状态: com.java.demo.monitor.deadlock.v1.Chopstick@39722420上的BLOCKED, 拥有者: 柏拉图
总阻止数: 2, 总等待数: 1

堆栈跟踪: 
com.java.demo.monitor.deadlock.v1.Philosopher.run(Philosopher.java:37)
   - 已锁定 com.java.demo.monitor.deadlock.v1.Chopstick@609a727c
-------------------------------------------------------------------- 
名称: 柏拉图
状态: com.java.demo.monitor.deadlock.v1.Chopstick@704889ef上的BLOCKED, 拥有者: 亚里士多德
总阻止数: 2, 总等待数: 0

堆栈跟踪: 
com.java.demo.monitor.deadlock.v1.Philosopher.run(Philosopher.java:37)
   - 已锁定 com.java.demo.monitor.deadlock.v1.Chopstick@39722420
--------------------------------------------------------------------
名称: 亚里士多德
状态: com.java.demo.monitor.deadlock.v1.Chopstick@39eac34d上的BLOCKED, 拥有者: 赫拉克利特
总阻止数: 12, 总等待数: 4

堆栈跟踪: 
com.java.demo.monitor.deadlock.v1.Philosopher.run(Philosopher.java:37)
   - 已锁定 com.java.demo.monitor.deadlock.v1.Chopstick@704889ef
--------------------------------------------------------------------
名称: 赫拉克利特
状态: com.java.demo.monitor.deadlock.v1.Chopstick@28362027上的BLOCKED, 拥有者: 阿基米德
总阻止数: 2, 总等待数: 0

堆栈跟踪: 
com.java.demo.monitor.deadlock.v1.Philosopher.run(Philosopher.java:37)
   - 已锁定 com.java.demo.monitor.deadlock.v1.Chopstick@39eac34d


```

### 活锁

活锁出现在两个线程互相改变对方的结束条件，最后谁也无法结束，例如

```java
package com.java.demo.monitor.livelock;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.LiveLockTest")
public class LiveLockTest {

    private  static volatile int count = 10;


    public static void main(String[] args) {
        new Thread(()->{
            // 期望减到 0 退出循环
            while (count > 0) {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                count--;
                log.debug("count: {}", count);
            }
        },"t1").start();

        new Thread(()->{
            // 期望减到 0 退出循环
            while (count < 20) {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                count++;
                log.debug("count: {}", count);
            }
        },"t2").start();
    }
}

```

### 饥饿

很多教程中把饥饿定义为，一个线程由于优先级太低，始终得不到 CPU 调度执行，也不能够结束，饥饿的情况不易演示，读写锁时会涉及饥饿问题。

用顺序加锁的方式解决之前的死锁问题

![](死锁.png)

顺序加锁的解决方案

![](死锁解决方案_顺序加锁.png)

## 13.  ReentrantLock

相对于 synchronized 它具备如下特点 

- 可中断 
- 可以设置超时时间 
- 可以设置为公平锁 
- 支持多个条件变量   

与`synchronized`一样，都支持可重入

**基本语法**

```java
// 获取锁
reentrantLock.lock();
try{
    // 临界区
}funally{
    // 释放锁
    reentrantLock.unlock();
}
```

### 13.1 可重入

**可重入**是指同一个线程如果首次获得了这把锁，那么因为它是这把锁的拥有者，因此有权利再次获取这把锁。 **如果是不可重入锁，那么第二次获得锁时，自己也会被锁挡住**。

```java
package com.java.demo.monitor.reentrantlock;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.locks.ReentrantLock;

@Slf4j(topic = "c.ReentrantTest")
public class ReentrantTest {

    private final static ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) {

        lock.lock();
        try{
            log.debug("excute main method...");
            m1();
        }finally {
            lock.unlock();
        }

    }

    public static void m1(){
        lock.lock();
        try{
            log.debug("excute m1 method...");
            m2();
        }finally {
            lock.unlock();
        }
    }

    public static void m2(){
        lock.lock();
        try{
            log.debug("excute m2 method...");
        }finally {
            lock.unlock();
        }
    }
}

```

执行结果

```java
15:38:40 [main] c.ReentrantTest - excute main method...
15:38:40 [main] c.ReentrantTest - excute m1 method...
15:38:40 [main] c.ReentrantTest - excute m2 method...
```

### 13.2 可打断

```java
package com.java.demo.monitor.reentrantlock;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

@Slf4j(topic = "c.InterruptTest")
public class InterruptTest {

    private static ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {


        Thread t1 = new Thread(() -> {
            try {
                log.debug("尝试去获得锁...");
                lock.lockInterruptibly();
            } catch (InterruptedException e) {
                e.printStackTrace();
                log.debug("获得锁过程中被打断，没有获得锁...");
                return;
            }
            try {
                log.debug("获得锁...");
            } finally {
                lock.unlock();
            }
        }, "t1");

        t1.start();

        lock.lock();
        TimeUnit.SECONDS.sleep(1);
        log.debug("打断t1...");
        t1.interrupt();

    }
}

```

输出：

```java
20:23:22 [t1] c.InterruptTest - 尝试去获得锁...
20:23:23 [main] c.InterruptTest - 打断t1...
20:23:23 [t1] c.InterruptTest - 获得锁过程中被打断，没有获得锁...
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at com.java.demo.monitor.reentrantlock.InterruptTest.lambda$main$0(InterruptTest.java:19)
	at java.lang.Thread.run(Thread.java:748)
```

### 13.3 锁超时

```java
package com.java.demo.monitor.reentrantlock;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

@Slf4j(topic = "c.LockTimeoutTest")
public class LockTimeoutTest {

    private static ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            log.debug("尝试获取锁...");
            try {
                if (!lock.tryLock(2, TimeUnit.SECONDS)){
                    log.debug("获取不到锁...");
                    return;
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }

            try{
                log.debug("获取到锁...");
            }finally {
                lock.unlock();

            }
        }, "t1");



        lock.lock();
        log.debug("获得到锁...");
        t1.start();
        TimeUnit.SECONDS.sleep(1);
        log.debug("释放了锁...");
        lock.unlock();
    }
}

```

输出：

```java
20:24:18 [main] c.LockTimeoutTest - 获得到锁...
20:24:18 [t1] c.LockTimeoutTest - 尝试获取锁...
20:24:19 [main] c.LockTimeoutTest - 释放了锁...
20:24:19 [t1] c.LockTimeoutTest - 获取到锁...
```

### 13.4 公平锁

```java
package com.java.demo.monitor.reentrantlock;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

@Slf4j(topic = "c.FairLockTest")
public class FairLockTest {

    private static ReentrantLock lock = new ReentrantLock(true);

    public static void main(String[] args) throws InterruptedException {
        lock.lock();

        for (int i = 0; i < 500; i++) {
            new Thread(() -> {
                log.debug("{} is start...",Thread.currentThread().getName());
                running();
            }, "t" + (i + 1)).start();
        }

        TimeUnit.SECONDS.sleep(1);
        lock.unlock();
    }

    private static void running(){
        for (int i = 0; i < 5; i++) {
            lock.lock();
            try{
                log.debug("{} 获取到锁...",Thread.currentThread().getName());
            }  finally {
                lock.unlock();
            }
        }
    }


}
```

### 13.5 条件变量

`synchronized` 中也有条件变量，就是我们讲原理时那个 `waitSet` 休息室，当条件不满足时进入 `waitSet` 等待 。

`ReentrantLock` 的条件变量比 `synchronized` 强大之处在于，它是支持多个条件变量的，这就好比 

- `synchronized` 是那些不满足条件的线程都在一间休息室等消息 
- 而 `ReentrantLock` 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤醒 

使用要点：

- `await` 前需要获得锁 
- `await` 执行后，会释放锁，进入 `conditionObject` 等待 
- `await` 的线程被唤醒（或打断、或超时）取重新竞争 `lock` 锁 
- 竞争 `lock` 锁成功后，从 `await` 后继续执行

例子：

```java
package com.java.demo.monitor.reentrantlock;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

@Slf4j(topic = "c.ContionTest")
public class ContionTest {

    private static ReentrantLock lock = new ReentrantLock();

    static Condition waitCigaretteQueue = lock.newCondition();

    static Condition waitTakeoutQueue = lock.newCondition();

    static volatile boolean hasCigarette = false;

    static volatile boolean hasTakeout = false;

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            lock.lock();
            try{
                log.debug("没香烟！先歇会！");
                while (!hasCigarette){
                    waitCigaretteQueue.await();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
            log.debug("有烟了，抽完烟开始干活...");
        },"小南").start();

        new Thread(()->{
            lock.lock();
            try{
                log.debug("没外卖！先歇会！");
                while (!hasTakeout){
                    waitTakeoutQueue.await();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
            log.debug("外卖到了，干完饭开始干活...");
        },"小女").start();

        TimeUnit.SECONDS.sleep(1);

        new Thread(()->{
            lock.lock();
            try{
                log.debug("您好！美团外卖，您的香烟到了！");
                hasCigarette = true;
                waitCigaretteQueue.signal();
            }finally {
                lock.unlock();
            }

        },"送烟的").start();

        TimeUnit.SECONDS.sleep(2);

        new Thread(()->{
            lock.lock();
            try{
                log.debug("您好！饿了么，您的外卖到了！");
                hasTakeout = true;
                waitTakeoutQueue.signal();
            }finally {
                lock.unlock();
            }

        },"送外卖的").start();
    }
}

```

测试输出：

```java
22:16:20 [小南] c.ContionTest - 没香烟！先歇会！
22:16:20 [小女] c.ContionTest - 没外卖！先歇会！
22:16:21 [送烟的] c.ContionTest - 您好！美团外卖，您的香烟到了！
22:16:21 [小南] c.ContionTest - 有烟了，抽完烟开始干活...
22:16:23 [送外卖的] c.ContionTest - 您好！饿了么，您的外卖到了！
22:16:23 [小女] c.ContionTest - 外卖到了，干完饭开始干活...
```

### 13.6 同步模式 之 顺序控制

#### 固定顺序输出

> 比如，必须先 2 后 1 打印

##### wait-notify版

```java
package com.java.demo.pattern.synchronous.sequencecontrol;

import lombok.extern.slf4j.Slf4j;

@Slf4j(topic = "c.WaitNotifyTest")
public class WaitNotifyTest {

    static Object lock = new Object();

    static boolean t2Runed = false;

    public static void main(String[] args) {
    // 需求：固定输出顺序，先输出2，再输出1

        new Thread(()->{
            synchronized (lock){
                while(!t2Runed){
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                log.debug("1");
            }
        },"t1").start();

        new Thread(()->{
            synchronized (lock){
                log.debug("2");
                t2Runed = true;
                lock.notify();
            }
        },"t2").start();
    }
}
```

##### park-unpark版

```java
package com.java.demo.pattern.synchronous.sequencecontrol;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.locks.LockSupport;

@Slf4j(topic = "c.ParkUnParkTest")
public class ParkUnParkTest {
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            LockSupport.park();
            log.debug("1");
        }, "t1");

        Thread t2 = new Thread(() -> {
            log.debug("2");
            LockSupport.unpark(t1);
        }, "t2");

        t1.start();
        t2.start();
    }
}
```

#### 交替输出

> 线程 1 输出 a 5 次，线程 2 输出 b 5 次，线程 3 输出 c 5 次。现在要求输出 `abcabcabcabcabc` 怎么实现

##### wait-notify版

```java
package com.java.demo.pattern.synchronous.sequencecontrol;

import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j(topic = "c.WaitNotifyTest1")
public class WaitNotifyTest1 {
    public static void main(String[] args) {
        WaitNotify waitNotify = new WaitNotify(1, 5);
        new Thread(() -> {
            waitNotify.print("a", 1, 2);
        }).start();

        new Thread(() -> {
            waitNotify.print("b", 2, 3);
        }).start();

        new Thread(() -> {
            waitNotify.print("c", 3, 1);
        }).start();
    }
}

/**
 * 输出内容    等待标记    下一个标记
 * a            1          2
 * b            2          3
 * c            3          1
 */
@Slf4j(topic = "c.WaitNotify")
@AllArgsConstructor
class WaitNotify {
    // 等待标记
    private int flag;

    // 循环次数
    private int loopNum;

    // 打印方法
    public void print(String printStr, int waitFlag, int nextFlag) {
        for (int i = 0; i < loopNum; i++) {
            synchronized (this) {
                while (waitFlag != flag) {
                    try {
                        this.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                log.debug(printStr);
                flag = nextFlag;
                this.notifyAll();
            }
        }

    }

}

```

##### ReentrantLock条件变量版

```java
package com.java.demo.pattern.synchronous.sequencecontrol;

import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class AwaitSingleTest {

    public static void main(String[] args) throws InterruptedException {
        AwaitSingle awaitSingle = new AwaitSingle(5);
        Condition conditionA = awaitSingle.newCondition();
        Condition conditionB = awaitSingle.newCondition();
        Condition conditionC = awaitSingle.newCondition();

        new Thread(()->{
            awaitSingle.print("a",conditionA,conditionB);
        }).start();
        new Thread(()->{
            awaitSingle.print("b",conditionB,conditionC);
        }).start();
        new Thread(()->{
            awaitSingle.print("c",conditionC,conditionA);
        }).start();

        TimeUnit.SECONDS.sleep(1);

        awaitSingle.lock();
        try {
            conditionA.signal();
        }finally {
            awaitSingle.unlock();
        }



    }
}

@Slf4j(topic = "c.AwaitSingle")
@AllArgsConstructor
class AwaitSingle extends ReentrantLock {

    private int loopNum;

    public void print(String printStr, Condition current, Condition next){
        for (int i = 0; i < loopNum; i++) {
            lock();
            try{
                current.await();
                log.debug(printStr);
                next.signal();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                unlock();
            }
        }

    }
}

```

##### Park Unpark版

```java
package com.java.demo.pattern.synchronous.sequencecontrol;

import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.locks.LockSupport;

public class ParkUnparkTest1 {

    static Thread a;static Thread b;static Thread c;

    public static void main(String[] args) {
        ParkUnpark parkUnpark = new ParkUnpark(5);

        a = new Thread(() -> {
            parkUnpark.print("a", b);
        });


        b = new Thread(() -> {
            parkUnpark.print("b", c);
        });

        c = new Thread(() -> {
            parkUnpark.print("c", a);
        });

        a.start();
        b.start();
        c.start();

        LockSupport.unpark(a);


    }
}

@Slf4j(topic = "c.ParkUnpark")
@AllArgsConstructor
class ParkUnpark{

    private int loopNum;

    public void print(String printStr,Thread t){
        for (int i = 0; i < loopNum; i++) {
            LockSupport.park();
            log.debug(printStr);
            LockSupport.unpark(t);
        }
    }
}

```





---

参考资料：  
[黑马程序员全面深入学习Java并发编程，JUC并发编程全套教程](https://www.bilibili.com/video/BV16J411h7Rd)
[What is in Java object header?](https://stackoverflow.com/questions/26357186/what-is-in-java-object-header)  
[OpenJDK WIKI](https://wiki.openjdk.java.net/)  
[JAVA对象布局之对象头(Object Header)](https://segmentfault.com/a/1190000037643624)  
[面试官发问：“对象头(object header)”里知多少？](https://zhuanlan.zhihu.com/p/124278272)  
[Java的对象模型——Oop-Klass模型（一）](https://zhuanlan.zhihu.com/p/104494807)  

> JVM 32bit 和JVM 64bit的区别如下:   
>
>   1. 目前只有server VM支持64bit JVM，client不支持32bit JVM。   
>   2. The Java Plug-in, AWT Robot and Java Web Start这些组件目前不支持64bit JVM 
>   3. 本地代码的影响：对JNI的编程接口没有影响，但是针对32-bit VM写的代码必须重新编译才能在64-bit VM工作。 
>   4. 32-bit JVM堆大小最大是4G, 64-bit VMs 上, Java堆的大小受限于物理内存和操作系统提供的虚拟内存。(这里的堆并不严谨) 
>   5. 线程的默认堆栈大小：在windows上32位JVM,默认堆栈最大是320k 64-bit JVM是1024K。 
>   6. 性能影响: 
>      (1)64bit JVM相比32bit JVM,在大量的内存访问的情况下，其性能损失更少，AMD64和EM64T平台在64位模式下运行时，Java虚拟机得到了一些额外的寄存器，它可以用来生成更有效的原生指令序列。 
>      (2)性能上，在SPARC 处理器上，当一个java应用程序从32bit 平台移植到64bit平台的64bit JVM会用大约 10-20%的性能损失，而在AMD64和 EM64T平台上，其性能损失的范围在0-15%. 

以上摘自http://java.sun.com/docs/hotspot/HotSpotFAQ.html#64bit_description 

[Java微基准测试框架JMH - 知乎 (zhihu.com)](