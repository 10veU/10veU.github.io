---
title: Java IO流
date: 2021-09-11 23:00:43
categories: 
    - Java
    - IO
cover: /img/post_cover/IO.jpg
tags:
    - Java
    - IO
---
# Java IO流
## 1. 什么是文件？
从编程的角度看，文件就是保存数据的载体。可以是文字，图片，音频，视频...
## 2. 文件流
文件再程序中以流的形式来操作。  
![文件流](File.jpg)  
**流**  
数据在文件（数据源）和程序（内存）之间经历的路径。   
**输入流**  
数据从文件（数据源）到程序（内存）的路径。  
**输出流**
数据从程序（内存）到数据源（文件）的路径。
## 3. 常用的文件操作  
![File](File.png)
File的构造方法：  
![File](File_Constructor.jpg)
### 3.1 常用的创建文件方法  
**`File(String pathname)`**
```java
/**
     * File(String pathname)
     */
    @Test
    public void createTest01(){
        String pathname = "classpath://../resource/file01/test01.txt";
        File file = new File(pathname);
        try {
            file.createNewFile();
            System.out.println("文件创建成功！");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
**`File(String parent, String child)`**
```java
/**
    * File(String parent, String child)
    */
@Test
public void createTest02(){
    String parent = "classpath://../resource";
    String child = "/file02/test02.txt";
    File file = new File(parent, child);
    try {
        file.createNewFile();
        System.out.println("文件创建成功！");
    } catch (IOException e) {
        e.printStackTrace();
    }
}

