---
title: Java闭包与回调
tags: 
	- Thinking in Java
	- java8
	- lambda
categories:  读书笔记
date: 2019-07-05 00:25:16
---

# 前言

第一次知道闭包这个概念是在学习`JavaScript`的时候，相信接触过JavaScript的同学相信对闭包还有回调函数这两个名词一定不会陌生。深刻理解闭包这个概念，一定会使你的编程功力大增，而回调函数则是在编程中经常会用到的一种函数。然而在`Java8`以前，对于闭包这种函数式的编程方式支持很少，而且在其他语言中，回调通常利用指针进行。最近在学习《Java编程思想》的过程中，讲到内部类时就提到了Java的闭包与回调，下面就来分享一下学习的心得。

# 闭包

在第一次学习闭包`(closure)`这个名词时，着实是一头雾水，我们来看一下百度百科上面的解释：

[百度百科](https://baike.baidu.com/item/闭包)

-*闭包包含自由（未绑定到特定对象）变量，这些变量不是在这个代码块内或者任何全局上下文中定义的，而是在定义代码块的环境中定义（局部变量）。“闭包” 一词来源于以下两者的结合：要执行的代码块（由于自由变量被包含在代码块中，这些自由变量以及它们引用的对象没有被释放）和为自由变量提供绑定的计算环境（作用域）。*

再来看看维基百科上面的解释：

[维基百科](https://zh.wikipedia.org/wiki/闭包_(计算机科学))

*-**闭包**（英语：Closure），又称**词法闭包**（Lexical Closure）或**函数闭包**（function closures），是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。所以，有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。闭包在运行时可以有多个实例，不同的引用环境和相同的函数组合可以产生不同的实例。*

因此闭包就是**在作用域中引用了自由变量(一开始存在于作用域内)的函数。闭包一旦形成，这个变量就会和函数一起存在，即使离开了作用域也不会消失。**所以`闭包=自由变量+代码块`，闭包在形成之后自带运行环境。代码块就像是方便面的面饼，闭包就是自带调料包的方便面；代码块是发哥，闭包是高进——一个自带BGM的男人。就是因为这样，闭包在回调时不会丢失状态，这个后面通过书上的例子说明。

下面我们来看一下`Java`中的闭包是怎么实现的：

```Java
interface Incrementable{
    void increment;
}

class MyIncrement{
    public void increment(){System.out.println("Other operation");}
    static void f(MyIncrement mi){mi.increment();}
}

class Callee extends MyIncrement{ //创建闭包的作用域
    private int i = 0; //自由变量
    public void increment(){
        super.increment();
        i++;
        System.out.println(i);
    }
    /**
    * 创建闭包的代码块
    */
    private class Closure implements Incrementable{
        public void increment(){
            Callee.this.increment();
        }
    }
}

```

在这个程序中，类`closure`一旦被创建，它就会形成拥有对外部类变量的引用的一个闭包。

还有一点就是，虽然闭包和匿名内部函数（或者说Java中的匿名内部类）经常会同时出现，但是这两个不是同一个概念。

# 回调

回调函数是将函数指针调用的，假如将这个指针作为参数传递到另外一个函数中，那么在将来的某些时刻（或者说某些事件被触发）可以对函数进行调用。

# 测试

*~~blablablablabla~~*

# lambda表达式

*~~blablablabla~~*