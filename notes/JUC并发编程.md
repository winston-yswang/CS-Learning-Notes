## 1、多线程基础

现代操作系统（Windows，macOS，Linux）都可以执行多任务。多任务就是同时运行多个任务，CPU执行代码都是一条一条顺序执行的，但是，即使是单核cpu，也可以同时运行多个任务。因为操作系统执行多任务实际上就是让CPU对多个任务轮流交替执行。

在计算机中，进程和线程是包含关系，但是多任务既可以由多进程实现，也可以由单进程内的多线程实现，还可以混合多进程＋多线程。

具体采用哪种方式，要考虑到进程和线程的特点。

和多线程相比，多进程的缺点在于：

- 创建进程比创建线程开销大，尤其是在Windows系统上；
- 进程间通信比线程间通信要慢，因为线程间通信就是读写同一个变量，速度很快。

而多进程的优点在于：

​	多进程稳定性比多线程高，因为在多进程的情况下，一个进程崩溃不会影响其他进程，而在多线程的情况下，任何一个线程崩溃会直接导致整个进程崩溃。

对于大多数Java程序来说，多线程模型是Java程序最基本的并发模型，是必须掌握的技能之一。

### 1.1 创建新线程

Java用`Thread` 对象表示一个线程，通过调用`start()` 启动一个新线程。一个线程对象只能调用一次`start()` 方法。线程的执行代码写在`run()` 方法中，一旦`run()` 方法执行完毕，线程就结束了，并且线程其调度由操作系统决定，程序本身无法决定调度顺序。`Thread.sleep` 可以把当前线程暂停一段时间。

```java
// 方法一
public class Main {
    public static void main(String[] args) {
        System.out.println("main start...");
        Thread t = new Thread() {
            public void run() {
                System.out.println("thread run...");
                System.out.println("thread end.");
            }
        };
        t.start();
        System.out.println("main end...");
    }
}

// 方法二
public class Main {
    public static void main(String[] args) {
        new Thread(new Mythread()).start();
    }
    
}
class MyThread implements Runnable {
    @Override
    public void run() {...};
}
```

### 1.2 线程的状态

Java线程的状态有以下几种：

- New：新创建的线程，尚未执行；
- Runnable：运行中的线程，正在执行`run()`方法的Java代码；
- Blocked：运行中的线程，因为某些操作被阻塞而挂起；
- Waiting：运行中的线程，因为某些操作在等待中；
- Timed Waiting：运行中的线程，因为执行`sleep()`方法正在计时等待；
- Terminated：线程已终止，因为`run()`方法执行完毕。

当线程启动后，它可以在`Runnable`、`Blocked`、`Waiting`和`Timed Waiting`这几个状态之间切换，直到最后变成`Terminated`状态，线程终止。

线程终止的原因有：

- 线程正常终止：`run()`方法执行到`return`语句返回；
- 线程意外终止：`run()`方法因为未捕获的异常导致线程终止；
- 对某个线程的`Thread`实例调用`stop()`方法强制终止

一个线程还可以等待另一个线程直到其运行结束。例如，`main`线程在启动`t`线程后，可以通过`t.join()`等待`t`线程结束后再继续运行：

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            System.out.println("hello");
        });
        System.out.println("start");
        t.start();
        t.join();
        System.out.println("end");
    }
}
```



### 1.3 线程同步

如果多个线程同时读写共享变量，会出现数据不一致的问题。可以通过加锁和解锁的操作，就能保证指令总是在一个线程执行期间，不会有其他线程会进入此指令区间。即使在执行期线程被操作系统中断执行，其他线程也会因为无法获得锁导致无法进入此指令区间。只有执行线程将锁释放后，其他线程才有机会获得锁并执行。这种加锁和解锁之间的代码块我们称之为临界区（Critical Section），任何时候临界区最多只有一个线程能执行。

可见，保证一段代码的原子性就是通过加锁和解锁实现的。Java程序使用`synchronized`关键字对一个对象进行加锁：

```java
synchronized(lockObject) {
    ...;
}
```

`synchronized`保证了代码块在任意时刻最多只有一个线程能执行。

JVM规范定义了几种原子操作，可以不需要synchronized的操作：

- 基本类型（`long`和`double`除外）赋值，例如：`int n = m`；
- 引用类型赋值，例如：`List<String> list = anotherList`。

`long`和`double`是64位数据，JVM没有明确规定64位赋值操作是不是一个原子操作，不过在x64平台的JVM是把`long`和`double`的赋值作为原子操作实现的。

单条原子操作的语句不需要同步。例如：

```java
public void set(int m) {
    synchronized(lock) {
        this.value = m;
    }
}
```



### 1.4 同步方法

让线程自己选择锁对象往往会使得代码逻辑混乱，也不利于封装。更好的方法是把`synchronized`逻辑封装起来。例如：

```java
public class Counter {
    private int count = 0;

    public void add(int n) {
        synchronized(this) {
            count += n;
        }
    }

    public void dec(int n) {
        synchronized(this) {
            count -= n;
        }
    }

    public int get() {
        return count;
    }
}
```

这样一来，线程调用`add()`、`dec()`方法时，它不必关心同步逻辑，因为`synchronized`代码块在`add()`、`dec()`方法内部。并且，我们注意到，`synchronized`锁住的对象是`this`，即当前实例，这又使得创建多个`Counter`实例的时候，它们之间互不影响，可以并发执行。

如果一个类被设计为允许多线程正确访问，我们就说这个类就是“线程安全”的（thread-safe），上面的`Counter`类就是线程安全的。Java标准库的`java.lang.StringBuffer`也是线程安全的。

还有一些不变类，例如`String`，`Integer`，`LocalDate`，它们的所有成员变量都是`final`，多线程同时访问时只能读不能写，这些不变类也是线程安全的。

最后，类似`Math`这些只提供静态方法，没有成员变量的类，也是线程安全的。

除了上述几种少数情况，大部分类，例如`ArrayList`，都是非线程安全的类，我们不能在多线程中修改它们。但是，如果所有线程都只读取，不写入，那么`ArrayList`是可以安全地在线程间共享的。

> **同步方法**

当我们锁住的是`this`实例时，实际上可以用`synchronized`修饰这个方法。下面两种写法是等价的：

```java
public void add(int n) {
    synchronized(this) { // 锁住this
        count += n;
    } // 解锁
}

public synchronized void add(int n) { // 锁住this
    count += n;
} // 解锁
```

因此，用`synchronized`修饰的方法就是同步方法，它表示整个方法都必须用`this`实例加锁。

我们再思考一下，如果对一个静态方法添加`synchronized`修饰符，它锁住的是哪个对象？

```java
public synchronized static void test(int n) {
    ...
}
```

对于`static`方法，是没有`this`实例的，因为`static`方法是针对类而不是实例。但是我们注意到任何一个类都有一个由JVM自动创建的`Class`实例，因此，对`static`方法添加`synchronized`，锁住的是该类的`Class`实例。上述`synchronized static`方法实际上相当于：

```java
public class Counter {
    public static void test(int n) {
        synchronized(Counter.class) {
            ...
        }
    }
}
```

我们再考察`Counter`的`get()`方法：

```java
public class Counter {
    private int count;

    public int get() {
        return count;
    }
    ...
}
```

它没有同步，因为读一个`int`变量不需要同步。

然而，如果我们把代码稍微改一下，返回一个包含两个`int`的对象：

```java
public class Counter {
    private int first;
    private int last;

    public Pair get() {
        Pair p = new Pair();
        p.first = first;
        p.last = last;
        return p;
    }
    ...
}
```

就必须要同步了。



### 1.5 死锁

Java的线程锁是可重入的锁。

什么是可重入的锁？我们还是来看例子：

```java
public class Counter {
    private int count = 0;

    public synchronized void add(int n) {
        if (n < 0) {
            dec(-n);
        } else {
            count += n;
        }
    }

    public synchronized void dec(int n) {
        count += n;
    }
}
```

观察`synchronized`修饰的`add()`方法，一旦线程执行到`add()`方法内部，说明它已经获取了当前实例的`this`锁。如果传入的`n < 0`，将在`add()`方法内部调用`dec()`方法。由于`dec()`方法也需要获取`this`锁，现在问题来了：

对同一个线程，能否在获取到锁以后继续获取同一个锁？

答案是肯定的。JVM允许同一个线程重复获取同一个锁，这种能被同一个线程反复获取的锁，就叫做可重入锁。

由于Java的线程锁是可重入锁，所以，获取锁的时候，不但要判断是否是第一次获取，还要记录这是第几次获取。每获取一次锁，记录+1，每退出`synchronized`块，记录-1，减到0的时候，才会真正释放锁。

**死锁：**

一个线程可以获取一个锁后，再继续获取另一个锁。例如：

```java
public void add(int m) {
    synchronized(lockA) { // 获得lockA的锁
        this.value += m;
        synchronized(lockB) { // 获得lockB的锁
            this.another += m;
        } // 释放lockB的锁
    } // 释放lockA的锁
}

