# synchronized

## 1. synchronized 的使用方式有哪些或者能修饰哪些地方？


 1.1 修饰静态方法，作用与整个类，静态资源在整个内存中只有一份，通过new出来的对象调用该方法或者通过类名调用该方法，都是给当前类加锁，换句话说，这个类的这个静态方法在整个系统只有一个锁。

```
public static synchronized void myMethod() {
    //当前类加锁
}
```

 1.2 修饰实例方法，作用与实例对象，即每个实例对象加锁

```
public synchronized void getResource(){
   //实列对象加锁
}
```
 1.3 修饰代码块，如果是this或者是一个对象等，仍是是给当前对象加锁，如果是类名.class 这种是给整个类加锁

```
public void getSome(){
    synchronized(this){
        // 当前对象加锁
        System.out.println("this");
    }
    synchronized (SynTest.class){
        // 当前类加锁
        System.out.println("class");
    }
}
```

下面是针对实例方法测试内容，针对同一个对象synTest的两次线程调用，结果是先启动的线程获取锁先执行完成后，另外的线程才能得到锁，测试结果中线程2先启动获得锁执行完成后线程1才执行，而对象synTest2却并不受影响。所以，synchronized是针对每个对象加锁

```
public class SynTest {

    //普通方法
    public void normalMethod(int num){
        System.out.println("enter norma method:"+this+",num="+num);
        try {
            Thread.sleep(1000);
        }catch (Exception e){
        }
        System.out.println("out  norma method:"+this+",num="+num);
    }
    // synchronized 修饰的方法
    public synchronized void getResource(int num){
        System.out.println("enter getResource:"+this+",num="+num);
        try {
            Thread.sleep(1000);
        }catch (Exception e){
        }
        System.out.println("out getResource:"+this+",num="+num);
    }
    public static void main(String[] args){
        SynTest synTest = new SynTest();
        SynTest synTest2 = new SynTest();
        Runnable temp1 = new Runnable() {
            @Override
            public void run() {
                System.out.println("thread 1 started:");
                synTest.normalMethod(1);
                synTest.getResource(1);
            }
        };
        Runnable temp2 = new Runnable() {
            @Override
            public void run() {
                System.out.println("thread 2 started:");
                synTest.normalMethod(2);
                synTest.getResource(2);
            }
        };

        Runnable temp3 = new Runnable() {
            @Override
            public void run() {
                System.out.println("thread 3 started:");
                synTest2.normalMethod(3);
                synTest2.getResource(3);
            }
        };
        Thread t1 = new Thread(temp1);
        Thread t2 = new Thread(temp2);
        Thread t3 = new Thread(temp3);
        t1.start();
        t2.start();
        t3.start();
    }
}

输出结果（根据线程启动情况并不唯一）：
thread 2 started://线程2
thread 1 started://线程1
thread 3 started://线程3
enter norma method:com.jov.SynTest@7cfbac7b,num=2 //线程2普通方法被执行
enter norma method:com.jov.SynTest@7cfbac7b,num=1 //线程1普通方法被执行
enter norma method:com.jov.SynTest@31ba7625,num=3 //线程2普通方法被执行
out  norma method:com.jov.SynTest@7cfbac7b,num=2  
out  norma method:com.jov.SynTest@31ba7625,num=3
out  norma method:com.jov.SynTest@7cfbac7b,num=1
enter getResource:com.jov.SynTest@31ba7625,num=3  //线程3 synchronized 方法被执行
enter getResource:com.jov.SynTest@7cfbac7b,num=2  //线程2 synchronized 方法被执行
out getResource:com.jov.SynTest@31ba7625,num=3
out getResource:com.jov.SynTest@7cfbac7b,num=2
enter getResource:com.jov.SynTest@7cfbac7b,num=1  //线程1 synchronized 方法被执行
out getResource:com.jov.SynTest@7cfbac7b,num=1
```


## 2. synchronized 实现原理或者底层原理等？

