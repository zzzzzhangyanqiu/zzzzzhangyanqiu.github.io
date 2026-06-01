---
title: Java I/O
tags:
  - Java
date: 2018-05-13
categories:
 - Java
---

# Java IO

> 通过数据流，序列化和文件系统提供系统输入和输出

&emsp;由于存在大量不同的设计方案，所以该任务的困难性是很容易证明的。其中最大的挑战似乎是如何覆盖所有可能的因素。不仅有三种不同的种类的IO需要考虑（文件、控制台、网络连接），而且需要通过大量不同的方式与它们通信（顺序、随机访问、二进制、字符、按行、按字等等）。IO系统整体可归纳为输入和输出两个部分，其中又分别分为字节流(IO1.0中)和字符流(IO1.1中)，字节流分为InputStream(输入流)和OutputStream(输出流)。字符流分为Reader(读取流)和Writer(写入流)。由于I/O系统实在太过于庞大，故此篇文章也只是对Java中的I/O做一个简单的了解。
## 字节流

#### InputStream

>此类是以字节为单位的输入流，是输入字节流的所有类的父类。


&emsp;所有自InputStream衍生的类都拥有read()方法，用于读取单个字节或字节数组，相同的，所有自OutputStream衍生的类都拥有write()方法，用于写入单个字节或者字节数组。然而，我们并不会用到这些方法，它们的存在，是为了更复杂的类会用到这些方法，以便提供一个更便利的接口。

&emsp;InputStream的作用是标志那些从不同来源产生的输入流，这些来源地包括(每种类型都有一个对应的子类):

1. 字节数组。
2. 文件。
3. String对象。
4. "管道"，类似于生活中的管道:在一端输入，并在另一端输出。
5. 多个输入流，以便将它们合并成为一个输入流。
6. 其它输入流，如Internet等。



该类的常用子类如下：

- ByteArrayInputStream：包含一个内部缓冲区，其中包含从流中读取到的字节，因为包含缓冲区，所以可以在关闭流后使用此类的方法，不会生成IOException。
- FileInputStream：从文件系统的文件中获取输入字节，FileInputStream用于读取原始字节流，为了读取字符流，请考虑使用FileReader。
- FilterInputStream：使用一些其它输入流作为数据源，可能会做一些转换或者其它操作。
- ObjectInputStream：将先前使用ObjectOutputStream编写的原始数据和对象进行反序列化。
- PipedInputStream：配合PipedOutputStream进行多线程管道流的数据通信。
- SequenceInputStream：代表其它输入流的逻辑连接，从第一个输入流开始读到结束，然后从第二个输入流开始读到结束，直到最后一个结束。
- StringBufferInputStream：将String字符串作为数据源创建输入流。

FilterInputStream有两个常用的子类:
- BufferedInputStream：为另一个输入流增加功能-缓冲输入,mark(标记在缓冲区的位置)和reset(重置缓冲区为标记位置)的能力。
- DataInputStream：用来装饰其它输入流，它“允许应用程序以与机器无关方式从底层输入流中读取基本 Java 数据类型”。

#### OutputStream

> 用来将字节输出到特定的接收器中，是所有字节输出流的父类。所有OutputStream衍生的子类必须提供至少一种写入字节流的方法。

该类的常用子类如下:

- ByteArrayOutputStream：将数据写入字节数组，数据写入时，缓冲区会自动增长。因为包含缓冲区，所以可以在关闭流后使用此类的方法，不会生成IOException。
- FileOutputStream：将数据流写入File或FileDescriptor，FileOutputStream用于写入原始字节流，为了写入字符流，请考虑使用FileWriter。
- FilterOutputStream：使用一些其它输出流作为数据源，可能会做一些转换或者其它操作。
- ObjectOutputStream：将对象或图形写入输出流，可以使用ObjectInputStream重新读取。对象的持久存储可以通过流的文件来完成，如果是网络套接字流，则可以在网络另一端再次构建对象。只有实现java.io.Serializable接口的对象才能被写入流。每个可序列化对象的类都被编码，包括类的类名和签名，对象的字段和数组的值以及从初始对象引用的任何其他对象。
- PipedOutputStream：配置PipedInputStream进行多线程管道流的数据通信。

FilterOutStream:
- BufferedOutputStream：该类实现缓冲输出流。
- DataOutputStream：用来写入DataInputStream读取到的数据。
- PrintStream：为其它输出流提供打印功能。

#### 实践

&emsp;正如前文所说，一个IO系统是非常复杂的，因为要考虑多种情况，如果使用继承的方式，那类的数量会非常非常多，也不方便使用，那这个时候怎么办呢，IO系统的设计人员用到了一种设计模式:装饰模式。