public void dec(int m) {
    synchronized(lockB) { // 获得lockB的锁
        this.another -= m;
        synchronized(lockA) { // 获得lockA的锁
            this.value -= m;
        } // 释放lockA的锁
    } // 释放lockB的锁
}
```

在获取多个锁的时候，不同线程获取多个不同对象的锁可能导致死锁。对于上述代码，线程1和线程2如果分别执行`add()`和`dec()`方法时：

- 线程1：进入`add()`，获得`lockA`；
- 线程2：进入`dec()`，获得`lockB`。

随后：

- 线程1：准备获得`lockB`，失败，等待中；
- 线程2：准备获得`lockA`，失败，等待中。

此时，两个线程各自持有不同的锁，然后各自试图获取对方手里的锁，造成了双方无限等待下去，这就是死锁。

死锁发生后，没有任何机制能解除死锁，只能强制结束JVM进程。

因此，在编写多线程应用时，要特别注意防止死锁。因为死锁一旦形成，就只能强制结束进程。

那么我们应该如何避免死锁呢？答案是：线程获取锁的顺序要一致。即严格按照先获取`lockA`，再获取`lockB`的顺序，改写`dec()`方法如下：

```java
public void dec(int m) {
    synchronized(lockA) { // 获得lockA的锁
        this.value -= m;
        synchronized(lockB) { // 获得lockB的锁
            this.another -= m;
        } // 释放lockB的锁
    } // 释放lockA的锁
}
```



### 1.6 使用wait和notify

在Java程序中，`synchronized`解决了多线程竞争的问题。例如，对于一个任务管理器，多个线程同时往队列中添加任务，但是`synchronized`并没有解决多线程协调的问题。

仍然以上面的`TaskQueue`为例，我们再编写一个`getTask()`方法取出队列的第一个任务：

```java
class TaskQueue {
    Queue<String> queue = new LinkedList<>();

    public synchronized void addTask(String s) {
        this.queue.add(s);
    }

    public synchronized String getTask() {
        while (queue.isEmpty()) {
        }
        return queue.remove();
    }
}
```

上述代码看上去没有问题：`getTask()`内部先判断队列是否为空，如果为空，就循环等待，直到另一个线程往队列中放入了一个任务，`while()`循环退出，就可以返回队列的元素了。

但实际上`while()`循环永远不会退出。因为线程在执行`while()`循环时，已经在`getTask()`入口获取了`this`锁，其他线程根本无法调用`addTask()`，因为`addTask()`执行条件也是获取`this`锁。

因此，执行上述代码，线程会在`getTask()`中因为死循环而100%占用CPU资源。

如果深入思考一下，我们想要的执行效果是：

- 线程1可以调用`addTask()`不断往队列中添加任务；
- 线程2可以调用`getTask()`从队列中获取任务。如果队列为空，则`getTask()`应该等待，直到队列中至少有一个任务时再返回。

因此，多线程协调运行的原则就是：当条件不满足时，线程进入等待状态；当条件满足时，线程被唤醒，继续执行任务。

对于上述`TaskQueue`，我们先改造`getTask()`方法，在条件不满足时，线程进入等待状态：

```java
public synchronized String getTask() {
    while (queue.isEmpty()) {
        // 释放this锁:
        this.wait();
        // 重新获取this锁
    }
    return queue.remove();
}
```

当一个线程执行到`getTask()`方法内部的`while`循环时，它必定已经获取到了`this`锁，此时，线程执行`while`条件判断，如果条件成立（队列为空），线程将执行`this.wait()`，进入等待状态。

这里的关键是：`wait()`方法必须在当前获取的锁对象上调用，这里获取的是`this`锁，因此调用`this.wait()`。

`wait()`不是一个普通的Java方法，而是定义在`Object`类的一个`native`方法，也就是由JVM的C代码实现的。其次，必须在`synchronized`块中才能调用`wait()`方法，因为`wait()`方法调用时，会释放线程获得的锁，`wait()`方法返回后，线程又会重新试图获得锁。

当一个线程在`this.wait()`等待时，它就会释放`this`锁，从而使得其他线程能够在`addTask()`方法获得`this`锁。

现在我们面临第二个问题：如何让等待的线程被重新唤醒，然后从`wait()`方法返回？答案是在相同的锁对象上调用`notify()`方法。我们修改`addTask()`如下：

```java
public synchronized void addTask(String s) {
    this.queue.add(s);
    this.notify(); // 唤醒在this锁等待的线程
}
```

注意到在往队列中添加了任务后，线程立刻对`this`锁对象调用`notify()`方法，这个方法会唤醒一个正在`this`锁等待的线程（就是在`getTask()`中位于`this.wait()`的线程），从而使得等待线程从`this.wait()`方法返回。



## 2、JUC简要

JUC是java.util.concurrent工具包的简称，这是一个处理线程的工具包。

> 传统的synchronized

```java
public class SaleTicketDemo {
    public static void main(String[] args) {
        // 并发：多线程操作同一个资源类
        Ticket ticket = new Ticket();

        // @FunctionalInterface 函数式接口; Lamdba表达式 (参数)->{}
        new Thread(()->{
            for (int i = 1; i < 20; i++) {
                ticket.sale();
            }
        }, "A").start();
        new Thread(()->{
            for (int i = 1; i < 20; i++) {
                ticket.sale();
            }
        },"B").start();
        new Thread(()->{
            for (int i = 1; i < 20; i++) {
                ticket.sale();
            }
        },"C").start();
    }
}

class Ticket {
    private int ticketCount = 30;

    // synchronized本质是 队列+锁
    public synchronized void sale() {
        if (ticketCount > 0) {
            System.out.println(Thread.currentThread().getName() + "卖出了第" + (ticketCount--) + "张票，剩余: " + ticketCount);
        }
    }
}
```



> Lock接口

```java
public class SaleTiketDemo {
    public static void main(String[] args) {
        // 并发：多线程操作同一个资源类
        Ticket ticket = new Ticket();

        // @FunctionalInterface 函数式接口; Lamdba表达式
        new Thread(()->{ for (int i = 1; i < 20; i++) ticket.sale(); }, "A").start();
        new Thread(()->{ for (int i = 1; i < 20; i++) ticket.sale(); }, "B").start();
        new Thread(()->{ for (int i = 1; i < 20; i++) ticket.sale(); }, "C").start();
    }
}

class Ticket {
    private int ticketCount = 30;

    Lock lock = new ReentrantLock();

    public void sale() {

        lock.lock();    // 加锁

        try {
            // 业务代码
            if (ticketCount > 0) {
                System.out.println(Thread.currentThread().getName() + "卖出了第" + (ticketCount--) + "张票，剩余: " + ticketCount);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();  // 解锁
        }
    }
}
```



> Synchronized 和 Lock 区别

1、Synchronized 是内置的Java关键字，而Lock是一个Java接口；

2、Synchronized 无法判断和获取锁的状态，而Lock可以判断是否获取到了锁；

3、Synchronized 会自动释放锁，而Lock必须要手动释放锁，否则可能会导致**死锁**；

4、Synchronized 加入线程1获得锁但被阻塞了，线程2不会自动结束等待获取资源，而Lock有tryLock则不一定会等待下去；

5、Synchronized 是可重入锁，不可以中断，非公平锁，而Lock是一个接口，默认可重入锁，可以判断和实现锁；

6、Synchronized 适合锁少量的代码同步问题，Lock适合锁大量的同步代码。



> 生产者和消费者问题



```java
public class SaleTiketDemo {
    public static void main(String[] args) {
        // 并发：多线程操作同一个资源类
        Ticket ticket = new Ticket();
        new Thread(()->{ for (int i = 1; i <= 20; i++) ticket.sale(); }, "A").start();
        new Thread(()->{ for (int i = 1; i <= 20; i++) ticket.update(); }, "B").start();
    }
}

class Ticket {
    private int ticketCount = 20;

    Lock lock = new ReentrantLock();
    private Condition condition1 = lock.newCondition();
    private Condition condition2 = lock.newCondition();
    private int number = 1; // 1sale 2update


    public void sale() {
        lock.lock();    // 加锁
        try {
            // 业务， 判断-> 执行-> 通知
            
            // 等待应该总是出现在循环中，防止出现虚假唤醒，也就是被唤醒了却条件不满足还得等待
            while (number != 1) {
                condition1.await();
            }
            // 业务代码
            if (ticketCount > 0) {
                System.out.println(Thread.currentThread().getName() + "卖出了第" + (ticketCount--) + "张票");
            }
            // 唤醒指定线程
            number = 2;
            condition2.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();  // 解锁
        }
    }

    public void update() {
        lock.lock();    // 加锁
        try {
            while (number != 2) {
                condition2.await();
            }
            // 业务代码
            System.out.println(Thread.currentThread().getName() + "更新统计现有余票" + ticketCount + "张");

            // 唤醒指定线程
            number = 1;
            condition1.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();  // 解锁
        }
    }
}
```



## 3、多线程锁

**synchronized 锁的对象是方法的调用者**

**代码举例1：**

```java
package com.haust.lock8;
import java.util.concurrent.TimeUnit;
/**
 * 8锁，就是关于锁的8个问题
 * 1、标准情况下，两个线程先打印 发短信还是 先打印 打电话？ 1/发短信  2/打电话
 * 1、sendSms延迟4秒，两个线程先打印 发短信还是 打电话？ 1/发短信  2/打电话
 */.
public class Test1 {
    public static void main(String[] args) {
        Phone phone = new Phone();
        // 锁的存在
        new Thread(()->{
            phone.sendSms();
        },"A").start();
        // 捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(()->{
            phone.call();
        },"B").start();
    }
}
class Phone{
    // synchronized 锁的对象是方法的调用者！、
    // 两个方法用的是同一个对象调用(同一个锁)，谁先拿到锁谁执行！
    public synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);// 抱着锁睡眠
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }
    public synchronized void call(){
        System.out.println("打电话");
    }
}
// 先执行 发短信，后执行打电话
```

**普通方法没有锁！不是同步方法，就不受锁的影响，正常执行**

**代码举例2：**

```java
package com.haust.lock8;
import java.util.concurrent.TimeUnit;
/**
 * 3、 增加了一个普通方法后！先执行发短信还是Hello？// 普通方法
 * 4、 两个对象，两个同步方法， 发短信还是 打电话？ // 打电话
 */
