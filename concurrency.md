## 进程和线程
[proc fs and shared pids](http://lkml.iu.edu/hypermail/linux/kernel/9608/0191.html)
Both threads and processes are really just one thing: a "context ofexecution". Trying to artificially distinguish different cases is justself-limiting.

## double-checked locking
不用获得锁便可以检测锁的条件, 用来减少第一次判断锁的条件的开销, 经常被用在单例模式中. 但是这是不安全的一种模式.

```
public class Singleton {
    private static Singleton instance = null;
    private Singleton(){}
   
    public static Singleton  getInstance() {
       if(instance == null) {
           instance = new Singleton();
       }
       return instance;
    }
}
```
这段代码在多线程环境中有明显的错误(线程 A, B 同时判断instance == null)

于是有了下面的代码:
```
 public synchronized static Singleton getInstance() {
       if(instance == null) {
           instance = new Singleton();
       }
       return instance;
    }
```
这段代码正确性没有问题, 但是获取`getInstance`只能是串行的. 实际上只有初始化的时候才需要串行, 之后都可以并行的访问.

改进性能得到:
```
1. public static Singleton getInstance() {
2.       if(instance == null) {
3.           synchronized(Singleton.class) {
4.              if(instance == null) {
5.                  instance = new Singleton();
6.              }
7.           }
8.       }
9.       return instance;
10.    }
```
这就是 double-checked locking. 在`instance == null`, 同步的创建实例, 一般情况下可以并发访问 instance.

此时正确性出现了问题. 原因在于`instance = new Singleton()`, 不是原子的, 虽然 synchronized 保证了互斥, 但是不保证代码块中的代码原子的执行. 考虑如下情况. 线程 A 执行到了第5行, 注意第 5 行不是原子的, new Instance 获得了内存地址并赋给了 instance 但没初始化. 此时线程 B 执行到了第 2 行(synchronized 外, 可以并发), 通过了判断, 获得了 instance, 但是 instance 指向的内存却是空的.

此时需要声明 instance 为 volatile, volatile 保证保证内存可见性，防止指令重排序，并不保证操作原子性。 保证可见性意味着使用该变量必须重新去主内存读取，修改了该变量必须立刻刷新主内存。防止重排序意味着插入内存屏障.
https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#dcl

## 公平锁与非公平锁
公平体现在获取锁的顺序上, 公平锁意味着以请求锁的顺序获得锁, 非公平锁意味着随机获得锁. 由于大多数情况下, 获得锁的顺序并不重要, 而非公平锁的性能往往高于公平锁, 所以通常使用非公平锁.

## 读写锁
一个ReadWriteLock维护一对关联的锁，一个用于只读操作一个用于写。在没有写线程的情况下一个读锁可能会同时被多个读线程持有。写锁是独占的

## 虚假唤醒
todo