```
**`File(File parent, String child)`**
```java
 /**
     * File(File parent, String child)
     */
    @Test
    public void createFile03(){
        File file= new File("classpath://../resource");
        String childFilePath = "/file03/test03.txt";
        File file1 = new File(file, childFilePath);
        try {
            file1.createNewFile();
            System.out.println("文件创建成功！");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
### 3.2 获取文件信息
```java
@Test
    public void fileMethodTest(){
        File file = new File("class://../resource/file01/test01.txt");
        // 获取文件名称
        System.out.println("文件名称："+file.getName());
        // 获取文件绝对路径
        System.out.println("文件绝对路径："+file.getAbsolutePath());
        // 获取文件路径
        System.out.println("文件路径："+file.getPath());
        // 获取文件父级目录
        System.out.println("文件父级目录："+file.getParent());
        // 获取父文件
        System.out.println("父文件："+file.getParentFile());
        // 文件大小
        System.out.println("文件大小(字节)："+file.length());
        // 文件是否存在
        System.out.println("文件是否存在："+file.exists());
        // 是否是一个文件
        System.out.println("是否是一个文件："+file.isFile());
        // 是否是一个目录
        System.out.println("是否是一个目录："+file.isDirectory());

    }
```
### 3.3 目录操作
**创建单级目录及删除目录**
```java
/**
     * 创建单级目录
     */
    @Test
    public void makeDirTest(){
        File file = new File("classpath://../resource/dir");
        if(file.exists()){
            file.delete();
            System.out.println("此目录存在！已进行删除！");
        }else{
            file.mkdir();
            System.out.println("此目录不存在！创建目录成功！");
        }
    }
```
**创建多级目录及删除目录**
```java
/**
     * 创建多级目录
     */
    @Test
    public void makeDirsTest(){
        File file = new File("classpath://../resource/dir/dir1/dir2");
        if(file.exists()){
            file.delete();
            System.out.println("此目录存在！已进行删除！");
        }else{
            file.mkdirs();
            System.out.println("此目录不存在！创建目录成功！");
        }
    }

```
## 4. IO流原理及其分类
I/O是Input/Output的缩写，I/O技术是非常实用的技术，用于处理数据传输，如读/写文件，网络通讯等。  
Java程序中，对于数据的输入/输出操作以“流（Stream）”的方式进行。  
`java.io`包下提供各种“流”类和接口，用来获取不同种类的数据，并通过方法输入或输出数据。
### 4.1 流的分类
按操作数据单位不同分为：
  - 字节流（8bit）
  - 字符流（按字符）
按数据流向不同分为：
  - 输入流
  - 输出流
按流的角色不同分为：
  - 节点流
  - 处理流/包装流

| 抽象基类 | 字节流       | 字符流 |
|----------|--------------|--------|
| 输入流   | InputStream  | Reader |
| 输出流   | OutputStream | Writer |

> Java的IO流共涉及40多个类，实际上非常有规则，都是以4个抽象基类派生的。  
> 由这四个类派生出来的子类名称，都是以其父类名作为子类名后缀。

![IO](io_map.jpeg)  
![IO](IO.png)
### 4.2 InputStream和OutputStream
![InputStream](InputStream.png)
#### 4.2.1 FileInputStream和FileOutputStream（字节流）
**FileInputStream**
```java
 /**
     * FileInputStream演示
     * 单个字节的读取，效率低
     */
    @Test
    public void fileInputStreamTest(){
        String filePath = "class://../resource/fileinputstream/hello.txt";  // 定义读取的文件位置
        FileInputStream fileInputStream = null;
        try {
            fileInputStream = new FileInputStream(filePath);  // 创建流对象
            int readData = 0;
            while((readData = fileInputStream.read())!=-1){   // read()该输入流读取一个字节的数据。utf8英文字母占1个字节 少数是汉字每个占用3个字节,多数占用4个字节。
                System.out.print(readData);  // 读取出字母对应的ASCII数值
                System.out.print((char)readData);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fileInputStream.close();  // 关闭流
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * FileInputStream演示
     * 字节数组的读取，效率低
     */
    @Test
    public void fileInputStream01Test(){
        String filePath = "class://../resource/fileinputstream/hello.txt";  // 定义读取的文件位置
        FileInputStream fileInputStream = null;
        try {
            fileInputStream = new FileInputStream(filePath);  // 创建流对象
            byte[] bytes = new byte[8];  // 定义字节数组

            int readLength = 0;
            while((readLength = fileInputStream.read(bytes))!=-1){   // read(byte[] b)从该输入流读取最多 b.length个字节的数据为字节数组。utf8英文字母占1个字节 少数是汉字每个占用3个字节,多数占用4个字节。
                System.out.print(readLength);
                System.out.print(new String(bytes,0,readLength));
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fileInputStream.close();  // 关闭流
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

```
**FileOutputStream**
```java
 @Test
    public void fileOutputStreamTest(){
        String filePath = "classpath://../resource/fileoutputstream/hello.txt";
        FileOutputStream fileOutputStream = null;
        try {
            // fileOutputStream = new FileOutputStream(filePath);  // 此种构造器是覆盖内容
            fileOutputStream = new FileOutputStream(new File(filePath), true);
            // 写入一个字节
            // fileOutputStream.write('H');

            // 写入字符串
            //String s = "Hello World!";
            //fileOutputStream.write(s.getBytes());

            // 写入指定字符串
            String s = "Hello World!";
            fileOutputStream.write(s.getBytes(),0,5);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fileOutputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }
```
**FileInputStream和OutputStream实现文件复制**
```java
 /**
     * 将resource文件夹的图片复制到target中
     */
    @Test
    public void copy(){
        FileInputStream fileInputStream = null;
        FileOutputStream fileOutputStream = null;
        try {
            fileInputStream = new FileInputStream("classpath://../resource/source/2017510143717263.png");
            fileOutputStream = new FileOutputStream("classpath://../resource/target/2017510143717263_copy.png");
            byte[] bytes = new byte[1024];
            int readLength = 0;
            while ((readLength = fileInputStream.read(bytes)) !=-1 ){
                fileOutputStream.write(bytes,0,readLength);  // 一定要用这个方法，不然文件数据损坏，提示异常打不开
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fileOutputStream.close();
                fileInputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }
```
### 4.3 Reader和Writer(字符流)
#### 4.3.1 FileReader和FileWriter
**FileReader**
```java
 /**
     * 单个字节读取
     */
    @Test
    public void fileReaderTest(){
        String path = "classpath://../resource/filereader/test.txt";
        FileReader fileReader = null;
        int readData=0;
        try {
            fileReader = new FileReader(path);
            while((readData = fileReader.read())!=-1){
                System.out.print((char)readData);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fileReader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }

    /**
     * 字符数组读取
     */
    @Test
    public void fileReader01Test(){
        String path = "classpath://../resource/filereader/test.txt";
        FileReader fileReader = null;
        int readLen=0;
        char[] buf = new char[8];
        try {
            fileReader = new FileReader(path);
            while((readLen = fileReader.read(buf))!=-1){
                System.out.print(new String(buf,0,readLen));
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fileReader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }

```
**FileWriter**

```java

 @Test
    public void fileWriterTest(){
        String filePath = "classpath://../resource/filewriter/note.txt";
        FileWriter fileWriter = null;
        char[] chars = {'H','E','L','L','O'};
        try {
            fileWriter = new FileWriter(filePath);
            fileWriter.write('H');  // 写入单个字节
            fileWriter.write("Java从入门到入坟！"); // 写入字符串
            fileWriter.write(chars); // 写入字符数组
            fileWriter.write(chars,0,3); // 写入指定长度的字符数组
            fileWriter.write(" 编程入门",0,3); // 写入指定长度的字符串
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fileWriter.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

### 4.4 节点流 和 处理流/包装流
#### 4.4.1 定义
**节点流**
节点流可以从一个特定数据源**读写数据**。如FileReader、FileWriter。  
**处理流（包装流）**
处理流（也叫包装流）是“连接”在已存在的流（节点流或处理流）之上，为程序提供更为强大的读写功能。如BufferedReader、BufferedWriter。 
#### 4.4.2 节点流与处理流的区别与联系
- 节点流是底层流（低级流），直接与数据源相接。
- 处理流包装节点流，既可以消除不同节点流的实现差异，也可以提供方便的方法来完成输入输出。
- 处理流（也叫包装流）对节点流进行包装，使用了修饰器设计模式，不会直接与数据源相连。
#### 4.4.3 处理流的特点
- 性能的提高
主要以增加缓冲的方式来提高输入输出的效率。
- 操作的便捷
处理流可能提供了一系列便捷的方法来一次输入输出大批量的数据，使用更加灵活方便。
#### 4.4.4 处理流/包装流设计模式
**装饰模式（包装模式）**  
装饰模式又名包装（Wrapper）模式。装饰模式以对客户端透明的方式扩展对象的功能，是**继承**关系的一个替代方案。装饰模式通过创建一个包装对象，也就是装饰，来包裹真实的对象。装饰模式以对客户端透明的方式动态地给一个对象附加上更多的责任。换言之，客户端并不会觉得对象在装饰前和装饰后有什么不同。装饰模式可以在不创造更多子类的情况下，将对象的功能加以扩展。装饰模式把客户端的调用委派到被装饰类。装饰模式的关键在于这种扩展是完全透明的

包装模式中有以下几种角色：  
**抽象构件角色（Component）**：给出一个抽象接口，以规范准备接收附加责任的对象。  
**具体构件角色（Concrete Component）**：定义将要接收附加责任的类。  
**装饰角色（Decorator）**：持有一个构件（Component）对象的引用，并定义一个与抽象构件接口一致的接口。  
**具体装饰角色（Concrete Decorator）**：负责给构件对象“贴上”附加的责任。  
![InputStream_uml](InputStream_uml.png)  
示例：  
**抽象构建角色（Component）**
```java
package com.java.demo.designparrern;

public abstract class InputStream_ {

    /**
     * 无参构造器
     */
    public InputStream_(){

    }
    /**
     * 定义抽象读取方法
     * @return
     */
    public abstract int read();

    /**
     * 按字节数组读取的方法
     * @param bytes
     * @return
     */
    public int read(byte[] bytes){
        System.out.println("InputStream_按字节数组读取的方法");
        return -1;
    }
}

```
**具体构件角色（Concrete Component）**
```java
package com.java.demo.designparrern;

import java.io.File;

public class FileInputStream_ extends InputStream_{

    public FileInputStream_(File file){
        System.out.println("打开文件");
    }

    /**
     * 按字节数组读取的方法
     * @return
     */
    public int read() {
        System.out.println("FileInputStream_按字节读取的方法");
        return 0;
    }

    public int read(byte[] bytes){
        System.out.println("FileInputStream_按字节数组读取的方法");
        return 0;
    }
}

```
**装饰角色（Decorator）**
```java
package com.java.demo.designparrern;

public class FilterInputStream_ extends InputStream_{

    protected InputStream_ inputStream_;

    public FilterInputStream_(InputStream_ inputStream_){
        this.inputStream_ = inputStream_;
    }


    public int read() {
        inputStream_.read();
        return 0;
    }

    /**
     * 按字节数组读取
     * @param bytes
     * @return
     */
    public int read(byte[] bytes){
        inputStream_.read(bytes);
        return 0;
    }
}


```
**具体装饰角色（Concrete Decorator）**
```java
package com.java.demo.designparrern;

import java.io.InputStream;

public class BufferedInputStream_ extends FilterInputStream_{

    private InputStream_ getIn(){
        InputStream_ in = null;
        return  in = inputStream_;
    }

    public BufferedInputStream_(InputStream_ inputStream_) {
        super(inputStream_);
    }

    public int read() {
        System.out.println("BufferedInputStream_按字节读取方法");
        inputStream_.read();
        return 0;
    }

    public int read(byte bytes[]){
        System.out.println("BufferedInputStream_按字节数组读取方法");
        inputStream_.read(bytes);
        return 0;
    }

}

```
**测试**
```java

package com.java.demo.designparrern;

import org.junit.Test;

import java.io.File;

public class Test_ {

    @Test
    public void test(){
        BufferedInputStream_ bufferedInputStream_ = new BufferedInputStream_(new FileInputStream_(new File("classpath://../resource/file01/test01.txt")));
        bufferedInputStream_.read();

        BufferedInputStream_ bufferedInputStream_1 = new BufferedInputStream_(new ByteArrayInputStream_(new byte[8]));
        bufferedInputStream_1.read();
    }
}

```
### 4.5 BufferedReader和BufferedWriter
**BufferedReader**
```java
@Test
    public void bufferedReaderTest() throws IOException {
        String filePath = "classpath://../resource/fileReader/test.txt";
        BufferedReader bufferedReader = new BufferedReader(new FileReader(filePath));
        String readLine = null;
        while((readLine = bufferedReader.readLine())!=null){
            System.out.println(readLine);
        }
        bufferedReader.close();
    }
```
**BufferedWriter**
```java
@Test
    public void bufferedWriterTest() throws IOException {
        String filePath = "classpath://../resource/filewriter/new.txt";
        BufferedWriter bufferedWriter = new BufferedWriter(new FileWriter(filePath));
        bufferedWriter.write("Hello World!");
        bufferedWriter.newLine();
        bufferedWriter.write("你好 世界！");
        bufferedWriter.close();
    }
```
**文件复制**
```java
@Test
    public void bufferedCopy(){
        String sourcePath = "classpath://../resource/source/buffered.txt";
        String targetPath = "classpath://../resource/target/buffered_copy.txt";
        BufferedReader bufferedReader = null;
        BufferedWriter bufferedWriter = null;
        String readLine;
        try {
            bufferedReader = new BufferedReader(new FileReader(sourcePath));
            bufferedWriter = new BufferedWriter(new FileWriter(targetPath));
            while ((readLine=bufferedReader.readLine())!=null){
                bufferedWriter.write(readLine);
                bufferedWriter.newLine();
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (null != bufferedReader)
                    bufferedReader.close();
                if (null != bufferedWriter)
                    bufferedWriter.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```
### 4.6 BufferedInputStream和BufferedOutputStream
**二进制文件Copy**
```java
  @Test
    public void bufferedStreamTest(){
        String sourcePath = "classpath://../resource/source/2017510143811976.png";
        String targetPath = "classpath://../resource/target/2017510143811976_copy.png";
        BufferedInputStream bufferedInputStream = null;
        BufferedOutputStream bufferedOutputStream = null;
        byte[] bytes = new byte[1024];
        int readData;
        try {
            bufferedInputStream = new BufferedInputStream(new FileInputStream(sourcePath));
            bufferedOutputStream = new BufferedOutputStream(new FileOutputStream(targetPath));
            while((readData = bufferedInputStream.read(bytes))!=-1){
                bufferedOutputStream.write(bytes,0,bytes.length);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (bufferedInputStream!=null)
                bufferedInputStream.close();
                if (bufferedOutputStream!=null)
                bufferedOutputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
    }
```
### 4.7 ObjectInputStream和ObjectOutputStream
> 将`int num = 100`这个`int`数据保存到文件中，注意不是`100`这个数字，而是`int 100`,并且能够从文件中直接恢复。  
> 将`Dog dog = new Dog("小黄",3)`这个对象保存在文件中并且能够从文件恢复。  

上面的要求，就是能够将基本数据类型或者对象数据类型惊醒序列化和反序列化。
#### 4.7.1 序列化和反序列化
**序列化**就是在保存数据时，保存**数据的值**和**数据类型**。  
**反序列化**就是在恢复数据是，恢复**数据的值**和**数据类型**。
需要让某个对象支持序列化机制，则必须让其类是可序列化的，为了让某个类是可序列化的，该类必须实现如下两个接口之一：
- Serializable
- Externalizable  

示例：  
ObjectOutputStream
```java
package com.java.demo.objectstream;

import org.junit.Test;

import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.io.Serializable;

public class ObjectOutputSreamTest {

    @Test
    public void ObjectOutputStreamTest() throws IOException {
        String filePath = "classpath://../resource/objectstream/data.dat";
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(filePath));
        objectOutputStream.writeInt(100);  // int -> Integer  实现了 Serializable
        objectOutputStream.writeBoolean(true); // boolean -> Boolean  实现了 Serializable
        objectOutputStream.writeUTF("Java IO流"); // String 实现了 Serializable
        objectOutputStream.writeDouble(100.00);  // double -> Double  实现了 Serializable
        objectOutputStream.writeChar('a'); // char -> Character 实现了 Serializable

        Dog dog = new Dog("旺财", 6);
        objectOutputStream.writeObject(dog);

        objectOutputStream.close();
    }
}
class Dog implements Serializable {

    private String name;

    private Integer age;

    public Dog(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public void desc(){
        System.out.println("我是狗！汪汪汪！");
    }
}
```
ObjectInputStream
```java
package com.java.demo.objectstream;

import org.junit.Test;

import java.io.FileInputStream;
import java.io.IOException;
import java.io.ObjectInputStream;

public class ObjectInputStreamTest {

    @Test
    public void objetcInputStreamTest() throws IOException, ClassNotFoundException {
        String filePath = "classpath://../resource/objectstream/data.dat";
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(filePath));
        // 反序列化必须与粗劣话的顺序一致
        System.out.println(objectInputStream.readInt());
        System.out.println(objectInputStream.readBoolean());
        System.out.println(objectInputStream.readUTF());
        System.out.println(objectInputStream.readDouble());
        System.out.println(objectInputStream.readChar());

        Object o = objectInputStream.readObject();
        System.out.println(o);
        objectInputStream.close();
        Dog dog = (Dog)o; // 同一个包下的同一个类才可以强转，不然需要声明称public
        dog.desc();
    }

}

```
#### 4.7.2 对象处理流使用细节
注意事项和细节说明
- 读写顺序需要一致；
- 要求实现序列化或发序列化对象实现Serializable接口；
- 序列化的类中建议添加SerialVersionUID,为了提高版本的兼容性；
- 序列化对象时，默认将里面所有属性进行序列化，但是除了static或transient修饰的成员；
- 序列化对象时，要求里面属性的类型也需要实现序列化接口；
- 序列化具备科技成型，也就是如果某类已经实现了序列化，则其所有的子类也已经默认实现了序列化。

### 4.8 标准输入输出流
```java
@Test
    public void inputAndOutputTest(){
        // 标准输入流  默认设备  键盘
        // 编译类型
        InputStream in = System.in;
        // 运行类型
        System.out.println(System.in.getClass());
        // 标准输出流  默认设备  显示器
        // 编译类型
        PrintStream out = System.out;
        // 运行类型
        System.out.println(System.out.getClass());
    }
```
### 4.9 转换流(InputStreamReader和OutputStreamWriter)
创建一个.txt文件，内容如下：
```txt
Hello World!
您好 世界！
```
使用`BufferedReader`读取并打印
```java
// 文件默认读取编码是UTF-8
    @Test
    public void bufferedReaderTest() throws IOException {
        String filePath = "classpath://../resource/fileReader/test.txt";
        BufferedReader bufferedReader = new BufferedReader(new FileReader(filePath));
        String readLine = null;
        while((readLine = bufferedReader.readLine())!=null){
            System.out.println(readLine);
        }
        bufferedReader.close();
    }
```
运行输出结果：
```
Hello World!
您好 世界！

Process finished with exit code 0
```
修改文件的编码格式为`GBK`，运行结果如下：
```
Hello World!
���� ���磡

Process finished with exit code 0
```
![乱码](乱码.jpg)  
#### 4.9.1 使用InputStreamReader读取文件  
![InputStreamReader](InputStreamReader.png)
```java
@Test
    public void inputStringReaderTest() throws IOException {
        String filePath = "class://../resource/transformation/codequestion.txt";
        InputStreamReader inputStreamReader = new InputStreamReader(new FileInputStream(filePath), "GBK");
        // 使用InputStreamReader
//        int read = 0;
//        while((read = inputStreamReader.read())!=-1){
//            System.out.print((char)read);
//        }
//        inputStreamReader.close();
        // BufferedReader
        BufferedReader bufferedReader = new BufferedReader(inputStreamReader);
        String readData;
        while((readData = bufferedReader.readLine())!=null){
            System.out.println(readData);
        }
        bufferedReader.close();
    }
```
执行结果如下：
```
Hello World!
您好 世界！

Process finished with exit code 0
```
#### 4.9.2 使用OutputStreamReader读取文件 
```java
 @Test
    public void outputStreamWriterTest() throws IOException {
        String filePath = "classpath://../resource/transformation/hello.txt";
        OutputStreamWriter outputStreamWriter = new OutputStreamWriter(new FileOutputStream(filePath),"UTF-8");
        outputStreamWriter.write("我们一起去看海！");
        outputStreamWriter.close();
    }
```
### 5.PrintStream和PrintWiter
打印流只有输出流没有输入流。  
![PrintStream](PrintStream.png)
```java
@Test
    public void printStreamTest() throws IOException {
        PrintStream printStream = System.out;
        printStream.print("您好，世界！");
        // print方法底层调的write()方法
        printStream.write("哈哈哈".getBytes());
        printStream.close();

        // 修改打印流输出的位置/设备
        System.setOut(new PrintStream("classpath://../resource/print/print.txt"));
        System.out.print("您好,中国！");

    }
```
![PrintWriter](PrintWriter.png)  
```java
@Test
    public void printWriterTest() throws FileNotFoundException {
        PrintWriter printWriter = new PrintWriter(System.out);
        printWriter.println("终于干完了！");
        printWriter.close();

        PrintWriter printWriter1 = new PrintWriter("classpath://../resource/transformation/printwriter.txt");
        printWriter1.println("终于干完了！");
        printWriter1.close();
    }
```




---
参考资料：  
[java基础之IO流（设计模式）](https://www.jianshu.com/p/613ee60e08b4)  