![同步代码块 图1](https://images.gitee.com/uploads/images/2020/1106/144205_40c17e72_8076629.png "同步代码块 图1")
 
![同步静态方法 图2](https://images.gitee.com/uploads/images/2020/1106/150332_5630893e_8076629.png "同步静态方法 图2")

![同步实例方法 图3](https://images.gitee.com/uploads/images/2020/1106/150440_27ae4889_8076629.png "同步实例方法 图3")


如果是代码块，不管是this||object 还是类名.class 都是通过monitorenter和monitorexit两个指令来完成加锁的过程，monitorenter表示进入同步块，monitorexit表示退出同步块。而同步方法都是使用 ACC_SYNCHRONIZED标志。

那么monitorenter、monitorexit如何实现同步过程的呢？这里首先要对java对象内存结构做一下说明：

java中对象的结构主要包括：对象头，实例数据以及对齐填充信息

引用内容及图片摘自[《啃碎并发（七）：深入分析Synchronized原理》](https://juejin.im/post/6844903640197513230)
![java对象结构](https://images.gitee.com/uploads/images/2020/1106/151517_4ae76bdf_8076629.png "java对象结构.png")

> 实例数据：存放类的属性数据信息，包括父类的属性信息；
>
> 对齐填充：由于虚拟机要求 对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐；
>
> 对象头：Java对象头一般占有2个机器码（在32位虚拟机中，1个机器码等于4字节，也就是32bit，在64位虚拟机中，1个机器码是8个字节，也就是64bit），但是 如果对象是数组类型，则需要3个机器码，因为JVM虚拟机可以通过Java对象的元数据信息确定Java对象的大小，但是无法从数组的元数据来确认数组的大小，所以用一块来记录数组长度。

以上是java对象的整体结构，其中对象头又包含两部分结构：Mark Word（标记字段） 和 Klass Pointer（类型指针）

引用内容及图片摘自[《看完你就明白的锁系列之锁的状态》](https://www.cnblogs.com/cxuanBlog/p/11684390.html)

> Mark Word：主要包括哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等。这些信息都是与对象自身定义无关的数据，所以Mark Word被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据。它会根据对象的状态复用自己的存储空间，也就是说在运行期间Mark Word里存储的数据会随着锁标志位的变化而变化。
>
> Klass Point：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例
>
> 32位虚拟机的 Mark Word 和 Klass Pointer 分别占用 32bits 的字节，而 64位虚拟机的 Mark Word 和 Klass Pointer 占用了64bits 的字节

以32位虚拟机为例：
![Mark Word 和 Klass Pointer](https://images.gitee.com/uploads/images/2020/1106/153458_8c4b4c8b_8076629.png "Mark Word 和 Klass Pointer.png")

> 无状态也就是无锁的时候，对象头开辟 25bit 的空间用来存储对象的 hashcode ，4bit 用于存放分代年龄，1bit 用来存放是否偏向锁的标识位，2bit 用来存放锁标识位为01
>
> 偏向锁 中划分更细，还是开辟25bit 的空间，其中23bit 用来存放线程ID，2bit 用来存放 epoch，4bit 存放分代年龄，1bit 存放是否偏向锁标识， 0表示无锁，1表示偏向锁，锁的标识位还是01
>
> 轻量级锁中直接开辟 30bit 的空间存放指向栈中锁记录的指针，2bit 存放锁的标志位，其标志位为00

> 重量级锁中和轻量级锁一样，30bit 的空间用来存放指向重量级锁的指针，2bit 存放锁的标识位，为11
>
> GC标记开辟30bit 的内存空间却没有占用，2bit 空间存放锁标志位为11。

上面的内容对MarkWord做了详细说明，其存储的主要内容：哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等，以及MarkWord中锁的几种状态，无锁->偏向锁>轻量级锁->重量级锁。

#### 2.1 那么 synchronized 锁的升级过程是什么样子的呢？

#### 无锁
无锁状态，无锁即没有对资源进行锁定，所有的线程都可以对同一个资源进行访问，但是只有一个线程能够成功修改资源。

无锁的特点就是在循环内进行修改操作，线程会不断的尝试修改共享资源，直到能够成功修改资源并退出，在此过程中没有出现冲突的发生，这很像我 CAS 实现，CAS 的原理和应用就是无锁的实现。无锁无法全面代替有锁，但无锁在某些场合下的性能是非常高的。

#### 偏向锁
Hotspot 的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，还存在锁由同一线程多次获得的情况，偏向锁就是在这种情况下出现的，它的出现是为了解决只有在一个线程执行同步时提高性能。

可以从对象头的分配中看到，偏向锁要比无锁多了线程ID 和 epoch，当一个线程访问同步代码块并获取锁时，会在对象头和栈帧的记录中存储线程的ID，等到下一次线程在进入和退出同步代码块时就不需要进行 CAS 操作进行加锁和解锁，只需要简单判断一下对象头的 Mark Word 中是否存储着指向当前线程的线程ID，判断的标志当然是根据锁的标志位来判断的。

#### 偏向锁的获取过程
1. 访问 Mark Word 中偏向锁的标志是否设置成 1，锁的标志位是否是 01 --- 确认为可偏向状态。

2. 如果确认为可偏向状态，判断当前线程id 和 对象头中存储的线程 ID 是否一致，如果一致的话，则执行步骤5，如果不一致，进入步骤3

3. 如果当前线程ID 与对象头中存储的线程ID 不一致的话，则通过 CAS 操作来竞争获取锁。如果竞争成功，则将 Mark Word 中的线程ID 修改为当前线程ID，然后执行步骤5，如果不一致，则执行步骤4

4. 如果 CAS 获取偏向锁失败，则表示有竞争（CAS 获取偏向锁失败则表明至少有其他线程曾经获取过偏向锁，因为线程不会主动释放偏向锁）。当到达全局安全点（SafePoint）时，会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否存活（因为可能持有偏向锁的线程已经执行完毕，但是该线程并不会主动去释放偏向锁），如果线程不处于活动状态，则将对象头置为无锁状态(标志位为01)，然后重新偏向新的线程；如果线程仍然活着，撤销偏向锁后升级到轻量级锁的状态（标志位为00），此时轻量级锁由原持有偏向锁的线程持有，继续执行其同步代码，而正在竞争的线程会进入自旋等待获得该轻量级锁。

5. 执行同步代码

#### 偏向锁的释放过程
偏向锁的释放过程可以参考上述的步骤4 ，偏向锁在遇到其他线程竞争锁时，持有偏向锁的线程才会释放锁，线程不会主动释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为01）或轻量级锁（标志位为00）的状态。


#### 轻量级锁
轻量级锁是指当前锁是偏向锁的时候，被另外的线程所访问，那么偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，从而提高性能。

#### 轻量级锁 加锁过程
在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为 01 状态，是否为偏向锁为 0 ），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的 Mark Word 的拷贝，然后拷贝对象头中的 Mark Word 复制到锁记录中
![轻量级锁](https://images.gitee.com/uploads/images/2020/1106/160007_a8855803_8076629.png "轻量级锁.png")

拷贝成功后，虚拟机将使用 CAS 操作尝试将对象的 Mark Word 更新为指向 Lock Record 的指针，并将 Lock Record里的 owner 指针指向对象的 Mark Word。

如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为 00 ，表示此对象处于轻量级锁定状态。

![轻量级锁](https://images.gitee.com/uploads/images/2020/1106/160106_85b5f97c_8076629.png "轻量级锁.png")

如果这个更新操作失败了，虚拟机首先会检查对象的 Mark Word 是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为 10 ，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。

#### 重量级锁
重量级锁也就是通常说 synchronized 的对象锁，锁标识位为10，其中指针指向的是 monitor 对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如 monitor 可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态。
![重量级锁](https://images.gitee.com/uploads/images/2020/1106/160213_268df71d_8076629.png "重量级锁.png")

上图简单描述多线程获取锁的过程，当多个线程同时访问一段同步代码时，首先会进入 Entry Set当线程获取到对象的 monitor 后进入 The Owner 区域并把 monitor 中的 owner 变量设置为当前线程，同时 monitor 中的计数器count 加1，若线程调用 wait() 方法，将释放当前持有的 monitor，owner变量恢复为 null，count自减1，同时该线程进入 WaitSet 集合中等待被唤醒。若当前线程执行完毕也将释放 monitor (锁)并复位变量的值，以便其他线程进入获取monitor(锁)

以上 关于 synchronized 锁的升级过程 内容 摘自[《看完你就明白的锁系列之锁的状态》](https://www.cnblogs.com/cxuanBlog/p/11684390.html)

为什么摘抄了这么长一段？上面我们提到monitorenter、monitorexit两个指令如何实现锁的，翻阅大多数文章都说的不是很明白，大多都是扯到了monitor监视器，但是我们都知道synchronized是有升级过程的，不可能上来就使用monitor这种东西吧。只有这篇文章明确描述了monitor，Lock Record这些在哪一阶段使用，加上synchronized 锁的膨胀过程被面试的很频繁，所以干脆一块拿过来，我承认没有该作者写的好。

回到上面的问题：monitorenter、monitorexit如何实现锁的过程，其实就是每个线程竞争对象锁的过程，进入monitorenter之后，竞争从偏向锁开始，一步一步往上升级，文章介绍的很清楚，这里也不再多说。
当然这里其实也只介绍了个大概过程，有些面试官总是喜欢追根究底，对于这种情况，推荐下面的文章，这才是硬核文章，从c++代码一步一步分析，包括monitorenter，monitorexit两个指令的使用：

[死磕Synchronized底层实现--偏向锁](https://juejin.im/post/6844903928681742344) 


#### 2.2 那么ACC_SYNCHRONIZED 标志时如何实现锁的呢，它与monitorenter、monitorexit有关系吗？
 

#### jdk 1.6 对synchronized做了哪些优化
