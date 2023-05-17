# 一、Object简述

源码注释：Object类是所有类层级关系的Root节点，作为所有类的超类，包括数组也实现了该类的方法，注意这里说的很明确，指类层面。

所以在Java中有一句常说的话，一切皆对象，这话并不离谱。

**1、显式扩展**

**结论验证**

既然Object作为所有类的父级别的类，则不需要在显式的添加继承关系，`Each01`编译期就会提示移除冗余。

```java
public class Each01 extends Object {
    public static void main(String[] args) {
        System.out.println(new Each01().hashCode()+";"+new ObjEa02().hashCode());
    }
}
class ObjEa02 {}
class ObjEa03 extends ObjEa02{}
```

这里`Each01`与`ObjEa02`对象实例都有Object类中的`hashCode`方法，这里对既有结论的验证。

**编译文件**

再从JVM编译层面看下字节码文件，是如何加载，使用`javap -c`命令查看编译后的文件，注意Jdk版本`1.8`；

```
javap -c Each01.class
Compiled from "Each01.java"
public class com.base.object.each.Each01 {
  public com.base.object.each.Each01();
    Code:
       0: aload_0
       1: invokespecial #1 // Method java/lang/Object."<init>":()V
       4: return
}

javap -c ObjEa02.class 
Compiled from "Each01.java"
class com.base.object.each.ObjEa02 {
  com.base.object.each.ObjEa02();
    Code:
       0: aload_0
       1: invokespecial #1 // Method java/lang/Object."<init>":()V
       4: return
}

javap -c ObjEa03.class 
Compiled from "Each01.java"
class com.base.object.each.ObjEa03 extends com.base.object.each.ObjEa02 {
  com.base.object.each.ObjEa03();
    Code:
       0: aload_0
       1: invokespecial #1 // Method com/base/object/each/ObjEa02."<init>":()V
       4: return
}
```

**invokespecial命令**：可以查看Jvm的官方文档中的指令说明，调用实例化方法，和父类的初始化方法调用等，这里通过三个类的层级关系，再次说明Object超类不需要显式继承，即使显式声明但编译后源码依旧会清除冗余。

**2、引用与对象**

通常把下面过程称为：创建一个object对象；

```java
Object object = new Object() ;
```

细节描述：声明对象引用`object`；通过`new`关键字创建对象并基于默认构造方法初始化；将对象引用`object`指向创建的对象。

这一点可以基于Jvm运行流程去理解，所以当对象一旦失去全部引用时，会被标记为垃圾对象，在垃圾收集器运行时清理。

**接受任意数据类型对象的引用**

既然Object作为Java中所有对象的超类，则根据继承关系的特点，以及向上转型机制，Object可以接受任意数据类型对象的引用，例如在集合容器或者传参过程，不确定对象类型时可以使用Object：

```java
public class Each02 {
    public static void main(String[] args) {
        // 向上转型
        Object obj01 = new Each02Obj01("java") ;
        System.out.println(obj01);
        // 向下转型
        Each02Obj01 each02Obj01 = (Each02Obj01)obj01;
        System.out.println("name="+each02Obj01.getName());
    }
}
class Each02Obj01 {
    private String name ;
    public Each02Obj01(String name) { this.name = name; }
    @Override
    public String toString() {
        return "Each02Obj01{" +"name='" + name +'}';
    }
    public String getName() { return name; }
}
```

这里要强调一下这个向上转型的过程：

```java
Object obj01 = new Each02Obj01("java") ;
```

通过上面流程分析，这里创建一个父类引用`obj01`，并指向子类`Each02Obj01`对象，所以在输出的时候，调用的是子类的`toString`方法。

# 二、基础方法

**1、getClass**

在程序运行时获取对象的实例类，进而可以获取详细的结构信息并进行操作：

```java
public final native Class<?> getClass();
```

该方法在泛型，反射，动态代理等机制中有很多场景应用。

**2、toString**

返回对象的字符串描述形式，Object提供的是类名与无符号十六进制的哈希值组合表示，为了能返回一个信息明确的字符串，子类通常会覆盖该方法：

