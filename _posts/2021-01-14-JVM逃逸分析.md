TODO 

## 是什么

### 前导

java的编译分两种：javac编译（静态）和JIT编译（即时）。

JIT编译的功能，一是缓存，而是代码优化：如逃逸分析，锁消除，锁膨胀，方法内联，空值检查消除，类型检查消除，公共子表达式消除等。

### 定义

​	分析对象的动态作用域，当一个对象在方法中被定义之后，它可能被外部方法引用，例如作为参数传递到其他地方，这种现象称之为**方法逃逸**。如果返回得对象赋值给类变量或者成员变量还有可能被其他线程访问到，称为**线程逃逸**。



### 判断依据

1. 对象被赋值给堆中对象的字段和类的静态变量
2. 对象被传进了不确定的代码中去运行



## 有什么用

使用逃逸分析，编译器可以做出优化，降低堆内存占用，降低GC频率。

### 同步消除。

原代码

```java
public void f() {
    Object obj = new Object();
    synchronized(obj) {
        System.out.println(hollis);
    }
}
```

由于obj的作用域就在f()里，优化后的代码

```java
public void f() {
    Object obj = new Object();
    System.out.println(obj);
}
```



### 堆分配转化为栈分配

实现原理：标量替换。

验证见下面。

### 分离对象或者标量替换

对象的部分或者全部不放在内存而是CPU寄存器。

>
>
>标量：不可再分的数据，如java的原始类型数据。
>
>聚合量：还可以分解的数据，如Java中的对象。
>
>标量替换：逃逸分析之后，如果对象不可被外界访问的化，经过JIT优化之后，会把该对象拆分成若干个成员变量来代替。



```java
public static void main(String[] args) {
   create();
}

private static void create() {
   User user = new User（1,2）;
   System.out.println("user.name="+user.name+"; user.age="+user.age);
}
class User{
    private String name;
    private int age;
}
```

因为User对象不可逃逸，将被优化成如下，这样就大大节省了堆内存空间，为栈上分配提供了很好的基础。

```java
private static void create() {
    String name;
    int age;
   System.out.println("user.name="+user.name+"; user.age="+user.age);
}
```

问题：String是标量吗？

## 怎么用

jvm参数：-XX:+DoEscapeAnalysis , jdk 1.7开始默认开启。可以关闭-XX:-DoEscapeAnalysis

### 验证

#### 关闭逃逸分析

> -Xloggc:D:/gc.log  -XX:+PrintGC  -XX:+PrintGCDetails -Xms5M -Xmn5M  -XX:-DoEscapeAnalysis

```java
 /**
     * 创建对象
     * @return
     */
    public static    Object  creatObj(){
        return new Object();
    }

    public static void main(String[] args) {
        LocalDateTime start1 = LocalDateTime.now();
        for (long i = 0; i < 50_000_000L; i++) {
            creatObj();
        }
        System.out.println("花费：" + Duration.between(LocalDateTime.now(), start1));
    }
```

输出：

```
花费：PT-1.016S
```

gc日志：

```
1.501: [GC (Allocation Failure) [PSYoungGen: 4096K->504K(4608K)] 4096K->1265K(5632K), 0.0057320 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
1.507: [Full GC (Ergonomics) [PSYoungGen: 504K->502K(4608K)] [ParOldGen: 761K->722K(3584K)] 1265K->1225K(8192K), [Metaspace: 3558K->3558K(1056768K)], 0.0289070 secs] [Times: user=0.11 sys=0.00, real=0.03 secs] 
1.868: [GC (Allocation Failure) [PSYoungGen: 4598K->480K(4608K)] 5321K->1770K(8192K), 0.0034619 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
1.918: [GC (Allocation Failure) [PSYoungGen: 4576K->440K(4608K)] 5866K->1730K(8192K), 0.0022103 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
1.961: [GC (Allocation Failure) [PSYoungGen: 4536K->488K(4608K)] 5826K->1778K(8192K), 0.0025524 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2.002: [GC (Allocation Failure) [PSYoungGen: 4584K->504K(4608K)] 5874K->1802K(8192K), 0.0019622 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2.040: [GC (Allocation Failure) [PSYoungGen: 4600K->504K(2560K)] 5898K->1819K(6144K), 0.0021787 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2.059: [GC (Allocation Failure) [PSYoungGen: 2552K->0K(3584K)] 3867K->1741K(7168K), 0.0019794 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
.....
```

可以看到gc比较频繁。



#### 开启逃逸分析

> -Xloggc:D:/gc.log  -XX:+PrintGC  -XX:+PrintGCDetails -Xms5M -Xmn5M  -XX:+DoEscapeAnalysis

上面同样的代码

输出：

```
花费：PT-0.14S
```

gc日志：

```
0.853: [GC (Allocation Failure) [PSYoungGen: 4096K->488K(4608K)] 4096K->1565K(8192K), 0.0065691 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
1.059: [GC (Allocation Failure) [PSYoungGen: 4584K->504K(4608K)] 5661K->1884K(8192K), 0.0026558 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 4608K, used 3271K [0x00000007bfb00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 4096K, 67% used [0x00000007bfb00000,0x00000007bfdb3f68,0x00000007bff00000)
  from space 512K, 98% used [0x00000007bff80000,0x00000007bfffe010,0x00000007c0000000)
  to   space 512K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007bff80000)
 ParOldGen       total 3584K, used 1380K [0x00000006c1800000, 0x00000006c1b80000, 0x00000007bfb00000)
  object space 3584K, 38% used [0x00000006c1800000,0x00000006c19592e0,0x00000006c1b80000)
 Metaspace       used 5299K, capacity 5448K, committed 5632K, reserved 1056768K
  class space    used 578K, capacity 644K, committed 768K, reserved 1048576K
```

可以看到gc很少，标识对象不是在堆上分配的。





## 小结

1. 对象和数组不一定都在堆内存分配空间。
2. 即使开启逃逸分析的情况下，也不是所有的对象或者数组都没有在堆上创建。
3. 优化不一定会提升性能，极端情况下，经过逃逸分析之后，都不满足优化条件，这个分析过程中的性能损耗就白白浪费了。



## 参考

https://www.hollischuang.com/archives/2398