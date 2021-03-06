---
layout: post_layout
title: java内部类总结[转]
date: 2016-09-27
location: 北京
pulished: true
excerpt_separator: "```"
---
本文来自：[Nerxious](http://www.cnblogs.com/nerxious/archive/2013/01/24/2875649.html)  

*片外语：一个比较不怎么实用的知识，可能只会用在面试或是考试，smiling~*

---
内部类不是很好理解，但说白了其实也就是一个类中还包含着另外一个类。

如同一个人是由大脑、肢体、器官等身体结果组成，而内部类相当于其中的某个器官之一，例如心脏：它也有自己的属性和行为（血液、跳动）。

显然，此处不能单方面用属性或者方法表示一个心脏，而需要一个类。

而心脏又在人体当中，正如同是内部类在外部内当中。

---
 

#### 实例1：内部类的基本结构
```javascript
//外部类
class Out {
    private int age = 12;
     
    //内部类
    class In {
        public void print() {
            System.out.println(age);
        }
    }
}
```
```javascript
public class Demo {
    public static void main(String[] args) {
        Out.In in = new Out().new In();
        in.print();
        //或者采用下种方式访问
        /*
        Out out = new Out();
        Out.In in = out.new In();
        in.print();
        */
    }
}
```

运行结果：`12`

---


从上面的例子不难看出，内部类其实严重破坏了良好的代码结构，但为什么还要使用内部类呢？

因为内部类可以随意使用外部类的成员变量（包括私有）而不用生成外部类的对象，这也是内部类的唯一优点。

如同心脏可以直接访问身体的血液，而不是通过医生来抽血。

程序编译过后会产生两个.class文件，分别是`Out.class`和`Out$In.class`

其中$代表了上面程序中`Out.In`中的那个  
`Out.In in = new Out().new In()`  
可以用来生成内部类的对象，这种方法存在两个小知识点需要注意：


>1. 开头的Out是为了标明需要生成的内部类对象在哪个外部类当中
>2. 必须先有外部类的对象才能生成内部类的对象，因为内部类的作用就是为了访问外部类中的成员变量

---

#### 实例2：内部类中的变量访问形式
```javascript
class Out {
    private int age = 12;
     
    class In {
        private int age = 13;
        public void print() {
            int age = 14;
            System.out.println("局部变量：" + age);
            System.out.println("内部类变量：" + this.age);
            System.out.println("外部类变量：" + Out.this.age);
        }
    }
}
 
public class Demo {
    public static void main(String[] args) {
        Out.In in = new Out().new In();
        in.print();
    }
}
```

运行结果：

```code
局部变量：14  
内部类变量：13  
外部类变量：12 
```

---
从实例1中可以发现，内部类在没有同名成员变量和局部变量的情况下，内部类会直接访问外部类的成员变量，而无需指定`Out.this.`属性名

否则，内部类中的局部变量会覆盖外部类的成员变量

而访问内部类本身的成员变量可用`this.`属性名，访问外部类的成员变量需要使用`Out.this.`属性名

---

#### 实例3：静态内部类

```javascript
class Out {
    private static int age = 12;
     
    static class In {
        public void print() {
            System.out.println(age);
        }
    }
}
 
public class Demo {
    public static void main(String[] args) {
        Out.In in = new Out.In();
        in.print();
    }
}
```
运行结果：`12`

---

可以看到，如果用static 将内部内静态化，那么内部类就只能访问外部类的静态成员变量，具有局限性

其次，因为内部类被静态化，因此`Out.In`可以当做一个整体看，可以直接new 出内部类的对象（通过类名访问static，生不生成外部类对象都没关系）

---

#### 实例4：私有内部类
```javascript
class Out {
    private int age = 12;
     
    private class In {
        public void print() {
            System.out.println(age);
        }
    }
    public void outPrint() {
        new In().print();
    }
}
 
public class Demo {
    public static void main(String[] args) {
        //此方法无效
        /*
        Out.In in = new Out().new In();
        in.print();
        */
        Out out = new Out();
        out.outPrint();
    }
}
```
运行结果：`12`

---
如果一个内部类只希望被外部类中的方法操作，那么可以使用private声明内部类

上面的代码中，我们必须在Out类里面生成In类的对象进行操作，而无法再使用  
`Out.In in = new Out().new In()`  
生成内部类的对象

也就是说，此时的内部类只有外部类可控制

如同是，我的心脏只能由我的身体控制，其他人无法直接访问它

---

#### 实例5：方法内部类
```javascript
class Out {
    private int age = 12;
 
    public void Print(final int x) {
        class In {
            public void inPrint() {
                System.out.println(x);
                System.out.println(age);
            }
        }
        new In().inPrint();
    }
}
 
public class Demo {
    public static void main(String[] args) {
        Out out = new Out();
        out.Print(3);
    }
}
```
运行结果：

```code
3  
12
```

---
在上面的代码中，我们将内部类移到了外部类的方法中，然后在外部类的方法中再生成一个内部类对象去调用内部类方法

如果此时我们需要往外部类的方法中传入参数，那么外部类的方法形参必须使用final定义

至于final在这里并没有特殊含义，只是一种表示形式而已
