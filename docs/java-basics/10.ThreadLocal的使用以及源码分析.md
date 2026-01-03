@[toc]
# 前言
> 当多线程访问共享可变数据时，涉及到线程间同步的问题，并不是所有时候，都要用到共享数据，所以就需要线程封闭出场了。本文中的ThreadLocal就起到了线程封闭的作用。它提供了线程内的局部变量，不同线程之间不会相互干扰，这种变量在线程的生命周期内起作用，减少了同一个线程内多个函数或组件之间一些公共变量传递的复杂度。
# 特点
通俗的说ThreadLocal具备三个特性：

 - 线程并发： 在多线程并发的场景下使用
 - 传递数据： 我们通过ThreadLocal在同一线程，不同组件中传递公共变量
 - 线程隔离： 每个线程的变量都是独立的，不会相互影响

ThreadLocal用于保存每个线程独享的对象，为每个线程创建一个副本，这样每个线程都可以修改自己所拥有的副本，而不会影响其他线程的副本，从而确保了线程安全。总而言之，ThreadLocal的核心作用就是将变量在线程中隔离。
#  ThreadLocal的基本使用
## 常用方法
我们先看一下它的类图中的方法：
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/9905084-4892ea41581c9042?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
方法申明|描述
--|--
ThreadLocal()|创建ThreadLocal对象
public void set(T value)|设置当前线程绑定的局部变量
public T get()|获取当前线程绑定的局部变量
public void remove()|移除当前线程绑定的局部变量

## 例子
### 线程之间的变量非独立
```java
/**
 * Description
 * 线程隔离例子
 * 在多线程并发场景下，每个线程中的变量都是相互独立的
 * 线程A：设置（变量1）      获取（变量1）
 * 线程B：设置（变量2）      获取（变量2）
 * Date 2020/6/3 22:03
 * Created by kwz
 */
public class ThreadLocalExample1 {

    @Getter
    @Setter
    private String content;

    public static void main(String[] args) {
        ThreadLocalExample1 example = new ThreadLocalExample1();
        for (int i = 0; i < 5; i++) {
            Thread thread = new Thread(
                    () -> {
                        //每个线程存一个变量，过一会儿取这个变量
                        example.setContent(Thread.currentThread().getName() + "的数据");
                        System.out.println("----------------------------------------");
                        System.out.println(Thread.currentThread().getName() + "---->" + example.getContent());
                    }
            );
            //0->4一共有5个线程
            thread.setName("线程" + i);
            thread.start();
        }
    }
}

```
我们看它的控制台输出结果：
```
----------------------------------------
----------------------------------------
----------------------------------------
线程2---->线程2的数据
线程0---->线程2的数据
----------------------------------------
线程4---->线程4的数据
----------------------------------------
线程1---->线程2的数据
线程3---->线程4的数据
```
我们可以看到一个线程取到了其他线程的变量
### 线程之间的变量相互独立
```java
/**
 * Description(利用ThreadLocal)
 * 线程隔离例子
 * 在多线程并发场景下，每个线程中的变量都是相互独立的
 * 线程A：设置（变量1）      获取（变量1）
 * 线程B：设置（变量2）      获取（变量2）
 *
 * ThreadLocal
 * 1.set(): 将变量绑定到
 * Date 2020/6/3 22:03
 * Created by kwz
 */
public class ThreadLocalExample2 {

    ThreadLocal<String> t1 = new ThreadLocal<>();

    private String content;

    private String getContent(){
        String s = t1.get();
        return s;
    }

    private void setContent(String content){
        //变量content绑定到当前线程
        t1.set(content);
    }

    public static void main(String[] args) {
        ThreadLocalExample2 example = new ThreadLocalExample2();
        for (int i = 0; i < 5; i++) {
            Thread thread = new Thread(
                    () -> {
                        //每个线程存一个变量，过一会儿取这个变量
                        example.setContent(Thread.currentThread().getName() + "的数据");
                        System.out.println("----------------------------------------");
                        System.out.println(Thread.currentThread().getName() + "---->" + example.getContent());
                    }
            );
            //0->4一共有5个线程
            thread.setName("线程" + i);
            thread.start();
        }
    }
}

```
我们看它的控制台输出结果：
```
----------------------------------------
----------------------------------------
----------------------------------------
线程4---->线程4的数据
----------------------------------------
线程1---->线程1的数据
线程2---->线程2的数据
----------------------------------------
线程3---->线程3的数据
线程0---->线程0的数据
```
通过两个例子可以看到，通过引入ThreadLocal可以做到不同线程之前访问变量的相互独立性。
## ThreadLocal和Synchronized关键字
### Synchronized的同步方式
线程之间共享变量的相互隔离，我们首先想到的其实是Synchronized，我们通过下面的Synchronized也能够实现
```java
public class ThreadLocalExample3 {

    @Getter
    @Setter
    private String content;

    public static void main(String[] args) {
        ThreadLocalExample3 example = new ThreadLocalExample3();
        for (int i = 0; i < 5; i++) {
            Thread thread = new Thread(
                    () -> {
                        //每个线程存一个变量，过一会儿取这个变量
                        synchronized (ThreadLocalExample3.class) {
                            example.setContent(Thread.currentThread().getName() + "的数据");
                            System.out.println("----------------------------------------");
                            System.out.println(Thread.currentThread().getName() + "---->" + example.getContent());
                        }
                    }
            );
            //0->4一共有5个线程
            thread.setName("线程" + i);
            thread.start();
        }
    }
}
```
上面的这段代码也能起到线程对于共享变量的隔离效果，但是需要线程挨个排队去实现业务，这样就失去了多线程并发执行的意义了。
### ThreadLocal和Synchronized的区别
虽然ThreadLocal模式和Synchronized关键字都用于多线程并发访问变量的问题，不过两者处理问题的角度和思路的不同的
|Synchronized|ThreadLocal|
---|----|--|--|
原理|同步机制采用了以时间换空间的方式，只提供了一份变量，让不同线程排队访问|ThreadLocal采用了以空间换时间的方式，为每个线程都提供了一份变量的副本，从而实现同时访问而互不干扰
侧重点|多个线程之间访问资源的同步|多个线程中让每个线程之间的数据相互隔离