&emsp;装饰模式指的是在不必改变原类文件和使用继承的情况下，动态地扩展一个对象的功能。它是通过创建一个包装对象，也就是装饰来包裹真实的对象。以下举例说明:

1. 从文件系统中读取文件:

    ```java
    String fileName = "E:\\test.txt";
    DataInputStream in = null;
    try{
        in = new DataInputStream(new FileInputStream(fileName));
        String s;
        StringBuilder s1 = new StringBuilder();
        while ((s = in.readLine()) != null){
            s1.append(s).append("\n");
        }
        System.out.println(s1);
    }catch (Exception e){
        e.printStackTrace();
    }finally {
        if(in != null){
            try {
                in.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    ```

    

2. 如果你想给文件读取加入缓冲的功能，也只需将FileInputStream"包装"成缓冲流即可:

    ```java
    in = new DataInputStream(new BufferedInputStream(new FileInputStream(fileName)));
    ```

相同的，如果你想写入文件:

1. 普通文件写入:

    ```java
    String fileName = "E:\\test.txt";
    DataOutputStream op = null;
    try{
        op = new DataOutputStream(new FileOutputStream(fileName));
        op.writeBytes("Hello!This is Java I/O.");
    }catch (Exception e){
        e.printStackTrace();
    }finally {
        if(op != null){
            try {
                op.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    ```

    

2. 加入缓冲功能:

    ```java
    op = new DataOutputStream(new BufferedOutputStream(new FileOutputStream(fileOut)));
    ```

这里只做简单介绍，更多信息请查看JavaAPI文档。

## 字符流
&emsp;为了满足国际化的需求。老式IO流层次结构只支持8位字节流，不能很好地控制16位Unicode字符。由于Unicode主要面向的是国际化支持（Java内含的char是16位的Unicode），所以添加了Reader和Writer层次，以提供对所有IO操作中的Unicode的支持。除此之外，新库也对速度进行了优化，可比旧库更快地运行。
&emsp;几乎IO1.0中的每个类都有对应的IO1.1的类，如下:

| 功能         | IO1.0 | IO1.1 |
|---|
| 输入流       | InputStream              | Reader   converter:InputStreamReader|
| 输出流       | OutputStream             | Writer   converter:OutputStreamWriter|
| 文件输入流  | FileInputStream           | FileReader |
| 文件输出流  | FileOutputStream          | FileWriter |
| 字符串输入流| StringBufferInputStream   | StringReader |
| 字符串输出流| none                      | StringWriter |
| 数组输入流  | ByteArrayInputStream      | CharArrayReader |
| 数组输出流  | ByteArrayOutputStream     | CharArrayWriter |
| 管道输出流  | PipedInputStream          | PipedReader |
| 管道输出流  | PipedOutputStream         | PipedWriter |


#### 实例

读取文件:

```java
	String fileName = "E:\\test.txt";
	BufferedReader br = null;
	try{
		br = new BufferedReader(new FileReader(fileName));
        String s;
        StringBuilder s1 = new StringBuilder();
        while ((s = br.readLine()) != null){
            s1.append(s).append("\n");
        }
        System.out.println(s1);
	}catch (Exception e){
        e.printStackTrace();
    }finally {
        if(br != null){
            try {
                br.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
	}
```

写入文件:

```java
	String fileName = "E:\\test.txt";
	BufferedWriter bw = null;
	try{
		bw = new BufferedWriter(new FileWriter(fileOut));
        bw.write("This is Java IO 1.1");
	}catch (Exception e){
        e.printStackTrace();
    }finally {
        if(bw != null){
            try {
                bw.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
	}
```

## 压缩

&emsp;Java中提供了文件的压缩与解压类，此处主要介绍GZIP和Zip两种压缩方法。

#### 使用GZIP进行简单压缩

```java
	String fileName = "E:\\test.txt";
    BufferedReader br = null;
    BufferedOutputStream bos = null;
    try{
        br = new BufferedReader(new FileReader(fileName));
        bos = new BufferedOutputStream(new GZIPOutputStream(new FileOutputStream("E:\\test.gz")));
        int c;
        while ((c = br.read()) != -1){
            bos.write(c);
        }
    }catch (Exception e){
        e.printStackTrace();
    }finally {
        if(br != null){
            try {
                br.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if(bos != null){
            try {
                bos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }
```

#### 使用GZIP进行解压读取

