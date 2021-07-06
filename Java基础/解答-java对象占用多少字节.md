## Java 对象占用多少字节

### 一、预备知识：

#### Java 对象模型

HotSpot JVM 使用名为 oops (Ordinary Object Pointers)  的数据结构来表示对象，对象在内存中分3部分：

- Header: 对象头， 分3部分，mark word、元数据指针、数组长度
  - mark word : 存储 hashcode, locking pattern, locking information, and GC metadata(对象的引用计数数量，便于GC回收)，**这部分在 64 位操作系统下占 8 字节，32 位操作系统下占 4 字节**。 详情可查看：https://www.baeldung.com/java-memory-layout
  - kclass 指针：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪一个类的实例。
    这部分就涉及到指针压缩的概念，**在开启指针压缩的状况下占 4 字节，未开启状况下占 8 kb。**
  - 数组长度：这部分只有是数组对象才有，**若是是非数组对象就没这部分。这部分占 4 kb。**
- Instance Data: 实例数据
- Alignment Padding: 对象对齐填充。Java 对象的大小默认是按照 8 字节对齐，也就是说 Java 对象的大小必须是 8 字节的倍数。若是算到最后不够 8 字节的话，那么就会进行对齐填充。

#### 指针压缩

JVM 为了节省内存，如果 heap size 小于 32GB，JVM会自动开启指针压缩。大于32GB会关闭指针压缩。可以用 *-XX:-UseCompressedOops* tuning flag 强行开启指针压缩

#### 基本类型占用空间

| 类型              | 占用空间(byte)              |
| ----------------- | --------------------------- |
| boolean           | 1                           |
| byte              | 1                           |
| short             | 2                           |
| char              | 2                           |
| int               | 4                           |
| float             | 4                           |
| long              | 8                           |
| double            | 8                           |
| object references | 4，如果未开启指针压缩则为 8 |



## 二、分析对象大小

**以下分析基于 jdk11，64位操作系统处理。**

其实有3个不同的指标来分析对象大小。

- shallow size: 指对象自身占用的内存, 不包括引用对象的实际大小，引用对象只计算引用对象指针大小。
- ratained size: 当对象自身被 GC 回收时, 对象自身 shallow size 加上其引用对象的 shallow size。
  - 只有加上能同时被 GC 回收的引用对象。
  - 怎么判断引用对象能否被回收？
  - ![引用对象图](https://www.yourkit.com/docs/java/help/retained_objects.gif)
  - 如上图，假设所有对象的 Shallow size 为 1 字节,
    - 如果要回收 obj1，obj1 引用了 obj2，obj2 又同时引用了 obj4 和 obj3
    - 由于 obj3 同时被Gc Roots 引用，所以不能加上 obj3 的 Shallow size
    - 所以 obj1 的 ratained size:  obj1 + obj2 + obj4 ，3个类的 Shallow size 总和为 3 字节。
- deep size: 与 shallow size 相反，不但要计算实例自身占用的空间，还要计算引用对象的大小，需要递归计算。



接下来会用代码实际分析对象大小

代码依赖：

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.10</version>
</dependency>
```

基础代码类：

```java
public class Course {
    private String name;

    public Course(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
}
```

大小大小

#### Shallow Size

计算公式：**对象头 + 实例数据 + 对齐填充字节**

代码：

```java
public class ObjectsSizeMain {
    public static void main(String[] args) {
        String ds = "Data Structures";
        Course course = new Course(ds);
        System.out.println("The shallow size is: " + VM.current().sizeOf(course));
        System.out.println(ClassLayout.parseInstance(course).toPrintable());
    }
}
```

实例数据：Course 类只有一个 String 类的类属性 name ，所以占用 4 字节。

对象头：Couse 类非数组对象，所以占用 12 字节

对齐填充字节： 由于对象头+实例数据=16字节，所以不需要填充，填充字节为 0。

**所以 Course 类实例 shallow size 是16字节。**

实际输出：

```java
The shallow size is: 16
org.example.objects_size.Course object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4   java.lang.String Course.name                               (object)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```



#### Ratained Size

计算公式：**对象自身的 Shallow Size + 引用对象的 Shallow Size**

代码：

```java
public class ObjectsSizeMain {
    public static void main(String[] args) {
        String ds = "Data Structures";
        Course course = new Course(ds);
        System.out.println("course size is: " + VM.current().sizeOf(course));
        System.out.println("name size is: " + VM.current().sizeOf(ds));
        System.out.println(ClassLayout.parseInstance(ds).toPrintable());
    }
}
```

Course 自身的 shallow size: 16 字节

引用对象的 shallow size：Course 的引用对象为 name， 是个 String 类，可以用`ClassLayout.parseInstance(course.getName()).toPrintable()`分析大小。

Course.name 的对象大小：

```
java.lang.String object internals:
 OFFSET  SIZE     TYPE DESCRIPTION                               VALUE
      0     4          (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4          (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4          (object header)                           08 18 00 00 (00001000 00011000 00000000 00000000) (6152)
     12     4   byte[] String.value                              [68, 97, 116, 97, 32, 83, 116, 114, 117, 99, 116, 117, 114, 101, 115]
     16     4      int String.hash                               0
     20     1     byte String.coder                              0
     21     3          (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 3 bytes external = 3 bytes total
```

可以看到 String 对象大小为 24 字节。其中请求头 12 字节 ，byte[] 数据 4 字节，hash code 4 字节，byte coder 1 字节，对齐填充 3 字节。

**所以 Course 类的 ratained size = 16+24=40 字节。**



#### Deep Size

代码：

```java
public class ObjectsSizeMain {
    public static void main(String[] args) {
        String ds = "Data Structures";
        Course course = new Course(ds);
        System.out.println("course size is: " + VM.current().sizeOf(course));
        System.out.println("name size is: " + VM.current().sizeOf(ds));
        System.out.println(ClassLayout.parseInstance(ds.getBytes()).toPrintable());
    }
}
```

由上已知，Course 类对象大小是 16 字节，String 类对象大小是 24 字节。只要在加上 String 对象实际数据大小就是 Deep Size。

可以通过 `ClassLayout.parseInstance(ds.getBytes()).toPrintable()`获取 String 对象实际数据大小。

输出：

```
[B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           20 08 00 00 (00100000 00001000 00000000 00000000) (2080)
     12     4        (object header)                           0f 00 00 00 (00001111 00000000 00000000 00000000) (15)
     16    15   byte [B.<elements>                             N/A
     31     1        (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 0 bytes internal + 1 bytes external = 1 bytes total
```

 String 对象实际数据大小为 32 字节。其中请求头为 16 字节（mark word 8 字节+ kclass 指针 4 字节 + 数据长度 4 字节），实际数据为 15 字节，对齐填充 1 字节。

**所以 Course 类的 deep size 为 16+24+32=72 字节。**

**注：jdk8 下 String 类的 value 为 char[]，所以要计算 char[] 的大小，`ClassLayout.parseInstance(ds.toCharArray()).toPrintable();`, 由于 char array 为 48 字节，所以jdk8 下 deep size 为 16+24+48=88 字节**