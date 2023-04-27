## 多线程

C++11开始支持原生的多线程 

在头文件thread库中被定义

一个小例子 :

```c++
void thread_task() {
    std::cout << "hello thread" << std::endl;
}

int main() {
    //创建一个线程 thread t（函数名）
    std::thread t(thread_task);

    //判断这个函数是否可以被join
    std::cout << t.joinable( )<< std::endl;
 
    //获取这个线程的ID
    std::cout << t.get_id()<< std::endl;
  
    //调用join函数 将这个线程加入程序
    t.join(); 
    sleep(5);
    // 会先输出hello thread
    // 5s后输出hello world
    std::cout << "Hello, World!" << std::endl;
    return 0;
}

//注意
//拷贝构造函数(被禁用)，意味着 thread 不可被拷贝构造。
```

关于join和detect

(1）当使用join()函数时，主调线程阻塞 **注意 是阻塞，即不再进行任何操作 直到子线程结束**，等待被调线程终止，然后主调线程回收被调线程资源，并继续运行；

(2）当使用detach()函数时，主调线程继续运行，被调线程驻留后台运行，主调线程无法再取得该被调线程的控制权。当主调线程结束时，由运行时库负责清理与被调线程相关的资源。

举个简单的例子 现在有一个线程sleep5s 

如果你使用join加入它，主线程也会在5s结束，但如果用detect，就会直接结束。

如果你想更好的操作线程

可以使用th.native_handle() 获取一个线程的handle

然后对其进行操作 例如

```
SuspendThread(th.native_handle()); //挂起th线程
ResumeThread(th.native_handle()); // 恢复th线程
```

# 锁

Mutex 又称互斥量，C++ 11中与 Mutex 相关的类（包括锁类型）和函数都声m明在 <mutex> 头文件中

mutex实际上是对pthread.h 中pthread_mutex的上层封装

#### Mutex 系列类(四种)

- std::mutex，最基本的 Mutex 类。
- std::recursive_mutex，递归 Mutex 类。
- std::time_mutex，定时 Mutex 类。
- std::recursive_timed_mutex，定时递归 Mutex 类。

#### Lock 类（两种）

- std::lock_guard，与 Mutex RAII 相关，方便线程对互斥量上锁。
- std::unique_lock，与 Mutex RAII 相关，方便线程对互斥量上锁，但提供了更好的上锁和解锁控制。

#### 其他类型

- std::once_flag

构建unique_ptr传参时使用

- std::adopt_lock_t
- std::defer_lock_t 
- std::try_to_lock_t

```
defer_lock_不要获取锁的所有权
try_to_lock_尝试在不阻塞的情况下获得mutex的所有权
adopt_lock_假设调用线程已经拥有了锁的所有权
```

#### 函数

- std::try_lock，尝试同时对多个互斥量上锁。
- std::lock，可以同时对多个互斥量上锁。
- std::call_once，如果多个线程需要同时调用某个函数，call_once 可以保证多个线程对该函数只调用一次。

---

---

---

#### 互斥锁

 **互斥锁的实现过程很简单，mutex是一个类，首先我们要先创建出类对象std::mutex mylock，然后在你需要锁的代码块前后加上mylock.lock()和mylock.unlock()，就可以实现互斥锁的加锁和解锁了。**

**当另一个线程想要去访问访问一些数据时，也需要去先申请这个互斥量的锁，如果无法获取这把锁，就无法继续进行下去，也就无法继续操作数据**

**可以具体实现可以看下面的代码：**

```c++
int counter = 0;
std::mutex mtx; // 保护counter

void increase(int time) {
    for (int i = 0; i < time; i++) {
    // 我们需要在counter为互斥量上锁  
    //  mtx.lock()；
        counter++;
    // 释放
    //  mtx.unlock();
    }
}

int main(int argc, char** argv) {
    std::thread t1(increase, 10000);
    std::thread t2(increase, 10000);
    t1.join();
    t2.join();
    std::cout << "counter:" << counter << std::endl;
    return 0;
}
```

std::mutex还有一个操作：mtx.try_lock()，字面意思就是：“尝试上锁”，与mtx.lock()的不同点在于：如果上锁不成功，**当前线程不阻塞**。

当一个线程中对Mutex进行两次上锁lock()操作时,会产生死锁，同样如果一个线程没有释放锁就直接退出了 同样会造成死锁。

这种情况我们可以用**lock_guard** 解决

```
// std::lock_guard对象构造时，自动调用mtx.lock()进行上锁
// std::lock_guard对象析构时，自动调mtx.unlock()释放锁
std::lock_guard<std::mutex> lk(mtx);
```

这样可以有效防止死锁情况 但是锁的自由度会变低。

为了保证安全同时不降低自由度 就有了**unique_lock**

```c++
// 构造方法类似
std::unique_guard<std::mutex> lk(mtx);
//这么写的话和普通的lock_guard没有区别
```

``` 
unique_lock( mutex_type& m, std::defer_lock_t t );  　//延迟加锁
unique_lock( mutex_type& m, std::try_to_lock_t t );　//尝试加锁
unique_lock( mutex_type& m, std::adopt_lock_t t );  　//马上加锁

lock()　　　　     //阻塞等待加锁
try_lock()　　      // 非阻塞等待加锁
try_lock_for()　　//在一段时间内尝试加锁
try_lock_until()　  //在某个时间点之前尝试加锁
```

可以看出 unique_lock综合了普通互斥锁和时间锁的特定，还限制了死锁的问题，可以说非常方便，但也有内存占用更高的缺点。

##### 时间锁

std::time_mutex 比 std::mutex 多了两个成员函数，try_lock_for()，try_lock_until()。

try_lock_for() 表示函数接受一个时间范围，也就是在这个时间段范围内没有获取锁，则该线程被阻塞。

try_lock_until 函数则接受一个时间点作为参数，在指定时间点未到来之前线程如果没有获得锁则被阻塞住，如果在此期间其他线程释放了锁，则该线程可以获得对互斥量的锁，如果超时（即在指定时间内还是没有获得锁），则返回 false。

##### 递归锁

互斥锁可被分为**递归锁**(recursive mutex)*和*非递归锁\*(**non-recursive mutex**)*。可递归锁也可称为***可重入锁\****(reentrant mutex)*，非递归锁又叫***不可重入锁****(non-reentrant mutex)*。

二者唯一的区别是，同一个线程可以多次获取同一个递归锁，不会产生死锁。而如果一个线程多次获取同一个非递归锁，则会产生死锁。

当然 上锁几次就要释放几次 不然还是还是会出错

---

#### 条件锁

两个线程都想操作数据，都需要对想要改动的变量（例如消息队列）加上一把锁，但是如果这两个线程 是有先后顺序的，也就是说一个线程需要等待另一个线程，比如读取消息队列的线程需要在写消息队列的线程之后执行，这时候，条件锁就有用了。

条件变量（Condition Variable）的一般用法是：线程 A 等待某个条件并挂起，直到线程 B 设置了这个条件，并通知条件变量，然后线程 A 被唤醒

在

```c++
std::mutex mtx;        // 全局互斥锁
std::condition_variable cr;   // 全局条件变量

//需要前置做完的进程
//////////////////////////////////////////
/// 新建一个锁 加锁
std::unique_lock<std::mutex> lck(mtx);

///... 对变量操作

/// cr.notify_all(); 唤醒所有的wait
//////////////////////////////////////////

//后续进程
// 加锁
std::unique_lock<std::mutex> lck(mtx);
// 阻塞，直到 my_flag == true 并且其他线程 notify 之后才返回
	cv.wait(lk, []{ return my_flag; }); // 等价于下面带 while循环的代码块
```

![image-20220703140906364](/Users/whz/Library/Application Support/typora-user-images/image-20220703140906364.png)

#### **读写锁**

C++17起。支持shared_mutex

共享锁支持11中mutex的上锁操作，以及添加“读锁”的上锁方式

![image-20220703145020963](/Users/whz/Library/Application Support/typora-user-images/image-20220703145020963.png)

![image-20220703145705250](/Users/whz/Library/Application Support/typora-user-images/image-20220703145705250.png)

换言之 如果你在上锁时用了shared_mutex，那么别的线程如果也用shared_mutex上锁，那么两个线程都可以访问数据。

两个线程 如果都用了unique_lock上锁，那么就会有一个先后顺序进行。

如果都用了shard_lock上锁，那么就都会进行，但是在shard_lock的作用域内 她们都不能对数据进行改写。

如果一个线程想了unique_lock 另一个上了share_lock 这是 share_lock的线程还是会被阻塞

#### 自旋锁

自旋锁的定义：当一个线程尝试去获取某一把锁的时候，如果这个锁此时已经被别人获取(占用)，那么此线程就无法获取到这把锁，该线程将会等待，间隔一段时间后会再次尝试获取。这种采用循环加锁 -> 等待的机制被称为`自旋锁(spinlock)`。

#### 原子量

原子量从操作系统底层就被定义了是不可被脏读的量。所以原子量一定不会出现问题。

##### std::atomic_flag 

```
std::atomic<bool> ready(false); 
```

test_and_set() 函数检查 std::atomic_flag 标志，如果 std::atomic_flag 之前没有被设置过，则设置 std::atomic_flag 的标志，并返回先前该 std::atomic_flag 对象是否被设置过，如果之前 std::atomic_flag 对象已被设置，则返回 true，否则返回 false

clear()

清除 std::atomic_flag 对象的标志位，即设置 atomic_flag 的值为 false

// 可以用来实现一个自旋锁



----

#### async future promise

1. async是c++中用于实现异步线程的一种方式，他的出现是为了解决异步线程的返回值问题。正常来说，为了实现一些异步操作，我们会将需要等待的过程交给另一个子线程去处理，也就是用thread库，去开一个新的线程，但是正常来说，thread调用的函数是没有返回值的，我们必须要使用传参的形式，去获取一个返回值。基于以上场景，我们可能会需要锁(互斥量)，去保证子线程的数据安全，以及条件变量，在子线程执行完去通知我们的父线程。

   不难想象，以上过程有些许复杂，所以我们就可以用到async关键字。

   async的使用方法和thread一样，但是他有一个future类型的返回值，对这个返回值使用get函数，可以得到future封装的变量，就是我们想要的值。

2. future可以理解为 未来会有的值，也就是子线程执行完给我们返回的值，他有两个特点：1. 懒加载，也就是只有当我们调用get函数的时候，他才会去执行我们所需的线程 2. 异步，就是会去创建一个新的线程来处理我们需要执行的函数。当然 这两个特点都是可以通过参数自己修改的std::launch::async|std::launch::deferred

3. future的get函数只能被调用一次，多次调用会报错

4. promise：如果说之前的future是为了父线程能从子线程中拿到值，那么promise就是为了异步的从父线程中向子线程传递值。 可以理解为是一个保证，父线程对子线程作出保证，一定会在将来给你传进一个参数，但不是现在。

   也就是说，我们在使用async注册一个函数的时候，这个函数的参数，我们还没有准备好，那么，我们此时需要新建一个promise，然后对promise调用get_future函数，获取一个promise对应的future，将这个future，作为参数传入函数内，函数内通过对这个future调用get方法，获取其中的变量值，进行计算。

   在将来某一时候，主线程确定了想要穿入参数中的值，我们就用promise的set_value方法，设置这个值即可

5. 我们在上述提到 一个future只能get一次，这就意味着多个async不能调用同一个promise产生的future, 当线程比较多时，会产生麻烦。

   针对这种情况，我们可以使用shared_future来解决