public class Test2  {
    public static void main(String[] args) {
        // 两个对象，两个调用者，两把锁！
        Phone2 phone1 = new Phone2();
        Phone2 phone2 = new Phone2();
        //锁的存在
        new Thread(()->{
            phone1.sendSms();
        },"A").start();
        // 捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(()->{
            phone2.call();
        },"B").start();
        new Thread(()->{
            phone2.hello();
        },"C").start();
    }
}
class Phone2{
    // synchronized 锁的对象是方法的调用者！
    public synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }
    public synchronized void call(){
        System.out.println("打电话");
    }
    // 这里没有锁！不是同步方法，不受锁的影响
    public void hello(){
        System.out.println("hello");
    }
}
// 先执行打电话，接着执行hello，最后执行发短信
```

**不同实例对象的Class类模板只有一个，static静态的同步方法，锁的是Class **

**代码举例3：**

```java
package com.haust.lock8;
import java.util.concurrent.TimeUnit;
/**
 * 5、增加两个静态的同步方法，只有一个对象，先打印 发短信？打电话？
 * 6、两个对象！增加两个静态的同步方法， 先打印 发短信？打电话？
 */
public class Test3  {
    public static void main(String[] args) {
        // 两个对象的Class类模板只有一个，static，锁的是Class
        Phone3 phone1 = new Phone3();
        Phone3 phone2 = new Phone3();
        //锁的存在
        new Thread(()->{
            phone1.sendSms();
        },"A").start();
        // 捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(()->{
            phone2.call();
        },"B").start();
    }
}
// Phone3唯一的一个 Class 对象
class Phone3{
    // synchronized 锁的对象是方法的调用者！
    // static 静态方法
    // 类一加载就有了！锁的是Class
    public static synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    } 
    public static synchronized void call(){
        System.out.println("打电话");
    }
}
// 先执行发短信，后执行打电话
```

**代码举例4：**

```java
package com.haust.lock8;
import java.util.concurrent.TimeUnit;
/**
 * 7、1个静态的同步方法，1个普通的同步方法 ，一个对象，先打印 发短信？打电话？
 * 8、1个静态的同步方法，1个普通的同步方法 ，两个对象，先打印 发短信？打电话？
 */
public class Test4  {
    public static void main(String[] args) {
        // 两个对象的Class类模板只有一个，static，锁的是Class
        Phone4 phone1 = new Phone4();
        Phone4 phone2 = new Phone4();
        //锁的存在
        new Thread(()->{
            phone1.sendSms();
        },"A").start();
        // 捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(()->{
            phone2.call();
        },"B").start();
    }
}
// Phone3唯一的一个 Class 对象
class Phone4{
    // 静态的同步方法 锁的是 Class 类模板
    public static synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }
    // 普通的同步方法  锁的调用者(对象),二者锁的对象不同,所以不需要等待
    public synchronized void call(){
        System.out.println("打电话");
    }
}
// 7/8 两种情况下，都是先执行打电话,后执行发短信,因为二者锁的对象不同,
// 静态同步方法锁的是Class类模板,普通同步方法锁的是实例化的对象,
// 所以不用等待前者解锁后 后者才能执行,而是两者并行执行,因为发短信休眠4s
// 所以打电话先执行。
```

> 小结

- new this 具体的一个手机
- static Class 唯一的一个模板



## 4、集合类不安全

### 4.1 List不安全

List、ArrayList 等在并发多线程条件下，不能实现数据共享，多个线程同时调用一个list对象时候就会出现并发修改异常ConcurrentModificationException 

```java
package com.haust.unsafe;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;
// java.util.ConcurrentModificationException 并发修改异常！
public class ListTest {
    public static void main(String[] args) {
        // 并发下 ArrayList 不安全的吗，Synchronized；
        /*
         * 解决方案；
         * 方案1、List<String> list = new Vector<>();
         * 方案2、List<String> list =
         * Collections.synchronizedList(new ArrayList<>());
         * 方案3、List<String> list = new CopyOnWriteArrayList<>()；
         */
       /* CopyOnWrite 写入时复制  COW  计算机程序设计领域的一种优化策略；
        * 多个线程调用的时候，list，读取的时候，固定的，写入（覆盖）
        * 在写入的时候避免覆盖，造成数据问题！
        * 读写分离
        * CopyOnWriteArrayList  比 Vector Nb 在哪里？
        */    
        List<String> list = new CopyOnWriteArrayList<>();
        for (int i = 1; i <= 10; i++) {
            new Thread(()->{
                list.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(list);
            },String.valueOf(i)).start();
        }
    }
}
```



### 4.2 Set不安全

Set、Hash 等在并发多线程条件下，不能实现数据共享，多个线程同时调用一个set对象时候就会出现并发修改异常ConcurrentModificationException 

```java
package com.haust.unsafe;
import java.util.HashSet;
import java.util.Set;
import java.util.UUID;
/**
 * 同理可证 ： ConcurrentModificationException 并发修改异常
 * 1、Set<String> set = 
 *                     Collections.synchronizedSet(new HashSet<>());
 * 2、
 */
public class SetTest {
    public static void main(String[] args) {
        //Set<String> set = new HashSet<>();//不安全
        // Set<String> set = Collections.synchronizedSet(new HashSet<>());//安全
        Set<String> set = new CopyOnWriteArraySet<>();//安全
        for (int i = 1; i <=30 ; i++) {
           new Thread(()->{
               set.add(UUID.randomUUID().toString().substring(0,5));
               System.out.println(set);
           },String.valueOf(i)).start();
        }
    }
}
```

**扩展：hashSet 底层是什么？**

```java
// 可以看出HashSet的底层就是一个HashMap
public HashSet() {
    map = new HashMap<>();
}
// set.add(e) 本质就是利用 map key无法重复
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

private static final Object PRESENT = new Object();	// 不变值
```

### 4.3 Map不安全

同理Map也是线程不安全的，推荐使用 ConcurrentHashMap

```java
package com.haust.unsafe;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
// ConcurrentModificationException
public class  MapTest {
    public static void main(String[] args) {
        // map 是这样用的吗？ 不是，工作中不用 HashMap
        // 默认等价于什么？  new HashMap<>(16,0.75);
        // Map<String, String> map = new HashMap<>();
        // 扩展：研究ConcurrentHashMap的原理
        Map<String, String> map = new ConcurrentHashMap<>();
        for (int i = 1; i <=30; i++) {
            new Thread(()->{
                map.put(Thread.currentThread().getName(),
                       UUID.randomUUID().toString().substring(0,5));
                System.out.println(map);
            },String.valueOf(i)).start();
        }
    }
}
```



## 5、常用辅助类

### 5.1 Callable 和 Runnable

- **Callable** 是 java.util 包下 concurrent 下的接口，有返回值，可以抛出被检查的异常
- **Runable** 是 java.lang 包下的接口，没有返回值，不可以抛出被检查的异常
- 二者调用的方法不同，**run**()/ **call**()

同样的 **Lock** 和 **Synchronized** 二者的区别，前者是java.util 下的接口 后者是 java.lang 下的关键字。

```java
package com.haust.callable;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;
/**
 * 1、探究原理
 * 2、觉自己会用
 */
