# iOS多线程同步、锁和文件读写方案
#iOS知识点/多线程

![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/FD4176E0-C2BF-4331-9FB6-E2FEF9E4BF64.png)
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/1132864E-8E9E-4283-AF14-CB073384C2BE.png)

# 线程同步方案之GCD
## 1. DispathQueue（serial串行队列）
同一个串行队列依次执行
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/C15F2EFA-5B4B-4A63-B18E-AB415A7E46BF.png)

## 2. DispathQueueSemaphore
信号量本身是控制资源对线程的最大并发数量，但是通过信号量为1时可以保证只有一个线程访问资源，达到线程同步的目的。
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/29A167C8-EC38-478D-B359-7F4CFB91B31A.png)

允许同时5个线程线程访问资源
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/B0672371-3E1D-427A-AD2F-F792F43F33A3.png)

# 线程同步方案之各种锁
*互斥锁（Blocking locks）*
常见的表现形式是当前线程会进入休眠，直到锁被其他线程释放。
*自旋锁（Spinlocks）*
使用一个循环不断地检查锁是否被释放，如果等待情况很少话这种锁是非常高效的，但如果等待情况非常多的情况下会浪费 CPU 时间。

**其他：**
*读写锁（Reader/writer locks）*
允许多个读线程同时进入一段代码，但当写线程获取锁时，其他线程（包括读取器）只能等待。这是非常有用的，因为大多数数据结构读取时是线程安全的，但当其他线程边读边写时就不安全了。
*递归锁（Recursive locks）*
允许单个线程多次获取相同的锁，非递归锁被同一线程重复获取时可能会导致死锁、崩溃或其他错误行为。

*高级锁和低级锁的区别*
低级锁：等不到锁就休眠睡觉，不占资源
高级锁：等不到锁就一直等待。

*线程阻塞有2种方案：忙等的性能比较好，因为睡眠唤醒消耗资源。*
1. 直接睡眠（Runloop底层）（低级锁）
2. 忙等（相当于一个条件while循环）（高级锁）

*使用锁的注意事项：*
1. 加锁、解锁操作一定要对应同一把锁
2. 加完锁一定要注意解锁，OSSpinLock（自旋锁）的锁会使线程会一直忙等，os_unfair_lock类型的锁会使线程休眠，一直不放开就导致死锁。

## 1. OSSpinLock：自旋锁（不安全，不推荐）
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/B1822058-7DD6-4764-A26F-B787557AA1A2.png)
当一个线程加锁获得这把锁后，其他线程不能对其进行加锁操作，并且会处于忙等状态。

### OSSpinLock不安全原因
当低优先级线程获取了锁，高优先级线程访问时陷入忙等状态，由于是循环调用，所以占用了系统调度资源，导致低优先级线程迟迟不能处理资源并释放锁，导致陷入死锁。

### OSSpinLock的使用
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/4064DD71-694B-4BDB-9498-D391A3302DE3.png)

## 2. os_unfair_lock（类似互斥锁）
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/85770B7D-65B3-4B6E-A5AE-0D394EC49FED.png)
当一个线程加锁获得这把锁后，其他线程不能对其进行加锁操作，并且会处于休眠状态，并非OSSpinLock的忙等。

### os_unfair_lock的使用
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/F69F4F11-88FA-454B-AF40-A4D9F9FC19B4.png)

## 3. pthread_mutex 普通互斥锁
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/D4AA46DF-1FCF-4420-855F-15591B7998C5.png)
当一个线程加锁获得这把锁后，其他线程不能对其进行加锁操作，并且会处于休眠状态。

### pthread_mutex锁类型
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/D4862A2B-B93F-4EC1-A4CC-84F7435D556C.png)

### pthread_mutex互斥锁注意事项
涉及递归调用时，产生互斥死锁的情况
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/F86289A1-B685-4D62-98CB-201E7DCFA318.png)
*产生死锁的原因：线程二次加锁和解锁互相等待*
第二次加锁时，锁已经被当前线程加过了，所以第二次加锁时当前线程会等待锁被释，因为是互斥锁，所以它会进入休眠。没有进行第二次加锁、解锁。
::当前线程想结束休眠的前提是它第一次加的锁被释放，而想要释放第一次加的锁又必须等待第二次加锁、解锁完毕。所以这里出现了2次加锁和解锁操作互相等待的问题导致了死锁。::