```java
public String toString() {
    return getClass().getName()+"@"+Integer.toHexString(hashCode());
}
```

在Java中，打印对象的时候，会执行`String.valueOf`转换为字符串，该方法的底层依旧是对象的`toString`方法：

```java
public void println(Object x) {
    String s = String.valueOf(x);
}
public static String valueOf(Object obj) {
    return (obj == null) ? "null" : obj.toString();
}
```

**3、equals与hashCode**

- equals：判断两个对象是否相等；
- hashCode：返回对象的哈希码值；

```java
public native int hashCode();
public boolean equals(Object obj) {
    return (this == obj);
}
```

`equals`判断方法需要考量实际的场景与策略，例如常见的公民注册后分配的身份ID是不能修改的，但是名字可以修改，那么就可能存在这样的场景：

```java
EachUser eachUser01 = new EachUser(1,"A") ;
EachUser eachUser02 = new EachUser(1,"B") ;
class EachUser {
    private Integer cardId ;
    private String name ;
}
```

从程序本身看，这确实是创建两个对象，但是放在场景下，这的确是描述同一个人，所以这时候可以在`equals`方法中定义比较规则，如果ID相同则视为同一个对象：

```java
@Override
public boolean equals(Object obj) {
    if (obj != null){
        EachUser compareObj = (EachUser)obj ;
        return this.cardId.intValue()==compareObj.cardId ;
    }
    return Boolean.FALSE ;
}
```

这里还要注意值类型和引用类型的区别，如果出现`null`比较情况，要返回false。

通常在子类中会同时覆盖这两个方法，这样做法在集合容器的设计上已经体现的淋漓尽致。

**4、thread相关**

- wait：线程进入waiting等待状态，不会争抢锁对象
- notify：随机通知一个在该对象上等待的线程；
- notifyAll：唤醒在该对象上所有等待的线程；

```java
public final native void wait(long timeout) throws InterruptedException;
public final native void notify();
public final native void notifyAll();
```

注意这里：`native`关键字修饰的方法，即调用的是原生函数，也就是常说的基于C/C++实现的本地方法，以此提高和系统层面的交互效率降低交互复杂程度。

**5、clone**

返回当前对象的拷贝：

```java
protected native Object clone() throws CloneNotSupportedException;
```

关于该方法的细节规则极度复杂，要注意下面几个核心点：

- 对象必须实现Cloneable接口才可以被克隆；
- 数据类型：值类型，String类型，引用类型；
- 深浅拷贝的区别和与之对应的实现流程；
- 在复杂的包装类型中，组合的不同变量类型；

**6、finalize**

当垃圾收集器确认该对象上没有引用时，会调用finalize方法，即清理内存释放资源：

```java
protected void finalize() throws Throwable { }
```

通常子类不会覆盖该方法，除非在子类中有一些其他必要的资源清理动作。

# 三、生命周期

**1、作用域**

在下面main方法执行结束之后，无法再访问`Each05Obj01`的实例对象，因为对象的引用`each05`丢失：

```java
public class Each05 {
    public static void main(String[] args) {
        Each05Obj01 each05 = new Each05Obj01 (99) ;
        System.out.println(each05);
    }
}
```

这里就会存在一个问题，引用丢失导致对象无法访问，但是对象在此时可能还是存在的，并没有释放内存的占用。

**2、垃圾回收机制**

Java通过new创建的对象会在堆中开辟内存空间存储，当对象失去所有引用时会被标记为垃圾对象，进而被回收；

这里涉及下面几个关键点：

- Jvm中垃圾收集器会监控创建的对象 ；
- 当判断对象不存在引用时，会执行清理动作；
- 完成对象清理后会重新整理内存空间；

这里存在一个很难理解的概念，即**对象不存在引用的判断**，也就是常说的**可达性分析算法**：基于对象到根对象的引用链是否可达来判断对象是否可以被回收；GC-Roots根引用集合，也可以变相理解为存活对象的集合。(详见JVM系列)

通过Object对象的分析，结合Java方方面面的机制和设计，可以去意会一些所谓的编程思想。

**源码参考：** https://gitee.com/cicadasmile/java-base-parent