public class CallableTest {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // new Thread(new Runnable()).start();// 启动Runnable
        // new Thread(new FutureTask<V>()).start();
        // new Thread(new FutureTask<V>( Callable )).start();
        new Thread().start(); // 怎么启动Callable？
        // new 一个MyThread实例
        MyThread thread = new MyThread();
        // MyThread实例放入FutureTask
        FutureTask futureTask = new FutureTask(thread); // 适配类
        new Thread(futureTask,"A").start();
        new Thread(futureTask,"B").start(); // call()方法结果会被缓存，提高效率，因此只打印1个call
        // 这个get 方法可能会产生阻塞！把他放到最后
        Integer o = (Integer) futureTask.get(); 
        // 或者使用异步通信来处理！
        System.out.println(o);// 1024
    }
}
class MyThread implements Callable<Integer> {
    @Override
    public Integer call() {
        System.out.println("call()"); // A,B两个线程会打印几个call？（1个）
        // 耗时的操作
        return 1024;
    }
}
//class MyThread implements Runnable {
//
//    @Override
//    public void run() {
//        System.out.println("run()"); // 会打印几个run
//    }
//}
```

**细节：**

1、有缓存

2、结果可能需要等待，会阻塞！

### 5.2 CountDownLatch

![img](pics\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzU5MTk4MA==,size_16,color_FFFFFF,t_70)



**减法计数器： 实现调用几次线程后 再触发某一个任务**

```java
package com.haust.add;    
import java.util.concurrent.CountDownLatch;
// 计数器
public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        // 总数是6，必须要执行任务的时候，再使用！
        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 1; i <=6 ; i++) {
            new Thread(()->{
                System.out.println(Thread.currentThread().getName()
                                                           +" Go out");
                countDownLatch.countDown(); // 数量-1
            },String.valueOf(i)).start();
        }
        countDownLatch.await(); // 等待计数器归零，然后再向下执行
        System.out.println("Close Door");
    }
}
```

原理：

`countDownLatch.countDown();` // 数量-1

`countDownLatch.await();` // 等待计数器归零，然后再向下执行

每次有线程调用 **countDown**() 数量-1，假设计数器变为0，**countDownLatch.await**() 就会被唤醒，继续执行！

### 5.3 CyclicBarrier

![img](pics\122.png)



**加法计数器: **

```java
package com.haust.add;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
public class  CyclicBarrierDemo {
    public static void main(String[] args) {
        /*
         * 集齐7颗龙珠召唤神龙
         */
        // 召唤龙珠的线程
        CyclicBarrier cyclicBarrier = new CyclicBarrier(7,()->{
            System.out.println("召唤神龙成功！");
        });
        for (int i = 1; i <=7 ; i++) {
            final int temp = i;
            // lambda能操作到 i 吗
            new Thread(()->{
                System.out.println(Thread.currentThread().getName()
                                                 +"收集"+temp+"个龙珠");
                try {
                    cyclicBarrier.await(); // 等待
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

### 5.4 Semaphore

![img](pics\10211.png)

```java
package com.haust.add;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;
public class SemaphoreDemo {
    public static void main(String[] args) {
        // 线程数量：停车位! 限流！、
        // 如果已有3个线程执行（3个车位已满），则其他线程需要等待‘车位’释放后，才能执行！
        Semaphore semaphore = new Semaphore(3);
        for (int i = 1; i <=6 ; i++) {
            new Thread(()->{
                // acquire() 得到
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread()
                                               .getName()+"抢到车位");
                    TimeUnit.SECONDS.sleep(2);
                    System.out.println(Thread.currentThread()
                                               .getName()+"离开车位");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release(); // release() 释放 
                }
            },String.valueOf(i)).start();
        }
    }
}
```

只有三个车位，只有当某辆车离开车位，车位空出来后，下一辆车才能在此停放。

输出结果如图:

![img](pics\j01.png)

**原理：**

`semaphore.acquire();` 获得，假设如果已经满了，等待，等待被释放为止！`semaphore.release();` 释放，会将当前的信号量释放 + 1，然后唤醒等待的线程！

作用： 多个共享资源互斥的使用！并发限流，控制最大的线程数！



## 6、读写锁 ReadWriteLock

![img](pics\j02.png)

```java
package com.haust.rw;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;
/**
 * 独占锁（写锁） 一次只能被一个线程占有
 * 共享锁（读锁） 多个线程可以同时占有
 * ReadWriteLock
 * 读-读  可以共存！
 * 读-写  不能共存！
 * 写-写  不能共存！
 */
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        //MyCache myCache = new MyCache();
        MyCacheLock myCacheLock = new MyCacheLock();
        // 写入
        for (int i = 1; i <= 5 ; i++) {
            final int temp = i;
            new Thread(()->{
                myCacheLock.put(temp+"",temp+"");
            },String.valueOf(i)).start();
        }
        // 读取
        for (int i = 1; i <= 5 ; i++) {
            final int temp = i;
            new Thread(()->{
                myCacheLock.get(temp+"");
            },String.valueOf(i)).start();
        }
    }
}
/**
 * 自定义缓存
 * 加锁的
 */
class MyCacheLock{
    private volatile Map<String,Object> map = new HashMap<>();
    // 读写锁： 更加细粒度的控制
    private ReadWriteLock readWriteLock = new             
                                    ReentrantReadWriteLock();
    // private Lock lock = new ReentrantLock();
    // 存，写入的时候，只希望同时只有一个线程写
    public void put(String key,Object value){
        readWriteLock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName()
                                                       +"写入"+key);
            map.put(key,value);
            System.out.println(Thread.currentThread().getName()
                                                       +"写入OK");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            readWriteLock.writeLock().unlock();
        }
    }
    // 取，读，所有人都可以读！
    public void get(String key){
        readWriteLock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName()
                                                       +"读取"+key);
            Object o = map.get(key);
            System.out.println(Thread.currentThread().getName()
                                                       +"读取OK");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            readWriteLock.readLock().unlock();
        }
    }
}
/**
 * 自定义缓存
 * 不加锁的
 */
class MyCache{
    private volatile Map<String,Object> map = new HashMap<>();
    // 存，写
    public void put(String key,Object value){
        System.out.println(Thread.currentThread().getName()
                                                       +"写入"+key);
        map.put(key,value);
        System.out.println(Thread.currentThread().getName()
                                                       +"写入OK");
    }
    // 取，读
    public void get(String key){
        System.out.println(Thread.currentThread().getName()
                                                       +"读取"+key);
        Object o = map.get(key);
        System.out.println(Thread.currentThread().getName()
                                                       +"读取OK");
    }
}
```

执行效果如图：

![img](pics\j03.png)



## 7、阻塞队列

![img](pics\j04.png)

**阻塞队列：**

![img](pics\j05.png)

![img](pics\j06.png)



> BlockingQueue

![在这里插入图片描述](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\j07.png)

什么情况下我们会使用 阻塞队列?：多线程并发处理，线程池用的较多 ！

**学会使用队列**：添加、移除

> **四组API**

| 方式         | 抛出异常 | 有返回值，不抛出异常 | 阻塞 等待 | 超时等待 |
| ------------ | -------- | -------------------- | --------- | -------- |
| 添加         | add      | offer()              | put()     | offer(,) |
| 移除         | remove   | poll()               | take()    | poll(,)  |
| 检测队首元素 | element  | peek()               | -         | -        |

```java
package com.kuang.bq;
import java.util.Collection;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.TimeUnit;
public class Test {
    public static void main(String[] args) throws InterruptedException {
        test4();
    }
    /**
     * 1. 无返回值，抛出异常的方式
     */
    public static void test1(){
        // 队列的大小
        ArrayBlockingQueue blockingQueue = 
                                    new ArrayBlockingQueue<>(3);
        System.out.println(blockingQueue.add("a"));// true
        System.out.println(blockingQueue.add("b"));// true
        System.out.println(blockingQueue.add("c"));// true
        // System.out.println(blockingQueue.add("d"));
        // IllegalStateException: Queue full 抛出异常---队列已满！
        System.out.println("===========================");
        System.out.println(blockingQueue.element());//
        // 查看队首元素是谁
        System.out.println(blockingQueue.remove());//
        System.out.println(blockingQueue.remove());//
        System.out.println(blockingQueue.remove());//
        // System.out.println(blockingQueue.remove());
        // java.util.NoSuchElementException 抛出异常---队列已为空！
    }
    /**
     * 2. 有返回值，不抛出异常的方式
     */
    public static void test2(){
        // 队列的大小
        ArrayBlockingQueue blockingQueue = 
                                    new ArrayBlockingQueue<>(3);
        System.out.println(blockingQueue.offer("a"));
        System.out.println(blockingQueue.offer("b"));
        System.out.println(blockingQueue.offer("c"));
        System.out.println(blockingQueue.peek());
        // System.out.println(blockingQueue.offer("d")); 
        // false 不抛出异常！
        System.out.println("===========================");
        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll()); 
        // null  不抛出异常！
    }
    /**
     * 3. 等待，阻塞（一直阻塞）
     */
    public static void test3() throws InterruptedException {
        // 队列的大小
        ArrayBlockingQueue blockingQueue = 
                                    new ArrayBlockingQueue<>(3);
        // 一直阻塞
        blockingQueue.put("a");
        blockingQueue.put("b");
        blockingQueue.put("c");
        // blockingQueue.put("d"); // 队列没有位置了，一直阻塞等待
        System.out.println(blockingQueue.take());
        System.out.println(blockingQueue.take());
        System.out.println(blockingQueue.take());
        System.out.println(blockingQueue.take()); 
        // 没有这个元素，一直阻塞等待
    }
    /**
     * 4. 等待，阻塞（等待超时）
     */
    public static void test4() throws InterruptedException {
        // 队列的大小
        ArrayBlockingQueue blockingQueue = 
                                    new ArrayBlockingQueue<>(3);
        blockingQueue.offer("a");
        blockingQueue.offer("b");
        blockingQueue.offer("c");
        // blockingQueue.offer("d",2,TimeUnit.SECONDS); 
        // 等待超过2秒就退出
        System.out.println("===============");
        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll());
        blockingQueue.poll(2,TimeUnit.SECONDS); // 等待超过2秒就退出
    }
}
```

#### SynchronousQueue

> SynchronousQueue 同步队列

**没有容量，进去一个元素，必须等待取出来之后，才能再往里面放一个元素！** put、take

```java
package com.haust.bq;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.TimeUnit;
/**
 * 同步队列:
 * 和其他的BlockingQueue 不一样， SynchronousQueue 不存储元素
 * put了一个元素，必须从里面先take取出来，否则不能在put进去值！
 */
public class SynchronousQueueDemo {
    public static void main(String[] args) {
        BlockingQueue<String> blockingQueue = 
                                new SynchronousQueue<>(); // 同步队列
        new Thread(()->{
            try {
                System.out.println(Thread.currentThread().getName()
                                                           +" put 1");
                // put进入一个元素
                blockingQueue.put("1");
                System.out.println(Thread.currentThread().getName()
                                                           +" put 2");
                blockingQueue.put("2");
                System.out.println(Thread.currentThread().getName()
                                                           +" put 3");
                blockingQueue.put("3");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"T1").start();
        new Thread(()->{
            try {
                // 睡眠3s取出一个元素
                TimeUnit.SECONDS.sleep(3);
                System.out.println(Thread.currentThread().getName()
                                           +"=>"+blockingQueue.take());
                TimeUnit.SECONDS.sleep(3);
                System.out.println(Thread.currentThread().getName()
                                           +"=>"+blockingQueue.take());
                TimeUnit.SECONDS.sleep(3);
                System.out.println(Thread.currentThread().getName()
                                           +"=>"+blockingQueue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"T2").start();
    }
}
```

执行结果如图所示：

![在这里插入图片描述](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\j08.png)



## 8、线程池

**线程池：3大方法、7大参数、4种拒绝策略**

> 池化技术

程序的运行，本质：占用系统的资源！ （优化资源的使用 => 池化技术）

线程池、连接池、内存池、对象池///… 创建、销毁。十分浪费资源

池化技术：事先准备好一些资源，有人要用，就来我这里拿，用完之后还给我。

> 线程池的好处:

- 1、降低系统资源的消耗
- 2、提高响应的速度
- 3、方便管理

**线程复用、可以控制最大并发数、管理线程**

#### 线程池：3大方法

> 线程池：3大方法

![img](pics\j09.png)

```java
package com.haust.pool;
import java.util.concurrent.ExecutorService;
import java.util.List;
import java.util.concurrent.Executors;
public class Demo01 {
    public static void main(String[] args) {
        // Executors 工具类、3大方法
        // Executors.newSingleThreadExecutor();// 创建单个线程的线程池
        // Executors.newFixedThreadPool(5);// 创建一个固定大小的线程池
        // Executors.newCachedThreadPool();// 创建一个可伸缩的线程池
        // 单个线程的线程池
        ExecutorService threadPool =     
                                Executors.newSingleThreadExecutor();
        try {
            for (int i = 1; i < 100; i++) {
                // 使用了线程池之后，使用线程池来创建线程
                threadPool.execute(()->{
                    System.out.println(
                        Thread.currentThread().getName()+" ok");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 线程池用完，程序结束，关闭线程池
            threadPool.shutdown();
        }
    }
}
```

#### 线程池：7大参数

> 7大参数

源码分析：

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService (
        new ThreadPoolExecutor(
            1, 
            1,
            0L, 
            TimeUnit.MILLISECONDS, 
            new LinkedBlockingQueue<Runnable>())); 
}
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(
        5, 
        5, 
        0L, 
        TimeUnit.MILLISECONDS, 
        new LinkedBlockingQueue<Runnable>()); 
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(
        0, 
        Integer.MAX_VALUE, 
        60L, 
        TimeUnit.SECONDS, 
        new SynchronousQueue<Runnable>()); 
}
// 本质ThreadPoolExecutor（） 
public ThreadPoolExecutor(int corePoolSize, // 核心线程池大小 
                          int maximumPoolSize, // 最大核心线程池大小 
                          long keepAliveTime, // 超时没有人调用就会释放 
                          TimeUnit unit, // 超时单位 
                          // 阻塞队列 
                          BlockingQueue<Runnable> workQueue, 
                          // 线程工厂：创建线程的，一般 不用动
                          ThreadFactory threadFactory,  
                          // 拒绝策略
                          RejectedExecutionHandler handle ) {
    if (corePoolSize < 0 
        || maximumPoolSize <= 0 
        || maximumPoolSize < corePoolSize 
        || keepAliveTime < 0) 
        throw new IllegalArgumentException(); 
    if (workQueue == null 
        || threadFactory == null 
        || handler == null) 
        throw new NullPointerException(); 
    this.acc = System.getSecurityManager() == null 
        ? null : AccessController.getContext(); 
    this.corePoolSize = corePoolSize; 
    this.maximumPoolSize = maximumPoolSize; 
    this.workQueue = workQueue; 
    this.keepAliveTime = unit.toNanos(keepAliveTime); 
    this.threadFactory = threadFactory; 
    this.handler = handler; 
}
```

![img](pics\j10.png)

![img](pics\j11.png)

> 手动创建一个线程池

因为实际开发中工具类**Executors** 不安全，所以需要手动创建线程池，自定义7个参数。

```java
package com.haust.pool;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;
// Executors 工具类、3大方法
// Executors.newSingleThreadExecutor();// 创建一个单个线程的线程池
// Executors.newFixedThreadPool(5);// 创建一个固定大小的线程池
// Executors.newCachedThreadPool();// 创建一个可伸缩的线程池
/**
 * 四种拒绝策略：
 *
 * new ThreadPoolExecutor.AbortPolicy() 
 * 银行满了，还有人进来，不处理这个人的，抛出异常
 *
 * new ThreadPoolExecutor.CallerRunsPolicy() 
 * 哪来的去哪里！比如你爸爸 让你去通知妈妈洗衣服，妈妈拒绝，让你回去通知爸爸洗
 *
 * new ThreadPoolExecutor.DiscardPolicy() 
 * 队列满了，丢掉任务，不会抛出异常！
 *
 * new ThreadPoolExecutor.DiscardOldestPolicy() 
 * 队列满了，尝试去和最早的竞争，也不会抛出异常！
 */
public class Demo01 {
    public static void main(String[] args) {
        // 自定义线程池！工作 ThreadPoolExecutor
        ExecutorService threadPool = new ThreadPoolExecutor(
                2,// int corePoolSize, 核心线程池大小(候客区窗口2个)
                5,// int maximumPoolSize, 最大核心线程池大小(总共5个窗口) 
                3,// long keepAliveTime, 超时3秒没有人调用就会释，放关闭窗口 
                TimeUnit.SECONDS,// TimeUnit unit, 超时单位 秒 
                new LinkedBlockingDeque<>(3),// 阻塞队列(候客区最多3人)
                Executors.defaultThreadFactory(),// 默认线程工厂
                // 4种拒绝策略之一：
                // 队列满了，尝试去和 最早的竞争，也不会抛出异常！
                new ThreadPoolExecutor.DiscardOldestPolicy());  
        //队列满了，尝试去和最早的竞争，也不会抛出异常！
        try {
            // 最大承载：Deque + max
            // 超过 RejectedExecutionException
            for (int i = 1; i <= 9; i++) {
                // 使用了线程池之后，使用线程池来创建线程
                threadPool.execute(()->{
                    System.out.println(
                        Thread.currentThread().getName()+" ok");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 线程池用完，程序结束，关闭线程池
            threadPool.shutdown();
        }
    }
}
```

#### 线程池：4种拒绝策略

> 4种拒绝策略

![img](pics\j12.png)

```java
/**
 * 四种拒绝策略：
 *
 * new ThreadPoolExecutor.AbortPolicy() 
 * 银行满了，还有人进来，不处理这个人的，抛出异常
 *
 * new ThreadPoolExecutor.CallerRunsPolicy() 
 * 哪来的去哪里！比如你爸爸 让你去通知妈妈洗衣服，妈妈拒绝，让你回去通知爸爸洗
 *
 * new ThreadPoolExecutor.DiscardPolicy() 
 * 队列满了，丢掉任务，不会抛出异常！
 *
 * new ThreadPoolExecutor.DiscardOldestPolicy() 
 * 队列满了，尝试去和最早的竞争，也不会抛出异常！
 */
```

> 小结和拓展

池的最大容量如何去设置！

了解：**IO密集型，CPU密集型：（调优）**

```java
package com.haust.pool;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;
public class Demo01 {
    public static void main(String[] args) {
        // 自定义线程池！工作 ThreadPoolExecutor
        // 最大线程到底该如何定义
        // 1、CPU 密集型，几核，就是几，可以保持CPu的效率最高！ 
        // 2、IO 密集型 > 判断你程序中十分耗IO的线程， 
        // 比如程序 15个大型任务 io十分占用资源！
        // IO密集型参数(最大线程数)就设置为大于15即可，一般选择两倍
        // 获取CPU的核数
        System.out.println(
            Runtime.getRuntime().availableProcessors());// 8核
        ExecutorService threadPool = new ThreadPoolExecutor(
                2,// int corePoolSize, 核心线程池大小
                // int maximumPoolSize, 最大核心线程池大小 8核电脑就是8
                Runtime.getRuntime().availableProcessors(),
                3,// long keepAliveTime, 超时3秒没有人调用就会释放
                TimeUnit.SECONDS,// TimeUnit unit, 超时单位 秒 
                new LinkedBlockingDeque<>(3),// 阻塞队列(候客区最多3人)
                Executors.defaultThreadFactory(),// 默认线程工厂
                // 4种拒绝策略之一：
                // 队列满了，尝试去和 最早的竞争，也不会抛出异常！
                new ThreadPoolExecutor.DiscardOldestPolicy());  
        //队列满了，尝试去和最早的竞争，也不会抛出异常！
        try {
            // 最大承载：Deque + max
            // 超过 RejectedExecutionException
            for (int i = 1; i <= 9; i++) {
                // 使用了线程池之后，使用线程池来创建线程
                threadPool.execute(()->{
                    System.out.println(
                        Thread.currentThread().getName()+" ok");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 线程池用完，程序结束，关闭线程池
            threadPool.shutdown();
        }
    }
}
```



## 9、四大函数式接口

新时代的程序员：lambda表达式、链式编程、函数式接口、Stream流式计算

> 函数式接口： 只有一个方法的接口

```java
@FunctionalInterface 
public interface Runnable {
    public abstract void run(); 
}
// 泛型、枚举、反射 
// lambda表达式、链式编程、函数式接口、Stream流式计算 
// 超级多FunctionalInterface 
// 简化编程模型，在新版本的框架底层大量应用！ 
// foreach(消费者类的函数式接口)
```

![img](pics\20210130190944452.png)



> Function 函数式接口

![img](pics\j13.png)

```java
package com.haust.function;
import java.util.function.Function;
/**
 * Function 函数型接口, 有一个输入参数，有一个输出参数
 * 只要是 函数型接口 可以 用 lambda表达式简化
 */
public class Demo01 {
    public static void main(String[] args) {
       /*Function<String,String> function = new 
                                        Function<String,String>() {
            @Override
            public String apply(String str) {
                return str;
            }
        };*/
        // lambda 表达式简化：
        Function<String,String> function = str->{
    return str;};
        System.out.println(function.apply("asd"));
    }
}
```

> Predicate 断定型接口：有一个输入参数，返回值只能是 布尔值！

![img](pics\j14.png)

```java
package com.haust.function;
import java.util.function.Predicate;
/*
 * 断定型接口：有一个输入参数，返回值只能是 布尔值！
 */
public class Demo02 {
    public static void main(String[] args) {
        // 判断字符串是否为空
        /*Predicate<String> predicate = new Predicate<String>(){
            @Override
            public boolean test(String str) {
                return str.isEmpty();//true或false
            }
        };*/
        Predicate<String> predicate = 
                            (str)->{
    return str.isEmpty(); };
        System.out.println(predicate.test(""));//true
    }
}
```



> Consumer 消费型接口

![img](pics\j15.png)

```java
package com.haust.function;
import java.util.function.Consumer;
/**
 * Consumer 消费型接口: 只有输入，没有返回值
 */
public class Demo03 {
    public static void main(String[] args) {
        /*Consumer<String> consumer = new Consumer<String>() {
            @Override
            public void accept(String str) {
                System.out.println(str);
            }
        };*/
        Consumer<String> consumer = 
                                (str)->{
    System.out.println(str);};
        consumer.accept("sdadasd");
    }
}
```

> Supplier 供给型接口

![在这里插入图片描述](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\j16.png)

```java
package com.haust.function;
import java.util.function.Supplier;
/**
 * Supplier 供给型接口 没有参数，只有返回值
 */
public class Demo04 {
    public static void main(String[] args) {
        /*Supplier supplier = new Supplier<Integer>() {
            @Override
            public Integer get() {
                System.out.println("get()");
                return 1024;
            }
        };*/
        Supplier supplier = ()->{
     return 1024; };
        System.out.println(supplier.get());
    }
}
```



## 10、Stream流式计算

![img](pics\20210130191049934.png)

```java
package com.haust.stream;
import java.util.Arrays;
import java.util.List;
/**
 * 题目要求：一分钟内完成此题，只能用一行代码实现！
 * 现在有5个用户！筛选：
 * 1、ID 必须是偶数
 * 2、年龄必须大于23岁
 * 3、用户名转为大写字母
 * 4、用户名字母倒着排序
 * 5、只输出一个用户！
 */
public class Test {
    public static void main(String[] args) {
        User u1 = new User(1,"a",21);
        User u2 = new User(2,"b",22);
        User u3 = new User(3,"c",23);
        User u4 = new User(4,"d",24);
        User u5 = new User(6,"e",25);
        // 集合就是存储
        List<User> list = Arrays.asList(u1, u2, u3, u4, u5);
        // 计算交给Stream流
        // lambda表达式、链式编程、函数式接口、Stream流式计算
        list.stream()
                .filter(u->{
    return u.getId()%2==0;})// ID 必须是偶数
                .filter(u->{
    return u.getAge()>23;})// 年龄必须大于23岁
                // 用户名转为大写字母
                .map(u->{
    return u.getName().toUpperCase();})
                // 用户名字母倒着排序
                .sorted((uu1,uu2)->{
    return uu2.compareTo(uu1);})
                .limit(1)// 只输出一个用户！
                .forEach(System.out::println);
    }
}
```



## 11、ForkJoin

ForkJoin 在 JDK 1.7 ， 并行执行任务！提高效率。大数据量！

大数据：Map Reduce （把大任务拆分为小任务）

![img](pics\j17.png)

> ForkJoin 特点：工作窃取

这个里面维护的都是双端队列

![img](pics\j18.png)

> ForkJoin

![img](pics\20210130191412266.png)

![img](pics\j19.png)

```java
package com.haust.forkjoin;
import java.util.concurrent.RecursiveTask;
/**
 * 求和计算的任务！
 * 3000   6000（ForkJoin）  9000（Stream并行流）
 * // 如何使用 forkjoin
 * // 1、forkjoinPool 通过它来执行
 * // 2、计算任务 forkjoinPool.execute(ForkJoinTask task)
 * // 3. 计算类要继承 RecursiveTask(递归任务，有返回值的)
 */
public class ForkJoinDemo extends RecursiveTask<Long> {
    private Long start;  // 1
    private Long end;    // 1990900000
    // 临界值
    private Long temp = 10000L;
    public ForkJoinDemo(Long start, Long end) {
        this.start = start;
        this.end = end;
    }
    // 计算方法
    @Override
    protected Long compute() {
        if ((end-start)<temp){
            Long sum = 0L;
            for (Long i = start; i <= end; i++) {
                sum += i;
            }
            return sum;
        }else {
     // forkjoin 递归
            long middle = (start + end) / 2; // 中间值
            ForkJoinDemo task1 = new ForkJoinDemo(start, middle);
            task1.fork(); // 拆分任务，把任务压入线程队列
            ForkJoinDemo task2 = new ForkJoinDemo(middle+1, end);
            task2.fork(); // 拆分任务，把任务压入线程队列
            return task1.join() + task2.join();
        }
    }
}
```

测试代码：

```java
package com.haust.forkjoin;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.ForkJoinTask;
import java.util.stream.LongStream;
/**
 * 同一个任务，别人效率高你几十倍！
 */
public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // test1(); // 12224
        // test2(); // 10038
        // test3(); // 153
    }
    // 普通程序员
    public static void test1(){
        Long sum = 0L;
        long start = System.currentTimeMillis();
        for (Long i = 1L; i <= 10_0000_0000; i++) {
            sum += i;
        }
        long end = System.currentTimeMillis();
        System.out.println("sum="+sum+" 时间："+(end-start));
    }
    // 会使用ForkJoin
    public static void test2() throws ExecutionException, InterruptedException {
        long start = System.currentTimeMillis();
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        ForkJoinTask<Long> task = new ForkJoinDemo(
                                                0L, 10_0000_0000L);
        // 提交任务
        ForkJoinTask<Long> submit = forkJoinPool.submit(task);
        Long sum = submit.get();// 获得结果
        long end = System.currentTimeMillis();
        System.out.println("sum="+sum+" 时间："+(end-start));
    }
    public static void test3(){
        long start = System.currentTimeMillis();
        // Stream并行流 ()  (]
        long sum = LongStream
            .rangeClosed(0L, 10_0000_0000L) // 计算范围(,]
            .parallel() // 并行计算
            .reduce(0, Long::sum); // 输出结果
        long end = System.currentTimeMillis();
        System.out.println("sum="+"时间："+(end-start));
    }
}
```



## 12、异步回调

> Future 设计的初衷： 对将来的某个事件的结果进行建模

![img](pics\j20.png)

```java
package com.haust.future;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
/**
 * 异步调用： CompletableFuture
 * 异步执行
 * 成功回调
 * 失败回调
 */
public class Demo01 {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // 没有返回值的 runAsync 异步回调
//        CompletableFuture<Void> completableFuture = 
//                                    CompletableFuture.runAsync(()->{
//            try {
//                TimeUnit.SECONDS.sleep(2);
//            } catch (InterruptedException e) {
//                e.printStackTrace();
//            }
//            System.out.println(
//                Thread.currentThread().getName()+"runAsync=>Void");
//        });
//
//        System.out.println("1111");
//
//        completableFuture.get(); // 获取阻塞执行结果
        // 有返回值的 supplyAsync 异步回调
        // ajax，成功和失败的回调
        // 返回的是错误信息；
        CompletableFuture<Integer> completableFuture = 
                                CompletableFuture.supplyAsync(()->{
            System.out.println(Thread.currentThread().getName()
                                           +"supplyAsync=>Integer");
            int i = 10/0;
            return 1024;
        });
        System.out.println(completableFuture.whenComplete((t, u) -> {
            System.out.println("t=>" + t); // 正常的返回结果
            System.out.println("u=>" + u); 
            // 错误信息：
            // java.util.concurrent.CompletionException: 
            // java.lang.ArithmeticException: / by zero
        }).exceptionally((e) -> {
            System.out.println(e.getMessage());
            return 233; // 可以获取到错误的返回结果
        }).get());
        /**
         * succee Code 200
         * error Code 404 500
         */
    }
}
```



## 13、JMM

> 请你谈谈你对 Volatile 的理解

**Volatile** 是 Java 虚拟机提供**轻量级的同步机制**，类似于**synchronized** 但是没有其强大。

1、保证可见性

**2、不保证原子性**

3、防止指令重排

> 什么是JMM

JMM ： Java内存模型，不存在的东西，概念！约定！

**关于JMM的一些同步的约定：**

1、线程解锁前，必须把共享变量**立刻**刷回主存。

2、线程加锁前，必须读取主存中的最新值到工作内存中！

3、加锁和解锁是同一把锁。

线程 **工作内存** 、**主内存**

**8 种操作：**

![img](pics\j21.png)

![img](pics\j22.png)

**内存交互操作有8种，虚拟机实现必须保证每一个操作都是原子的，不可在分的（对于double和long类型的变量来说，load、store、read和writ操作在某些平台上允许例外）**

- lock （锁定）：作用于主内存的变量，把一个变量标识为线程独占状态
- unlock （解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
- read （读取）：作用于主内存变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load动作使用
- load （载入）：作用于工作内存的变量，它把read操作从主存中变量放入工作内存中
- use （使用）：作用于工作内存中的变量，它把工作内存中的变量传输给执行引擎，每当虚拟机遇到一个需要使用到变量的值，就会使用到这个指令
- assign （赋值）：作用于工作内存中的变量，它把一个从执行引擎中接受到的值放入工作内存的变量副本中
- store （存储）：作用于主内存中的变量，它把一个从工作内存中一个变量的值传送到主内存中，以便后续的write使用
- write （写入）：作用于主内存中的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中

**JMM 对这八种指令的使用，制定了如下规则：**

- 不允许read和load、store和write操作之一单独出现。即使用了read必须load，使用了store必须write
- 不允许线程丢弃他最近的assign操作，即工作变量的数据改变了之后，必须告知主存
- 不允许一个线程将没有assign的数据从工作内存同步回主内存
- 一个新的变量必须在主内存中诞生，不允许工作内存直接使用一个未被初始化的变量。就是怼变量实施use、store操作之前，必须经过assign和load操作
- 一个变量同一时间只有一个线程能对其进行lock。多次lock后，必须执行相同次数的unlock才能解锁
- 如果对一个变量进行lock操作，会清空所有工作内存中此变量的值，在执行引擎使用这个变量前，必须重新load或assign操作初始化变量的值
- 如果一个变量没有被lock，就不能对其进行unlock操作。也不能unlock一个被其他线程锁住的变量
- 对一个变量进行unlock操作之前，必须把此变量同步回主内存

**问题： 程序不知道主内存的值已经被修改过了**

![img](pics\j23.png)

## 14、Volatile

> 1、保证可见性

```java
package com.haust.tvolatile;
import java.util.concurrent.TimeUnit;
public class JMMDemo {
    // 不加 volatile 程序就会死循环！
    // 加 volatile 可以保证可见性
    private volatile static int num = 0;
    public static void main(String[] args) {
     // main
        new Thread(()->{
     // 线程 1 对主内存的变化不知道的
            while (num==0){
            }
        }).start();
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        num = 1;
        System.out.println(num);
    }
}
```

> 不保证原子性

原子性 : 不可分割

线程A在执行任务的时候，不能被打扰的，也不能被分割。要么同时成功，要么同时失败。

```java
package com.haust.tvolatile;
import java.util.concurrent.atomic.AtomicInteger;
// volatile 不保证原子性
public class VDemo02 {
    // volatile 不保证原子性
    // 原子类的 Integer
    private volatile static AtomicInteger num = new AtomicInteger();
    public static void add(){
        // num++; // 不是一个原子性操作
        num.getAndIncrement(); // AtomicInteger + 1 方法， CAS
    }
    public static void main(String[] args) {
        //理论上num结果应该为 2 万
        for (int i = 1; i <= 20; i++) {
            new Thread(()->{
                for (int j = 0; j < 1000 ; j++) {
                    add();
                }
            }).start();
        }
        // 判断只要剩下的线程不大于2个，就说明20个创建的线程已经执行结束
        while (Thread.activeCount()>2){
     // Java 默认有 main gc 2个线程
            Thread.yield();
        }
        System.out.println(Thread.currentThread().getName() 
                                                       + " " + num);
    }
}
```

**如果不加** **lock** **和** **synchronized** **，怎么样保证原子性**

![img](pics\j24.png)



使用原子类，解决原子性问题。

![img](pics\j25.png)

```java
// volatile 不保证原子性
 // 原子类的 Integer
 private volatile static AtomicInteger num = new AtomicInteger();
 public static void add(){
    // num++; // 不是一个原子性操作
    num.getAndIncrement(); // AtomicInteger + 1 方法， CAS
 }
```

这些类的底层都直接和操作系统挂钩！在内存中修改值！Unsafe类是一个很特殊的存在！



> 禁止指令重排

什么是指令重排？：**我们写的程序，计算机并不是按照你写的那样去执行的。**

源代码 —> 编译器优化的重排 —> 指令并行也可能会重排 —> 内存系统也会重排 ——> 执行

**处理器在执行指令重排的时候，会考虑：数据之间的依赖性**

```java
int x = 1; // 1
int y = 2; // 2
x = x + 5; // 3
y = x * x; // 4
```

我们所期望的：1234 但是可能执行的时候会变成 2134 或者 1324

但是不可能是 4123！

前提：a b x y 这四个值默认都是 0：

可能造成影响得到不同的结果：

| 线程A | 线程B |
| ----- | ----- |
| x = a | y = b |
| b =1  | a = 2 |

正常的结果：x = 0; y = 0; 但是可能由于指令重排出现以下结果：

| 线程A | 线程B |
| ----- | ----- |
| b = 1 | a = 2 |
| x = a | y = b |

指令重排导致的诡异结果： x = 2; y = 1;

**volatile** 可以避免指令重排：

内存屏障。CPU指令。作用：

1. 保证特定操作的执行顺序！
2. 可以保证某些变量的内存可见性 (利用这些特性**volatile** 实现了可见性)

![在这里插入图片描述](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\j26.png)

**volatile 是可以保证可见性。不能保证原子性，由于内存屏障，可以保证避免指令重排的现象产生！**

**volatile 内存屏障在单例模式中使用的最多！**



## 15、多线程中的单例模式

> 饿汉式

```java
package com.haust.single;
// 饿汉式单例
public class Hungry {
    // 可能会浪费空间
    private byte[] data1 = new byte[1024*1024];
    private byte[] data2 = new byte[1024*1024];
    private byte[] data3 = new byte[1024*1024];
    private byte[] data4 = new byte[1024*1024];
    private Hungry(){
    }
    private final static Hungry HUNGRY = new Hungry();
    public static Hungry getInstance(){
        return HUNGRY;
    }
}
```

> DCL 懒汉式

```java
package com.haust.single;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
// 懒汉式单例
// 道高一尺，魔高一丈！
public class LazyMan {
    private static boolean csp = false;// 标志位
    // 单例不安全，因为反射可以破坏单例，如下解决这个问题：
    private LazyMan(){
        synchronized (LazyMan.class){
            if (csp == false){
                csp = true;
            }else {
                throw new RuntimeException("不要试图使用反射破坏异常");
            }
        }
    }
    /**
     * 计算机指令执行顺序：
     * 1. 分配内存空间
     * 2、执行构造方法，初始化对象
     * 3、把这个对象指向这个空间
     *
     * 期望顺序是：123
     * 特殊情况下实际执行：132  ===>  此时 A 线程没有问题
     *                               若额外加一个 B 线程 
     *                               此时lazyMan还没有完成构造
     */
    // 原子性操作：避免指令重排
    private volatile static LazyMan lazyMan;
    // 双重检测锁模式的 懒汉式单例  DCL懒汉式
    public static LazyMan getInstance(){
        if (lazyMan==null){
            synchronized (LazyMan.class){
                if (lazyMan==null){
                    lazyMan = new LazyMan(); // 不是一个原子性操作
                }
            }
        }
        return lazyMan;
    }
    // 反射！
    public static void main(String[] args) throws Exception {
        //LazyMan instance = LazyMan.getInstance();
        Field qinjiang = LazyMan.class.getDeclaredField("csp");
        csp.setAccessible(true);
        Constructor<LazyMan> declaredConstructor = 
                        LazyMan.class.getDeclaredConstructor(null);
        declaredConstructor.setAccessible(true);
        LazyMan instance = declaredConstructor.newInstance();
        qinjiang.set(instance,false);
        LazyMan instance2 = declaredConstructor.newInstance();
        System.out.println(instance);
        System.out.println(instance2);
    }
}
```

> 静态内部类

单例不安全，因为反射可以破坏单例

```java
package com.haust.single;
// 静态内部类
public class Holder {
    private Holder(){
    }
    public static Holder getInstace(){
        return InnerClass.HOLDER;
    }
    public static class InnerClass{
        private static final Holder HOLDER = new Holder();
    }
}
```

解决方式：

```java
private LazyMan(){	// 构造函数
    synchronized (LazyMan.class){
        if (csp == false){
            csp = true;
        }else {
            throw new RuntimeException("不要试图使用反射破坏异常");
        }
    }
}
```

> 枚举

```java
package com.haust.single;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
// enum 是一个什么？ 本身也是一个Class类
public enum EnumSingle {
    INSTANCE;
    public EnumSingle getInstance(){
        return INSTANCE;
    }
}
class Test{
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        EnumSingle instance1 = EnumSingle.INSTANCE;
        Constructor<EnumSingle> declaredConstructor = EnumSingle.class.getDeclaredConstructor(String.class,int.class);
        declaredConstructor.setAccessible(true);
        EnumSingle instance2 = declaredConstructor.newInstance();
        // NoSuchMethodException: com.kuang.single.EnumSingle.<init>()
        System.out.println(instance1);
        System.out.println(instance2);
    }
}
```



## 16、深入理解CAS

```java
package com.kuang.cas;
import java.util.concurrent.atomic.AtomicInteger;
public class CASDemo {
    // CAS compareAndSet : 比较并交换！ 
    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger(2020); 
        // 期望、更新 
        // public final boolean compareAndSet
        //                                    (int expect, int update) 
        // 如果我期望的值达到了，那么就更新，否则，
        // 就不更新, CAS 是CPU的并发原语！ 
        System.out.println(atomicInteger.compareAndSet(2020, 2021)); 
        System.out.println(atomicInteger.get()); 
        atomicInteger.getAndIncrement() // 看底层如何实现 ++ 
        System.out.println(atomicInteger.compareAndSet(2020, 2021)); 
        System.out.println(atomicInteger.get()); 
    } 
}
```

执行结果如图：

![img](pics\20210130191605818.png)

> Unsafe 类

![img](pics\j27.png)

![img](pics\j28.png)

![img](pics\j29.png)

CAS ： 比较当前工作内存中的值和主内存中的值，如果这个值是期望的，那么则执行操作！如果不是就一直循环！

> CAS ： ABA 问题

![在这里插入图片描述](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\j30.png)

```java
package com.haust.cas; 
import java.util.concurrent.atomic.AtomicInteger; 
public class CASDemo {
    // CAS compareAndSet : 比较并交换！ 
    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger(2020); 
        /*
         * 类似于我们平时写的SQL：乐观锁
         *
         * 如果某个线程在执行操作某个对象的时候，其他线程若操作了该对象，
         * 即使对象内容未发生变化，也需要告诉我。
         *
         * 期望、更新：
         * public final boolean compareAndSet(int 
         *                                    expect, int update) 
         * 如果我期望的值达到了，那么就更新，否则，就不更新, 
         *                                    CAS 是CPU的并发原语！ 
         */
        // ============== 捣乱的线程 ================== 
        System.out.println(atomicInteger.compareAndSet(2020, 2021)); 
        System.out.println(atomicInteger.get()); 
        System.out.println(atomicInteger.compareAndSet(2021, 2020)); 
        System.out.println(atomicInteger.get()); 
        // ============== 期望的线程 ================== 
        System.out.println(atomicInteger.compareAndSet(2020, 6666)); 
        System.out.println(atomicInteger.get()); 
    } 
}
```

![img](pics\j31.png)

## 17、原子引用

> 解决ABA 问题，引入原子引用！ 对应的思想：乐观锁！

带版本号 的原子操作！

```java
package com.haust.cas;
import java.util.concurrent.TimeUnit; 
import java.util.concurrent.atomic.AtomicStampedReference; 
    public class CASDemo {
        /*
         * AtomicStampedReference 注意，
         * 如果泛型是一个包装类，注意对象的引用问题 
         * 正常在业务操作，这里面比较的都是一个个对象 
         */
        // 可以有一个初始对应的版本号 1
        static AtomicStampedReference<Integer> 
                        atomicStampedReference = 
                            new AtomicStampedReference<>(2020,1);
        // CAS compareAndSet : 比较并交换！ 
        public static void main(String[] args) {
            new Thread(()->{
                // 获得版本号
                int stamp = atomicStampedReference.getStamp(); 
                System.out.println("a1=>"+stamp); 
                try {
                    TimeUnit.SECONDS.sleep(2); 
                } catch (InterruptedException e) {
                    e.printStackTrace(); 
                }
                atomicStampedReference.compareAndSet(
                    2020, 
                    2022, 
                    atomicStampedReference.getStamp(), // 最新版本号
                    // 更新版本号
                    atomicStampedReference.getStamp() + 1); 
                      System.out.println("a2=>"
                                 +atomicStampedReference.getStamp()); 
                     System.out.println(
                        atomicStampedReference.compareAndSet(
                            2022, 
                            2020, 
                            atomicStampedReference.getStamp(), 
                            atomicStampedReference.getStamp() + 1)); 
                    System.out.println("a3=>"
                                 +atomicStampedReference.getStamp()); 
                },"a").start(); 
            // 乐观锁的原理相同！ 
            new Thread(()->{
                // 获得版本号 
                int stamp = atomicStampedReference.getStamp(); 
                System.out.println("b1=>"+stamp); 
                try {
                    TimeUnit.SECONDS.sleep(2); 
                } catch (InterruptedException e) {
                    e.printStackTrace(); 
                }
                System.out.println(
                    atomicStampedReference.compareAndSet(
                                    2020, 6666, stamp, stamp + 1)); 
                System.out.println("b2=>"
                +atomicStampedReference.getStamp()); 
            },"b").start();
        } 
}
```

结果如图：

![img](pics\j32.png)

**注意：**

**Integer 使用了对象缓存机制，默认范围是 -128 ~ 127 ，推荐使用静态工厂方法 valueOf 获取对象实例，而不是 new，因为 valueOf 使用缓存，而 new 一定会创建新的对象分配新的内存空间；**

下面是阿里巴巴开发手册的规范点：

![img](pics\j33.png)

## 18、各种锁的理解

### 1、公平锁、非公平锁

公平锁： 非常公平， 不能够插队，必须先来后到！

非公平锁：非常不公平，可以插队 （默认都是非公平）

```java
public ReentrantLock() {
    sync = new NonfairSync(); 
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync(); 
}
```

不再详细论述。

### 2、可重入锁

可重入锁（递归锁）

![img](pics\j34.png)

> Synchronized 版

```java
package com.haust.lock;
// Synchronized
public class Demo01 {
    public static void main(String[] args) {
        Phone phone = new Phone();
        new Thread(()->{
            phone.sms();
        },"A").start();
        new Thread(()->{
            phone.sms();
        },"B").start();
    }
}
class Phone{
    public synchronized void sms(){
        System.out.println(Thread.currentThread().getName() 
                                                           + "sms");
        call(); // 这里也有锁(sms锁 里面的call锁)
    }
    public synchronized void call(){
        System.out.println(Thread.currentThread().getName() 
                                                           + "call");
    }
}
```

> Lock 版

```java
package com.haust.lock;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
public class Demo02 {
    public static void main(String[] args) {
        Phone2 phone = new Phone2();
        new Thread(()->{
            phone.sms();
        },"A").start();
        new Thread(()->{
            phone.sms();
        },"B").start();
    }
}
class Phone2{
    Lock lock = new ReentrantLock();
    public void sms(){
        lock.lock(); 
        // 细节问题：lock.lock(); lock.unlock(); 
        // lock 锁必须配对，否则就会死在里面
        // 两个lock() 就需要两次解锁
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() 
                                                           + "sms");
            call(); // 这里也有锁
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
            lock.unlock();
        }
    }
    public void call(){
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() 
                                                           + "call");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```



> 3、自旋锁

![在这里插入图片描述](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\j35.png)

我们来自定义一个锁测试：

```java
package com.haust.lock;
import java.util.concurrent.atomic.AtomicReference;
/**
 * 自旋锁
 */
public class SpinlockDemo {
    // int   0
    // Thread  null
    // 原子引用
    AtomicReference<Thread> atomicReference = 
                                            new AtomicReference<>();
    // 加锁
    public void myLock(){
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName() 
                                                       + "==> mylock");
        // 自旋锁
        while (!atomicReference.compareAndSet(null,thread)){
        }
    }
    // 解锁
    // 加锁
    public void myUnLock(){
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName()
                                                   + "==> myUnlock");
        atomicReference.compareAndSet(thread,null);// 解锁
    }
}
```

测试:

```java
package com.haust.lock;
import java.util.concurrent.TimeUnit;
public class TestSpinLock {
    public static void main(String[] args) throws 
                                            InterruptedException {
//        ReentrantLock reentrantLock = new ReentrantLock();
//        reentrantLock.lock();
//        reentrantLock.unlock();
        // 底层使用的自旋锁CAS
        SpinlockDemo lock = new SpinlockDemo();// 定义锁
        new Thread(()-> {
            lock.myLock();// 加锁
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                lock.myUnLock();// 解锁
            }
        },"T1").start();
        TimeUnit.SECONDS.sleep(1);
        new Thread(()-> {
            lock.myLock();
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                lock.myUnLock();
            }
        },"T2").start();
    }
}
```

结果如图：

![img](pics\20210130191812975.png)