```java
	String fileName = "E:\\test.txt";
    BufferedInputStream bis = null;
    try{
        int c;
        String content = "";
        bis = new BufferedInputStream(new GZIPInputStream(new FileInputStream("E:\\test.gz")));
        while ((c = bis.read()) != -1){
            content += (char)c;
        }
        System.out.println(content);
    }catch (Exception e){
        e.printStackTrace();
    }finally {
        if(bis != null){
            try {
                bis.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }
```

#### 使用ZIP压缩多个文件

```java
	String [] fileName = new String[]{"E:\\test.txt","E:\\test1.txt","E:\\test2.txt"};
    ZipOutputStream zs = null;
    CheckedOutputStream co = null;
    FileOutputStream fo = null;
    FileInputStream fi = null;
    try{
        fo = new FileOutputStream("E:\\test.zip");
        co = new CheckedOutputStream(fo,new Adler32());
        zs = new ZipOutputStream(new BufferedOutputStream(co));
        for (String s : fileName) {
            fi = new FileInputStream(s);
            BufferedReader br = new BufferedReader(new InputStreamReader(fi,"gb2312")); //解决中文乱码问题
            zs.putNextEntry(new ZipEntry(s));
            System.out.println("开始写入文件:"+s);
            String content;
            while ((content = br.readLine()) != null){
                //手动添加换行
                content += "\n\r";
                zs.write(content.getBytes());
            }
            zs.flush();
            fi.close();
            br.close();
            zs.closeEntry();
        }
    }catch (Exception e){
        e.printStackTrace();
    }finally {
        if(zs != null){
            try {
                zs.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if(co != null){
            try {
                co.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if(fo != null){
            try {
                fo.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

#### 使用Zip解压文件

```java
	String fileName = "E:\\test.zip";
    ZipInputStream zi = null;
    CheckedInputStream ci = null;
    FileInputStream zip = null;
    BufferedWriter fw;
    try{
        zip = new FileInputStream(fileName);
        //校验和
        ci = new CheckedInputStream(zip,new Adler32());
        zi = new ZipInputStream(new BufferedInputStream(ci));
        ZipEntry nextEntry;
        while((nextEntry = zi.getNextEntry()) != null){
            System.out.println("读取文件"+nextEntry.getName());
            fw = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(nextEntry.getName())));
            int c;
            while((c = zi.read()) != -1){
                fw.write(c);
            }
            fw.close();
        }
        zi.closeEntry();
    }catch (Exception e){
        e.printStackTrace();
    }finally {
        if(zip != null){
            try {
                zip.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if(ci != null){
            try {
                ci.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if(zi != null){
            try {
                zi.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

此方法会发生中文乱码，暂时没有找到解决方案，不过还有另一种方法可解决乱码问题:

```java
	String fileName = "E:\\test.zip";
    BufferedWriter bw;
    BufferedReader br;
    try{
        ZipFile zipFile = new ZipFile(fileName);
        Enumeration entries = zipFile.entries();
        while(entries.hasMoreElements()){
            ZipEntry zipEntry = (ZipEntry) entries.nextElement();
            System.out.println("开始处理:"+zipEntry.getName());
            bw = new BufferedWriter(new FileWriter(zipEntry.getName()));
            br = new BufferedReader(new InputStreamReader(zipFile.getInputStream(zipEntry),"utf-8"));
            int s;
            while ((s = br.read()) != -1){
                bw.write(s);
            }
            br.close();
            bw.close();
        }
    }catch (Exception e){
        e.printStackTrace();
    }
```




## 对象持久化
```
`{% blockquote David Levithan, Wide Awake %}Do not just seek happiness for yourself. Seek happiness for all. Through kindness. Through mercy.{% endblockquote %}`
```

#### 持久化
Data.java:

```java
public class Data implements Serializable {
    private String name;

    public Data(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Data{" +
                "name='" + name + '\'' +
                '}';
    }
}
```

Test.java

```java
Data data = new Data("测试对象持久化");
System.out.println(data);
ObjectOutputStream op;
try{
    op = new ObjectOutputStream(new FileOutputStream("E:\\data.out"));
    op.writeObject(data);
}catch (Exception e){
    e.printStackTrace();
}
```

#### 从持久化中恢复

```java
ObjectInputStream oi;
try{
    oi = new ObjectInputStream(new FileInputStream("E:\\data.out"));
    Data data = (Data) oi.readObject();
    System.out.println(data);
}catch (Exception e){
    e.printStackTrace();
}
```

&emsp;如果有字段不想被实例化(如密码等),可以对字段使用transient关键字:

```java
private transient String passWord;
```

#### 总结
&emsp;持久化的作用不止于此，比如Java远程方法调用(RMI)等。了解更多请查看API文档