---
layout: post
title: 理解Java synchronized 关键字
date: 2018-9-30
tags: Java基础
---
### 概念
&emsp;&emsp;在多线程编程中，线程安全问题是我们必须要面对的一个问题，Java中就使用了synchronized关键字来实现同步，在理解这个关键字之前，我们首先要明确一点，synchronized锁的不是代码，而是锁的对象。我们先看看下面的两个特性：  
- 互斥性：在同一时间，只允许一个线程持有某个对象锁，这样就可以保证在同一时间只有一个线程在访问互斥资源，互斥性也成为原子性。  
- 可见性：必须确保锁在释放之前，当前线程对于共享变量的修改，对下一个获得锁的线程来说，该修改是可见的（即获取到的共享变量的值是最新的），否则另一个变量获取到的变量值可能是之前缓存的副本，而不是最新的值。（Java中还有一个关键字volatile，它只保证可见性，但是不保证原子性）

### 实例锁和类锁
##### 1、实例锁
&emsp;&emsp;Java中一个对象可以有多个实例，每个实例都有其各独立的实例锁，互不干扰，通常也称作“内置锁”或“对象锁”。对于同一个实例对象，在同一时刻只有线程可以访问这个实例对象的<b>同步方法</b>,不同的实例对象，并不能保证多线程的同步操作。
##### 2、类锁
&emsp;&emsp;类锁其实是通过对象锁来实现的，因为每一个类只有一个`Class`对象，所以每个类只有一个类锁（全局锁），在同一时刻，只有一个线程可以访问这个类的同步方法。

### 用法
##### 1、同步非静态方法，锁的是当前实例对象
```Java
public class SynchronizedDemo {

    public synchronized void mehtodA() {
        System.out.println("method A ");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
    }    
    public synchronized void methodB() {
        System.out.println("method B ");
    }
    public static void main(String[] args) {
        SynchronizedDemo demo = new SynchronizedDemo();

        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                demo.mehtodA();
            }
        });
        t1.start();    
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                demo.methodB();

            }
        });
        t2.start();
    }
}
```
&emsp;&emsp;该例子中，运行程序会先输出`method A`，然后等待两秒之后才会输出`method B`。由于两个方法都加了`synchronized`关键字，所以当第一个线程通过demo这个对象实例调用`methodA()`的时候，此时该实例被锁住，线程2想要使用这个实例来调用`methodB()`的时候，必须等待当前对象实例释放之后才能调用（由于`methodA()`中sleep了两秒，所以`methodA()`会持有该实例两秒，释放之后`methodB()`才能进行调用）。

##### 2、同步静态方法，锁的是当前类的Class对象（全局锁）
```Java
public class SynchronizedDemo2 {

    public synchronized static void mehtodA() {
        System.out.println("method A ");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public synchronized static void methodB() {
        System.out.println("method B ");
    }

    public static void main(String[] args) {
        SynchronizedDemo2 demo1 = new SynchronizedDemo2();
        SynchronizedDemo2 demo2 = new SynchronizedDemo2();

        Thread t1 = new Thread(()-> {demo1.mehtodA();});
        t1.start();
        Thread t2 = new Thread(() -> {demo2.methodB();});
        t2.start();

    }
}
```
&emsp;&emsp;该例子和1中同步非静态方法输出的结果是一样的，区别是例1中是由同一个实例对象`demo`调用两个方法，`synchronized`对这个对这个实例对象进行加锁，导致`methodB()`延迟了两秒执行。在本例中，则是由两个实例对象`demo1`和`demo2`分别调用了`methodA()`和`methodB()`，按照例1中的思路，应该是两个线程执行不同实例对象，互不影响才对，为什么执行的结果是一样的呢？  
&emsp;&emsp;这就是因为在本例中，两个方法都是`static`方法，`synchronized`对静态方法加锁，是全局锁，锁的是这个类的`Class`对象，由于`demo1`和`demo2`的`Class`对象时相同的（都是`SynchronizedDemo2.class`），所以例子中虽然使用的是两个对象实例，但是`synchronized`关键字锁住的都是同一个对象，所以结果就和例1中的结果一样了。
##### 3、同步代码块  
- <b>1 synchronized(this),synchronized(实例对象);锁住的是`()`中的对象实例

```Java
public class SynchronizedDemo4 {

    private String str = "I'm a String Object";

    public void methodA() {
        synchronized (str) {
            System.out.println("method A sync block");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public void methodB() {
        synchronized (str) {
            System.out.println("method B sync block");
        }
    }

    public void methodC() {
        synchronized (this) {
            System.out.println("method C sync block");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public void methodD() {
        synchronized (this) {
            System.out.println("method D sync block");
        }
    }

    public static void main(String[] args) {
        SynchronizedDemo4 demo = new SynchronizedDemo4();

        Thread t1 = new Thread(()->{demo.methodA();});
        t1.start();

        Thread t2 = new Thread(()->{demo.methodB();});
        t2.start();

        Thread t3 = new Thread(()->{demo.methodC();});
        t3.start();

        Thread t4 = new Thread(()->{demo.methodD();});
        t4.start();
    }
}
```
&emsp;&emsp;在本例中`methodA()`和`methodB()`锁住的是同一个实例对象`str`,`methodC()`和`methodD()`锁住的是当前对象`this`。测试用例中使用的是同一个实例对象`demo`对四个方法用四个线程执行调用。输出的结果是`methodA()`和和`methodC()`先执行，等待两秒后`methodD()`和`methodB()`。  
&emsp;&emsp;由于在文章开头们就提到，synchronized关键字锁住的是对象，对于同步代码块来说，锁住的就是`()`中的对象。由于方法`methodA()`和`methodB()`锁住的是同一个对象实例`str`，`methodC()`和`methodD()`锁住的是当前对象，所以在执行的时候，线程t1和t3可以同时执行，当`str`和`this`释放之后，线程t2和t4才得以执行。所以最终我们就得到了上述的运行结果。  
- <b>2 synchronized(类.class);锁住的是`()`中的类对象（Class对象）
```Java
public class SynchronizedDemo5 {

    public void methodA() {
        synchronized (SynchronizedDemo5.class) {
            System.out.println("method A sync block");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public void methodB() {
        synchronized (SynchronizedDemo5.class) {
            System.out.println("method B sync block");
        }
    }

    public static void main(String[] args) {
        SynchronizedDemo5 demo = new SynchronizedDemo5();

        Thread t1 = new Thread(()->{demo.methodA();});
        t1.start();

        Thread t2 = new Thread(()->{demo.methodB();});
        t2.start();
    }
}
```
&emsp;&emsp;这个例子的输出也是，先执行方法A，等待两秒后再执行方法B，测试用例中，我同样使用的是两个不同的实例，分别对A和B进行调用。原因和同步静态方法类似，这里的同步代码块锁住的是类`SynchronizedDemo5`的`Class`对象，由于`demo1`和`demo2`的类对象都是一样的(同为`SynchronizedDemo5.class`)，所以方法B想要执行，就必须等待类对象被释放之后才能进行执行。

### 总结
&emsp;&emsp;其实要理解`synchronized`关键字，只需要搞明白两个东西，一个是实例锁，一个是类锁(全局锁)。实例锁是每个实例各自拥有的，实例之间相互独立；类锁则相反，每个类只有一个类锁，对于同一个类的不同实例，类锁是相同的，这也是为什么类锁又叫全局锁的原因。