*解决方案：使用如下递归锁，将pthread_mutex的类型改为递归锁*

## 4. pthread_mutex 递归锁
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/E1E903A8-D1CB-4888-A32A-ED1787B4DD00.png)
允许同一个线程对同一把锁进行重复加锁，重复加锁不会导致互相等待的问题。其他线程不能对它进行加锁，否则会休眠等待。

### pthread_mutex 递归锁使用场景
当加锁的时候需要递归调用时
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/F1163D4B-F8A6-4212-B7FB-7A0A42F0DDDC.png)

## 5. pthread_mutex 条件锁（多线程间依赖问题）
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/BBFCFCBA-1B6C-465F-A9CA-2B6A69454A51.png)

### 常用方法的理解
pthread_cond_wait(&condition, &moneyLock)
刚开始时：当前线程会进入休眠，等待被这个条件唤醒，并且会放开锁
被条件唤醒时：会先加锁，然后继续执行。
pthread_cond_signal(&condition)
激活一个等待该条件的线程
pthread_cond_boardcast(&condition)
激活所有等待该条件的线程

### 注意事项
发出signal或者boardcast激活等待条件的线程时，等着这个锁的线程也不会立马持有锁，这里只是个信号，需要当前线程放开这把锁后，等着的线程才能持有这把锁。

### 带条件的锁使用场景
多线程间的依赖问题，这里模拟多个线程同时进行存/取钱操作，mutex_t锁能保证存取操作不会同时发生，能保证对资源的线程安全。但是实际情况是取钱时应当有足够的余额，所以加锁的同时需要使用条件来控制。

![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/1158BC2C-9A29-458F-9B29-C3A0F057DEE1.png)
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/BE34CA3A-677D-40D0-98E1-ECEE2F82182C.png)

## 6. NSLock和NSRecursiveLock
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/303AC18A-F189-4E7A-9EBE-560146BECE5A.png)

NSLock是对pthread_mutex的default类型的面向对象封装
NSRecursiveLock是对pthread_mutex的Recursive类型的面向对象封装

### NSLock的使用
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/F8CB5346-7A24-4ECA-8AE1-5197666AE7E7.png)

## 7. NSCondition（条件锁）
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/5830713C-840D-465B-9D64-E2CE4B7A9001.png)
NSCondition是对pthread_mutex和pthread_cond_t的封装

### NSCondition的使用
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/E6902B84-904B-44EE-BBB0-DD4265D99AFC.png)

## 8. NSConditionLock（可设置具体条件的锁）
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/568BCD0A-FEEF-40E7-8B10-69FB963E75BB.png)
NSConditionLock是对NSCondition的进一步封装，可以允许设置具体的自定义条件，而非一个C的pthread_cons_t。

### 常用方法的理解
condition.lock(whenCondition: 0)
当条件值为0时会加锁，不为0则休眠，并等待锁条件为0时再加锁往下走

### NSConditionLock应用
保证线程按照条件添加和释放的顺序执行。
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/57A1E32B-5391-4B18-9B49-60FF5F3D828A.png)

# 文件读写安全方案（多读单写）
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/89F71FFA-BA37-4136-8EC0-91127D4E184E.png)

##  1. pthread_rwlock（文件读写锁）
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/DBE997D9-0B89-42B9-B7EA-F46C8D142B3A.png)
允许多个读线程同时进入一段代码，但当写线程获取锁时，其他线程（包括读取器）只能等待。这是非常有用的，因为大多数数据结构读取时是线程安全的，但当其他线程边读边写时就不安全了。

### pthread_rwlock的使用
3个同时读，1个同时写
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/5A6B0483-94A7-4E5E-915D-6B29EE980D58.png)

## 2. DispatchBarrier（zha栅栏）
[GCD全面详解](bear://x-callback-url/open-note?id=1FD00D8F-62F2-4C1E-B3A5-E25C4FEC68EE-43626-00058063B7A54AB1)
3个同时读，1个同时写
![](iOS%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E3%80%81%E9%94%81%E5%92%8C%E6%96%87%E4%BB%B6%E8%AF%BB%E5%86%99%E6%96%B9%E6%A1%88/D24468D3-EA46-4798-8BF8-FBF129CC2D9B.png)

