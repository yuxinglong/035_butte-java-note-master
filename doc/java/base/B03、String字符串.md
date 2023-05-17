# 一、String类简介

## 1、基础简介

字符串是一个特殊的数据类型，属于引用类型。String类在Java中使用关键字final修饰，所以这个类是不可以继承扩展和修改它的方法。String类用处极广泛，在对String对象进行初始化时，和基本类型的包装器类型一样，可以不使用new关键字构造对象。(是真的妖娆...)

## 2、类构造和方法

- String类结构

特点：final关键字修饰，实现Serializable序列化接口，Comparable比较接口，和CharSequence字符序列接口。

```java
final class String
    implements java.io.Serializable,
    Comparable<String>, CharSequence
```

- 声明方式

两种方式，常量和创建对象。

```java
String var1 = "cicada" ;           
String var2 = new String("smile") ;
```

var1：声明的是一个常量，显然是放在常量池中。

var2：创建字符串对象，对象存放在堆内存中。

# 二、常见应用

## 1、比较判断

常量池用来存放常量；堆内存用来存放new出来的引用对象。

```java
public class String02 {
    public static void main(String[] args) {
        String var1 = "cicada" ;
        String var2 = "cicada" ;
        // true;true
        System.out.println((var1==var2)+";"+var1.equals(var2));
        String var3 = new String("cicada");
        String var4 = new String("cicada");
        // false;true
        System.out.println((var3==var4)+";"+var3.equals(var4));
        // false;true
        System.out.println((var1==var4)+";"+var2.equals(var4));
        String var5 = "ci"+"cada";
        // true;true
        System.out.println((var1==var5)+";"+var5.equals(var4));
        String var6 = new String02().getVar6 () ;
        // true;true
        System.out.println((var1==var6)+";"+var6.equals(var4));
    }
    public String getVar6 (){
        return "cicada" ;
    }
}
```

`==`：对于基本类型，比较的是值，对于引用类型，比较的是地址的值；

`equals`：该方法源自Object中一个最基础的通用方法，在Object的方法中使用`==`判断地址的值，只是到了String类中进行了重写，用于字符内容的比较，该方法在继承关系中的变化，追踪JDK源码，变化非常清楚。

## 2、编码解析

字符串在String内部是通过一个char[]数组表示，Unicode统一的编码表示的字符，char类型的字符编码由此来。

- 构造源码

这里看下构造方法就会明白上面的概念逻辑。

```java
private final char value[];
public String() {this.value = "".value;}
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
```

- 编码转换

不同的国家和地区，使用的编码可能是不一样的，互联网中有UTF8编码又是最常用，一次在程序开发中，经常需要编码之间转换。

```java
public class String03 {
    public static void main(String[] args) throws Exception {
        String value = "Hello,知了";
        // UTF-8
        byte[] defaultCharset = value.getBytes(Charset.defaultCharset());
        System.out.println(Arrays.toString(defaultCharset));
        System.out.println(new String(defaultCharset,"UTF-8"));
        // GBK
        byte[] gbkCharset = value.getBytes("GBK");
        System.out.println(Arrays.toString(gbkCharset));
        System.out.println(new String(gbkCharset,"GBK"));
        // ISO-8859-1：表示的字符范围很窄,无法表示中文字符，转换之后无法解码
        byte[] isoCharset = value.getBytes("ISO8859-1");
        System.out.println(Arrays.toString(isoCharset));
        System.out.println(new String(isoCharset,"ISO8859-1"));
        // UTF-16
        byte[] utf16Charset = value.getBytes("UTF-16");
        System.out.println(Arrays.toString(utf16Charset));
        System.out.println(new String(utf16Charset,"UTF-16"));
    }
}
```

两个基础概念：

`编码Encode`：信息按照规则从一种形转换为另一种形式的过程,简称编码;

`解码Decode`：解码就是编码的逆向过程。

## 3、格式化操作

在日常开发中，字符串的格式不会都满足业务要求，通常就需要进行指定格式化操作。

```java
public class String04 {
    public static void main(String[] args) {
        // 指定位置拼接字符串
        String var1 = formatStr("cicada","smile");
        System.out.println("var1="+var1);
        // 格式化日期：2020-03-07
        SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd");
        Date date = new Date() ;
        System.out.println(format.format(date));
        // 浮点数：此处会四舍五入
        double num = 3.14159;
        System.out.print(String.format("浮点类型：%.3f %n", num));
    }
    public static String formatStr (String ...var){
        return String.format("key:%s:route:%s",var);
    }
}
```