ThreadLocal和Synchronized都能解决问题，但是使用ThreadLocal更为合适，因为这样可以让程序拥有更高的并发性
##  ThreadLocal的使用场景与优势
### 使用场景
- 如在银行证券相互转账时，我们手动开启事务，直接获取当前线程绑定的连接对象，如果连接对象是空的再去连接池中获取连接，将此连接对象跟当前线程进行绑定。
### 优势
- 传递数据： 保存每个线程绑定的数据，在需要的地方可以直接获取，避免参数直接传递带来的代码耦合性问题
- 线程隔离： 各线程之间的数据相互隔离却又具有并发性，避免同步方式带来的性能损失

# ThreadLocal的设计及源码分析
## 设计
JDK1.8中ThreadLocal的设计原则是：每个Thread维护一个ThreadLocalMap，这个Map的key是ThreadLocal实例本身，value才是真正要存储的值object，比如存储的user对象等，如下图所示。
![在这里插入图片描述](https://upload-images.jianshu.io/upload_images/9905084-9163eb137db0ba65?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
过程如下：
- 每个Thread线程内部都有一个Map(ThreadLocalMap)
- Map里面存储ThreadLocal对象(key)	和线程的变量副本（value）
- Thread内部的Map是由ThreadLocal维护的，由ThreadLocal负责向map获取和设置线程的变量值
- 对于不同的线程，每次获取副本值时，别的线程并不能获取到当前线程的副本值，形成副本的隔离，互不干扰
## 源码分析
### get 方法
```java
public T get() {
    //获取到当前线程
    Thread t = Thread.currentThread();
    //每个线程内都有一个ThreadLocalMap对象,获取到当前线程内的 ThreadLocalMap 对象，
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //获取 ThreadLocalMap 中的 Entry 对象并拿到 Value值
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //如果这个线程之前没创建过 ThreadLocalMap，就初始化一个ThreadLocalMap
    return setInitialValue();
}
```
### getMap方法
```java
//上面传入一个Thread.currentThread()，这个方法就是获取当前线程内的ThreadLocalMap对象
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
threadLocals
```java
//这个对象的名字叫threadLocals
ThreadLocal.ThreadLocalMap threadLocals = null;
```
### set方法
```java
//获取当前线程的引用，把值给set进去
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
    //这个线程之前没创建过 ThreadLocalMap就创建一个
        createMap(t, value);
}
```
#  ThreadLocal使用注意事项
   一个线程池，中间有个线程1号工作了一段时间，它的ThreadLocalMap里面装了许多东西，装完了之后如果没有被清理掉，然后直接还到线程池里面，
   任务不停的过来，然后都没清理就会产生很大问题。因此在读线程池源码的时候，发现任务执行完之后首先清理ThreadLocals，首先清理Map。

# 项目中使用遇到的问题

# 思考
 - ThreadLocal是不是用来解决共享资源的多线程访问问题的？
   不是，ThreadLocal虽然可以用于解决多线程情况下的线程安全问题，但其资源不是共享的，而是每个线程独享的。它解决并发资源的思路是在initialValue中new出自己线程独享的资源，而多个线程之间，它们所访问的对象本身是不共享的，自然就不存在任何并发问题。
 - ThreadLocal什么情况下会发生内存泄漏？如何避免的？
   ThreadLocal内存泄漏主要体现在key和value的内存泄漏，ThreadLocal的消亡是伴随着线程的，单纯的将ThreadLocal置为空它底层的ThreadLocalMap
   的key会存在一个引用而不能释放，因此会发生内存泄漏。
   解决方法：
   1.key采用弱引用（ThreadLocal内部解决）
   2.通过.remove的方式避免了value值的内存泄漏

# 小结
本文主要介紹ThreadLocal的一些基本用法，以及它的设计和源码分享，又介绍了它可能产生内存泄漏体现的两个方面，
因此每次在使用完之后都要做.remove操作。