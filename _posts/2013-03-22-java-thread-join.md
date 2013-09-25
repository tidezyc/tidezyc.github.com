---
layout: post
title: Java Thread.join()详解
category : java
tags:
- Java
- java
- join
- thread
published: true
---
### 一、使用方式。

join是Thread类的一个方法，启动线程后直接调用，例如：

> Thread t = new AThread();
> t.start();
> t.join();

### 二、为什么要用join()方法

在很多情况下，主线程生成并起动了子线程，如果子线程里要进行大量的耗时的运算，主线程往往将于子线程之前结束，但是如果主线程处理完其他的事务后，需要用到子线程的处理结果，也就是主线程需要等待子线程执行完成之后再结束，这个时候就要用到join()方法了。

### 三、join方法的作用

在JDk的API里对于join()方法是：

> ### join
> public final void *join()* throws InterruptedException
> Waits for this thread to die.
> Throws: InterruptedException 
> - if any thread has interrupted the current thread. The *interrupted status* of the current thread is cleared when this exception is thrown.

即join()的作用是：“等待该线程终止”，这里需要理解的就是该线程是指的主线程等待子线程的终止。也就是在子线程调用了join()方法后面的代码，只有等到子线程结束了才能执行。


### 四、用实例来理解

写一个简单的例子来看一下join()的用法：

1.AThread 类

2. BThread类

3. TestDemo 类

``` java
class BThread extends Thread {
	public BThread() {
		super("[BThread] Thread");
	};
	public void run() {
		String threadName = Thread.currentThread().getName();
		System.out.println(threadName + " start.");
		try {
			for (int i = 0; i &lt; 5; i++) {
				System.out.println(threadName + " loop at " + i);
				Thread.sleep(1000);
			}
			System.out.println(threadName + " end.");
		} catch (Exception e) {
			System.out.println("Exception from " + threadName + ".run");
		}
	}
}
class AThread extends Thread {
	BThread bt;
	public AThread(BThread bt) {
		super("[AThread] Thread");
		this.bt = bt;
	}
	public void run() {
		String threadName = Thread.currentThread().getName();
		System.out.println(threadName + " start.");
		try {
			bt.join();
			System.out.println(threadName + " end.");
		} catch (Exception e) {
			System.out.println("Exception from " + threadName + ".run");
		}
	}
}
public class TestDemo {
	public static void main(String[] args) {
		String threadName = Thread.currentThread().getName();
		System.out.println(threadName + " start.");
		BThread bt = new BThread();
		AThread at = new AThread(bt);
		try {
			bt.start();
			Thread.sleep(2000);
			at.start();
			at.join();
		} catch (Exception e) {
			System.out.println("Exception from main");
		}
		System.out.println(threadName + " end!");
	}
}
```

### 打印结果：

> main start.    //主线程起动，因为调用了at.join()，要等到at结束了，此线程才能向下执行。
> [BThread] Thread start.
> [BThread] Thread loop at 0
> [BThread] Thread loop at 1
> [AThread] Thread start.    //线程at启动，因为调用bt.join()，等到bt结束了才向下执行。
> [BThread] Thread loop at 2
> [BThread] Thread loop at 3
> [BThread] Thread loop at 4
> [BThread] Thread end.
> [AThread] Thread end.    // 线程AThread在bt.join();阻塞处起动，向下继续执行的结果
> main end!      //线程AThread结束，此线程在at.join();阻塞处起动，向下继续执行的结果。

<!--more-->

修改一下代码:

``` java
public class TestDemo {
	public static void main(String[] args) {
		String threadName = Thread.currentThread().getName();
		System.out.println(threadName + " start.");
		BThread bt = new BThread();
		AThread at = new AThread(bt);
		try {
			bt.start();
			Thread.sleep(2000);
			at.start();
			//at.join(); //在此处注释掉对join()的调用
		} catch (Exception e) {
			System.out.println("Exception from main");
		}
		System.out.println(threadName + " end!");
	}
}
```

### 打印结果：

> main start.    // 主线程起动，因为Thread.sleep(2000)，主线程没有马上结束;</p>
> [BThread] Thread start.    //线程BThread起动</p>
> [BThread] Thread loop at 0</p>
> [BThread] Thread loop at 1</p>
> main end!   // 在sleep两秒后主线程结束，AThread执行的bt.join();并不会影响到主线程。</p>
> [AThread] Thread start.    //线程at起动，因为调用了bt.join()，等到bt结束了，此线程才向下执行。</p>
> [BThread] Thread loop at 2</p>
> [BThread] Thread loop at 3</p>
> [BThread] Thread loop at 4</p>
> [BThread] Thread end.    //线程BThread结束了</p>
> [AThread] Thread end.    // 线程AThread在bt.join();阻塞处起动，向下继续执行的结果</blockquote>


### 五、从源码看join()方法

在AThread的run方法里，执行了bt.join();，进入看一下它的JDK源码：

``` java
    public final void join() throws InterruptedException {
		join(0L);
    }
```

然后进入join(0L)方法：

``` java
    public final synchronized void join(long l)
        throws InterruptedException
    {
        long l1 = System.currentTimeMillis();
        long l2 = 0L;
        if(l &lt; 0L)
            throw new IllegalArgumentException("timeout value is negative");
        if(l == 0L)
            for(; isAlive(); wait(0L));
        else
            do
            {
                if(!isAlive())
                    break;
                long l3 = l - l2;
                if(l3 &lt;= 0L)
                    break;
                wait(l3);
                l2 = System.currentTimeMillis() - l1;
            } while(true);
    }
```

单纯从代码上看：
* 如果线程被生成了，但还未被起动，isAlive()将返回false，调用它的join()方法是没有作用的。将直接继续向下执行。</li>
* 在AThread类中的run方法中，bt.join()是判断bt的active状态，如果bt的isActive()方法返回false，在bt.join(),这一点就不用阻塞了，可以继续向下进行了。从源码里看，wait方法中有参数，也就是不用唤醒谁，只是不再执行wait，向下继续执行而已。</li>
* 在join()方法中，对于isAlive()和wait()方法的作用对象是个比较让人困惑的问题：</li>

isAlive()方法的签名是：public final native boolean isAlive()，也就是说isAlive()是判断当前线程的状态，也就是bt的状态。

wait()方法在jdk文档中的解释如下：

> Causes the current thread to wait until another thread invokes the <code>notify()</code> method or the <code>notifyAll()</code> method for this object. In other words, this method behaves exactly as if it simply performs the call <tt>wait(0)</tt>.

> The current thread must own this object's monitor. The thread releases ownership of this monitor and waits until another thread notifies threads waiting on this object's monitor to wake up either through a call to the <code>notify</code> method or the <code>notifyAll</code> method. The thread then waits until it can re-obtain ownership of the monitor and resumes execution.

在这里，当前线程指的是at。
