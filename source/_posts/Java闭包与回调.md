---
title: Java闭包与回调
tags: 
	- Thinking in Java
	- java8
	- lambda
categories:  读书笔记
date: 2019-01-01 00:25:16
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
    void increment();
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

这里提到，通常回调函数是由指针传递实现的，然而Java中没有指针，那对于上面例子中写的闭包来说，应该怎么进行回调呢？这里书中给出了了答案：在外部类`Callee`中加入下列方法

```Java
public Incrementable getCallbackReference(){
   return new Closure();
}
```

这样虽然可以实现回调，但是一旦Callee的引用丢失，就无法再次找到闭包。

# 测试

这里对类的输出值进行打印。再增加一个类`caller`用于在外部回调闭包。

```Java
class caller{
	private Incrementable callbackReference;
	public caller(Incrementable cbh){
		this.callbackReference=cbh;
	}
	public void go(){
		callbackReference.increment(callbackReference);
	}
}
public class close {

	public static void main(String[] args) {
		Callee c = new Callee();//i=0
		MyIncrement.f(c);//i=1
		caller caller1 = new caller(c.getCallbackReference());
		caller caller2 = new caller(c.getCallbackReference());
		caller1.go();//i = 2
		caller1.go();//i = 3
		c= null;
		System.gc();
		caller2.go();//i = 4
	}
}
```

输入内容如下：

```java
Other operation
1
Other operation
2
Other operation
3
Other operation
4
```

可以看到`caller1`和`caller2`得到了的变量同时指向了外围类的`i`，即使引用变量`c`被设成`null`，由于对象内部已经形成了闭包，因此`i`不会被垃圾回收机制回收，再次调用`caller2`的`go()`方法时会`i`就递增到了4。

我们回顾一下闭包的概念：`要执行的代码块+环境作用域=闭包`。虽然`caller1`中的`closure`对象和`caller2`中的`closure`对象是两个不同的对象，但根据前面闭包的概念来说，他们是属于同一个闭包，因为他们拥有同样的作用域和将要执行的代码块。

```java
private class Closure implements Incrementable{
        public void increment(){
            Callee.this.increment();
            System.out.println(this);//打印当前对象地址
        }
    }
output:
      Other operation
      1
      Other operation										
      2
      com.simple.Callee$Closure@6d06d69c
      Other operation
      3
      com.simple.Callee$Closure@6d06d69c
      Other operation
      4
      com.simple.Callee$Closure@7852e922				/**分别存在两个closure对象**/

```



# lambda表达式与闭包

闭包通常是由匿名函数来实现，在Java中，可以利用匿名内部类来实现闭包：

```Java
    public Incrementable getCallbackReference(){
    	return new Incrementable() {
			@Override
			public void increment() {
				Callee.this.increment();
			}
		};
    }
```



然而，实现这个匿名内部类我们其实只不过是要实现一句话`Callee.this.increment();`加一个返回`return new Incrementable`，却要写这么一大堆代码，这对于简洁至上的我来说是绝对不能忍的。

在Java8之后，jdk提供了lambda表达式写法：

```Java
    public Incrementable getCallbackReference(){
    	return () ->Callee.this.increment();
    }
```

输出结果与上文一模一样，由此可以看出lambda表达式在运行时会自动形成一个闭包。而且由lambda形成的闭包可以作为函数参数进行传递。

# 总结

闭包这个概念最初出现于函数式语言中，如LISP等，但近几年来命令式语言也开始对函数式编程进行了支持，举例就是Java8新增的函数式接口与lambda等。虽然这里举例了Java在闭包的应用，但是总体来说Java并不适合函数式编程。最初看lambda表达式感觉怎么看怎么别扭，认为“这居然还是Java8新增的语法糖？”，但是在深入了解过lambda表达式的写法过后才发现，函数式编程如此美妙！

# 参考链接

[维基百科](https://zh.wikipedia.org/wiki/闭包_(计算机科学))

https://www.jianshu.com/p/c22db2a91989

[https://baike.baidu.com/item/%E9%97%AD%E5%8C%85](https://baike.baidu.com/item/闭包)

