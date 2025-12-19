# 高并发解决思路与手段
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191218224022205.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzIyNzk3NDI5,size_16,color_FFFFFF,t_70)
在线下的时候，同一时间自己本地测试没有问题，而一旦上线，面临着并发的情况，就会出现了各种各样的问题了。在这种情况下，就该思考在并发情况下我们该如何编码，才能得到我们想要的正确的结果。
# 基本概念
**并发：** 同时拥有两个或者多个线程，如果程序在单核处理器上运行，多个线程将交替的换入或者换出内存，这些线程是同时"存在"的，每个线程都处于执行过程中的某个状态，如果运行在多核处理器上，此时，程序中的每个线程都将分配到一个处理器核上，因此，可以同时运行。
并发情况的关注点在多个线程操作相同的资源时，如何保证线程安全，合理使用和分配资源！
**高并发（High	Concurrency）：** 高并发是互联网分布式系统架构设计中必须考虑的因素之一，它通常指，通过设计保证系统能够同时并行处理很多请求。
高并发指的是在短时间内出现的大量的请求，如双十一的抢购，以及12306的抢票，服务能够处理很多请求，提高程序性能。如果处理不好高并发，可能导致用户的请求时间过长，影响用户体验，也可能导致宕机！
# CPU多级缓存
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191223225135748.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzIyNzk3NDI5,size_16,color_FFFFFF,t_70)
**为什么需要CPU cache:**  因为CPU的 频率太快了，快到主存跟不上，这样在处理器时钟周期内，CPU常常需要等待主存，浪费资源。所以cache的出现是为了缓解CPU和内存之间速度的不匹配问题（结构：cpu-> cache -> memory）
**CPU cache的意义：**

 - **时间局部性：** 	如果某个数据被访问，那么在不久的将来它很可能被再次访问
 - **空间局部性：** 如果某个数据被访问，那么与它相邻的数据很快也可能被访问

**CPU多级缓存  ---  缓存一致性（MESI）**

 - 为了保证多个CPU cache 之间缓存共享数据的一致
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019122323081958.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzIyNzk3NDI5,size_16,color_FFFFFF,t_70)

**CPU多级缓存   --- 乱序执行优化**
乱序优化指的是处理器为了提高运算速度而做出违背代码原有顺序的优化
## Java内存模型（Java Memory Model, JMM）
Java内存模型规范了Java虚拟机和计算机内存是如何协同工作的，它规定了一个线程是如何和何时可以看到其他线程修改过的共享变量的值,以及在必须时如何同步的访问共享变量。
![Java内存模型(图片来源于网络)](https://img-blog.csdnimg.cn/20191224215516990.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzIyNzk3NDI5,size_16,color_FFFFFF,t_70)
堆是运行时的数据区，它是由垃圾回收来负责的，堆的优势是可以动态的分配内存大小，生存期也不必告诉编译器，因为它是在运行时动态分配内存的，Java的垃圾收集器可以动态的收集这些不要的数据。缺点是在运行的时候动态分配内存，存取速度比较慢。
栈的存取速度比堆要快，仅次于寄存器，栈的数据是可以共享的，缺点是存放的数据大小都是确定的，缺乏一些灵活性，栈中主要存放一些基本类型的变量，比如小写的int,short,byte,long,float,double,boolean,char等。引用存放栈内，它引用的对象放在堆里。 
如果两个线程同时调用了一个对象，如图中两个L2同时调用了Object3的同一个方法，他们都会访问这个对象的成员变量，但是这两个线程都拥有这个对象的**私有拷贝**。
![计算机硬件架构的简单图示（图片来源于网络）](https://img-blog.csdnimg.cn/20191224220910870.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzIyNzk3NDI5,size_16,color_FFFFFF,t_70)
![java内存模型和硬件架构之间的关联](https://img-blog.csdnimg.cn/20191224221223834.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzIyNzk3NDI5,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191224230231668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzIyNzk3NDI5,size_16,color_FFFFFF,t_70)
## Java内存模型---同步的八种操作

 - **Lock(锁定)**： 作用于主内存的变量，把一个变量标识为一条线程独占状态
 - **unlock(解锁)**:  作用于主内存的变量，把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
 - **read(读取):** 作用于主内存的变量，把一个变量值从主内存传输到线程的工作内存中，以便随后的load动作使用。
 - **load(载入)：** 作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中。
 - **use（使用）:** 作用于工作内存的变量，把工作内存中的一个变量值传递给执行引擎
 - **assign（赋值）**: 作用于工作内存的变量，它把一个从执行引擎接收到的值赋值给工作内存的变量
 - **store（存储）**: 作用于工作内存的变量，把工作内存中的一个变量的值传递到主内存中，以便随后的Write操作
 - **write(写入)**: 作用于主内存的变量，它把Store操作从工作内存中一个变量的值传送到主内存中的变量中。
