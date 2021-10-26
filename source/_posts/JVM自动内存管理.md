---
title: JVM自动内存管理
date: 2021-02-23 13:43:13
categories:
- GC
tags:
- JVM
---

#### 手动管理内存

当使用C++这类没有自动内存管理的预约编写代码时，每一个new出的对象不再使用后都需要手动delete进行内存回收，因此非常容易出现内存泄漏和内存溢出问题。

```c++
#include <iostream>
using namespace std;
class A
{
    private:
        char *m_cBuffer;
        int m_nLen;
    public:
        A(){ m_cBuffer = new char[m_nLen]; }
        ~A() { delete [] m_cBuffer; }
};

A *a = new A
    
delete a;
```

#### 自动内存管理

对于Java程序员来说，在虚拟机自动内存管理机制的帮助下，不再需要为每一个new操作去写配对的delete/free代码，不容易出现内存泄漏和内存溢出问题，由虚拟机管理内存这一切看起来都很美好。不过也正是因为Java程序员把内存控制的权力交给了Java虚拟机，一旦出现内存泄漏和溢出方面的问题，如果不了解虚拟机是怎样使用内存的，那么排查错误将会成为一项异常艰难的工作。

#### 虚拟机内存划分

![JVM自动内存管理-01](D:\Github\hexo\public\images\JVM自动内存管理-01.png)

- 虚拟机栈

  虚拟机栈描述的是Java方法执行的内存模型，这是线程私有的生命周期与线程相同，每个方法在执行的同时都会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常。

- 本地方法栈

  虚拟机使用到的Native方法服务

- 程序计数器

  程序计数器的主要作用是记录线程运行时的状态，方便线程被唤醒时能从上一次被挂起时的状态继续执行

- 堆

  对于大多数应用来说，Java堆（Java Heap）是Java虚拟机所管理的内存中最大的一块也是垃圾收集器管理的主要区域。

#### 判断对象是否死亡

- 引用计数

  对象被引用一次就在在它的对象头上加一次引用次数，如果没有被引用（引用次数为 0）则此对象可回收。

  ![JVM自动内存管理-02](D:\Github\hexo\public\images\JVM自动内存管理-02.gif)

  ```java
  public class TestRC {
  
  	TestRC instance;
      public TestRC(String name) {
      }
  
      public static void main(String[] args) {
         // 第一步
          A a = new TestRC("a");
          B b = new TestRC("b");
  
          // 第二步
          a.instance = b;
          b.instance = a;
  
          // 第三步
          a = null;
          b = null;
      }
  }
  ```

- 可达性分享

  这个算法的基本思路就是通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（用图论的话来说，就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。

  ![JVM自动内存管理-03](D:\Github\hexo\public\images\JVM自动内存管理-03.png)

  - GC ROOT

    1. 虚拟机栈（栈帧中的本地变量表）中引用的对象

       ```java
       public class Test {
           public static  void main(String[] args) {
       	Test a = new Test();
       	a = null;
           }
       }
       ```

       

    2. 方法区中类静态属性引用的对象
  
       ```java
       public class Test {
           public static Test s;
           public static  void main(String[] args) {
               Test a = new Test();
               a.s = new Test();
               a = null;
           }
       }
       ```
    
    3. 方法区中常量引用的对象
    
       ```java
       public class Test {
       	   public static final Test s = new Test();
           public static  void main(String[] args) {
       	   Test a = new Test();
               a = null;
           }
       }
       ```
    
    4. 本地方法栈中 JNI 引用的对象
  

#### 经典垃圾回收算法

- 标记清除算法
- 复制算法
- 标记整理法
- 分代收集算法