上面案例演示应用场景：Redis缓存Key生成，日期类型转换，超长浮点数的截取。

## 4、形参传递问题

String对象形参传递到方法里的时候,实际上传递的是引用的拷贝。

```java
public class String05 {
    String var1 = "hello" ;
    int[] intArr = {1,2,3};
    public static void main(String[] args) {
        String05 objStr = new String05() ;
        objStr.change(objStr.var1,objStr.intArr);
        // hello  4
        System.out.println(objStr.var1);
        System.out.println(objStr.intArr[2]);
    }
    public void change (String var1,int[] intArr){
        var1 = "world" ;
        intArr[2] = 4 ;
    }
}
```

案例中改变的是var1引用的拷贝,方法结束执行结束，形参var1被销毁, 原对象的引用保持不变。数组作为参数传递时传递是数组在内存中的地址值，这样直接找到数组在内存中的位置。

## 5、String工具类

字符串的处理在系统开发中十分的常见，通常会提供一个工具类统一处理，可以基于一个框架中的工具类二次封装，也可以全部自行封装。

```java
class StringUtil {
    private StringUtil(){}
    public static String getUUid (){
        return UUID.randomUUID().toString().replace("-","");
    }
}
```

上面是字符串工具类最基础的一个。不同框架中自带的工具类也不错。

```
org.apache.commons.lang3.StringUtils
org.springframework.util.StringUtils
com.alibaba.druid.util.StringUtils
```

这里推荐第一个，也可以把自定义的工具类继承该工具类，提供更丰富的公共方法。

`絮叨一句`：代码整洁之道的基础，就是有一颗《偷懒》的心，花点心思该封装的封装，该删除的删除。

# 三、扩展API

## 1、StringBuffer类

字符串修改拼接常用的API,内部的实现过程和String类似。

```java
public class String07 {
    public static void main(String[] args) {
        StringBuffer var = new StringBuffer(2) ;
        var.append("what");
        var.append("when");
        System.out.println(var);
    }
}
```

看到上面几行代码的反应，基本能反应编程的年龄：

`一年`：API是这样用的，没毛病；

`三年`：StringBuffer是线程安全的，效率相对偏低；

`五年`：默认字符数组大小是16，这里自定义字符数组的大小，如果长度不够需要扩容，所以要预估一下字符串的可能大小，减小消耗；

`絮叨一句`：Java中许多容器对象的大小默认是16，且具备动态扩容机制，这就是传说中的编程思想，在开发中照葫芦画瓢的写两段，这就是格调。

## 2、StringBuilder类

这个类出现比StringBuffer要晚很多，从JDK1.5才开始出现。

```java
public class String08 {
    public static void main(String[] args) {
        StringBuilder var = new StringBuilder() ;
        var.append("how").append("what") ;
        System.out.println(var);
    }
}
```

用法和StringBuffer差不多，不过是非线程安全操作，效率自然要高。

`补刀一句`：对于线程安全和操作和非安全操作，还有初始容量和扩容这种逻辑，都可以在源码中查看，这是进阶程序员的必备意识。

## 3、再看传参问题

这里原理解释同上，根本逻辑是一致的。

```
public class String09 {
    public static void main(String[] args) {
        String var1 = new String("A");
        String var2 = new String("B");
        StringBuffer var3 = new StringBuffer("C");
        StringBuffer var4 = new StringBuffer("D");
        join(var1,var2);
        join(var3,var4);
        //A<>B
        System.out.println(var1+"<>"+var2);
        //C<>DD
        System.out.println(var3+"<>"+var4);
    }
    public static void join (String s1,String s2){
        s1 = s2 ;
        s2 = s1+s2 ;
    }
    public static void join (StringBuffer s1,StringBuffer s2){
        s1 = s2 ;
        s2 = s2.append(s1) ;
    }
}
```

`絮叨一句`：String相关API传参问题，工作前三年跳槽基本都会被问到，如果不了解基本原理，心情再有点小慌，还基本会答错。

**源码参考：** https://gitee.com/cicadasmile/java-base-parent