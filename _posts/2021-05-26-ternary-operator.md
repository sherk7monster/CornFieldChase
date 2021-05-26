---
title:  "Java-关于三目运算符的一些坑"
category: "java"
---

2021-05-26 记

<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
</script>
<span id="busuanzi_container_page_pv">
  阅读量&nbsp;<span id="busuanzi_value_page_pv"></span>&nbsp;次，
</span>本文约 {{ content | strip_html | strip_newlines | split: "" | size }} 字

目录
* 目录
{:toc}

## 背景

#### 上代码

环境：`JDK 8`

```java
public class Demo {
    public static void main(String[] args) {
        Demo demo=new Demo();
        System.out.println(demo.strToInt("1"));
        System.out.println(demo.strToInt("2"));
        System.out.println(demo.strToInt(null));
    }

    public Integer strToInt(Object obj) {
        return (obj==null)?(Integer)obj:Integer.parseInt(obj.toString());
    }
}
```

#### 预测程序运行结果

```text
1
2
null
```

#### 实际运行结果

```text
1
2
Exception in thread "main" java.lang.NullPointerException
	at test.Demo.strToInt(Demo.java:12)
	at test.Demo.main(Demo.java:8)
```

## 分析

光从代码根本看不出来哪里会发生空指针，于是反编译代码，看看编译后的代码到底长什么样

#### 执行反编译

```text
javac -g Demo.java
javap -c Demo
```

#### 反编译结果

```text
Compiled from "Demo.java"
public class test.Demo {
  public test.Demo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class test/Demo
       3: dup
       4: invokespecial #3                  // Method "<init>":()V
       7: astore_1
       8: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
      11: aload_1
      12: ldc           #5                  // String 1
      14: invokevirtual #6                  // Method strToInt:(Ljava/lang/Object;)Ljava/lang/Integer;
      17: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
      20: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
      23: aload_1
      24: ldc           #8                  // String 2
      26: invokevirtual #6                  // Method strToInt:(Ljava/lang/Object;)Ljava/lang/Integer;
      29: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
      32: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
      35: aload_1
      36: aconst_null
      37: invokevirtual #6                  // Method strToInt:(Ljava/lang/Object;)Ljava/lang/Integer;
      40: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
      43: return

  public java.lang.Integer strToInt(java.lang.Object);
    Code:
       0: aload_1
       1: ifnonnull     14
       4: aload_1
       5: checkcast     #9                  // class java/lang/Integer
       8: invokevirtual #10                 // Method java/lang/Integer.intValue:()I
      11: goto          21
      14: aload_1
      15: invokevirtual #11                 // Method java/lang/Object.toString:()Ljava/lang/String;
      18: invokestatic  #12                 // Method java/lang/Integer.parseInt:(Ljava/lang/String;)I
      21: invokestatic  #13                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      24: areturn
}
```

#### 猜测

重点看 `strToInt()` 方法反编译的代码，发现有这么一段

```text
8: invokevirtual #10                 // Method java/lang/Integer.intValue:()I
```

再结合代码

```text
return (obj==null)?(Integer)obj:Integer.parseInt(obj.toString());
```

其中，`Integer.parseInt(obj.toString())` 返回的是 `int` 类型，怀疑 `(Integer)obj` 是先强转成 `Integer` 类型后，为了保持和其同样的 `int` 类型,做了自动拆箱。修改 `Integer.parseInt(obj.toString())` 为 `Integer.valueOf(obj.toString())` ，使其返回统一的 `Integer` 类型，程序运行后不再报错，返回了预期结果：

```text
1
2
null
```

#### 搜寻官网

查到如下官网文档：

```text
https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.25
```

其中有这么一句话：

```text
This conversion may include boxing or unboxing conversion (§5.1.7, §5.1.8).
```

应该是说三目运算符运算的时候，可能发生装箱拆箱。