## work

1、CopyOnWriteArraySet()读写分离，为线程安全类。底层是CopyOnWriteArrayList()

2、ConcurrentHashMap线程安全的HashMap（set，list，map由于底层设计思想一样，都是线程不安全的）

3、HashSet的底层是HashMase's'sp,其add()是map的put()，但是add()只添加了key，其Value是Object的常量



4、锁

​	1、公平锁：按照线程的申请顺序进行获取，按照先进先出的规则

​	2、非公平锁：不按照先进先出原则，线程上来先尝试占用锁，如果失败，则按照公平锁机制(吞吐量比公平锁大，sync、lock为非公平锁)



### JUC

####1、Semaphore，线程控制，保证内部有固定数目的线程数

```java

		Semaphore semaphore = new Semaphore(3);
		
		for(int i=0;i<6;i++) {
			new Thread(()->{
				try {
					semaphore.acquire();
					System.out.println(Thread.currentThread().getName()+"\t 占据位置-------");
					TimeUnit.SECONDS.sleep(3);
					System.out.println(Thread.currentThread().getName()+"\t 离开位置");
				} catch (InterruptedException e) {
					e.printStackTrace();
				}finally {
					semaphore.release();
				}
			},String.valueOf(i)).start();
		}
	
```

####2、CyclicBarrier线程增加，增加到某数量运行线程

```java
	CyclicBarrier barrier = new CyclicBarrier(7,()->{
		System.out.println("最后的实现le ");
	}) ;
	
	for(int i =0 ;i<7;i++) {
		final int tem =i;
		new Thread(()->{
			System.out.println(Thread.currentThread().getName()+"\t"+tem);
			try {
				barrier.await();
			} catch (InterruptedException e) {
				e.printStackTrace();
			} catch (BrokenBarrierException e) {
				e.printStackTrace();
			}
		},String.valueOf(i)).start();
	}
```

####	3、CountDownLatch线程减少，用法与上同

####4、阻塞队列

```java
public class BlockingQueueDemo {
	
	public static void main(String[] args) throws Exception {
		//List list = null;
		BlockingQueue<String> blockingQueue = new ArrayBlockingQueue<String>(1);
		blockingQueue.add("a");// true
		//System.out.println(blockingQueue.add("d"));//异常
		
		System.out.println(blockingQueue.element());// 返回首元素
		
		System.out.println(blockingQueue.remove());// true
		//System.out.println(blockingQueue.remove());//  异常
		
		System.out.println(blockingQueue.offer("a"));// true
		System.out.println(blockingQueue.offer("a"));// 越界false
		System.out.println(blockingQueue.poll());// a
		System.out.println(blockingQueue.poll());// 越界取null
		
		// 阻塞
		new Thread(()->{
			try {
				blockingQueue.take();// 如果没有取到元素则一直等待
				System.out.println("取出完成");
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}).start();
		
		blockingQueue.put("a");
		System.out.println("**************");
		blockingQueue.put("d");// 如果该元素满了，则进行阻塞等待
		System.out.println("结束插入");
		
		System.out.println(blockingQueue.offer("q", 2, TimeUnit.SECONDS));// 插入，如果队列满则等待2秒 超时返回false
		System.out.println("等待结束");
		
	}
	

```

####5、阻塞队列-->同步队列

```java
//- 阻塞队列SynchronsQueue
//- 同步队列不存储，存入一个消费一个
public class SynchronsQueueDemo {
public static void main(String[] args) {
	BlockingQueue<String> blockingQueue = new SynchronousQueue<String>();
	new Thread(()->{
		try {
			System.out.println(Thread.currentThread().getName()+"\t +a");
			blockingQueue.put("a");
			System.out.println(Thread.currentThread().getName()+"\t +b");
			blockingQueue.put("b");
			System.out.println(Thread.currentThread().getName()+"\t +c");
			blockingQueue.put("c");
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	},"AA").start();
	
	new Thread(()->{
		try {
			TimeUnit.SECONDS.sleep(3);
			System.out.println(Thread.currentThread().getName()+"\t take");
			blockingQueue.take();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	},"BB").start();
	
}
}
```

####6、sync与lock的区别

​	sync是关键字，在jvm层起作用，不需要用户手动释放锁，等待是不可中断的

​	lock是类，属于api层次，一定需要手动释放，否则可能造成死锁，可以中断

```java
package juc;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author wyg_edu
 * @date 2020年5月7日 上午8:19:10
 * @version v1.0
 * 多线程按顺序调用,按照AA，BB,CC线程分别打印循环5次
 */
public class SyncAndReenTrantLockDemo {
	
	public static void main(String[] args) {
		ShareResource resource = new ShareResource();
		new Thread(()->{
			for (int i = 0; i < 5; i++) {
				resource.work("c1");
			}
		},"AA").start();
		new Thread(()->{
			for (int i = 0; i < 5; i++) {
				resource.work("c2");
			}
		},"BB").start();
		new Thread(()->{
			for (int i = 0; i < 5; i++) {
				resource.work("c3");
			}
		},"CC").start();
	}

}

class ShareResource {

	private int number = 1;

	private Lock lock = new ReentrantLock();

	private Condition c1 = lock.newCondition();
	private Condition c2 = lock.newCondition();
	private Condition c3 = lock.newCondition();

	public void work(String str) {

		switch (str) {
		case "c1":
			print(c1, c2, 1, 2);
			break;
		case "c2":
			print(c2, c3, 2, 3);
			break;
		case "c3":
			print(c3, c1, 3, 1);
			break;
		default:
			break;
		}
	}

	public void print(Condition wait, Condition done, int flag, int aim) {
		lock.lock();
		try {
			while (number != flag) {// 判断 避免虚假唤醒
				wait.await();
			}
			for (int i = 0; i < 5; i++) {
				System.out.println(Thread.currentThread().getName() + "\t" + i);
			}
			number = aim;// 修改标志位
			done.signal();// 通知指定线程
		} catch (Exception e) {
		} finally {
			lock.unlock();
		}
	}

}
```
####7、阻塞队列、volatile、CAS、线程交互、原子整形 合并demo

```java
package juc;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * @author wyg_edu
 * @date 2020年5月8日 上午8:11:17
 * @version v1.0 volatile/CAS/atmicInteger.BlockQueue/线程交互/原子引用 串讲
 */
public class ProdConsumer_BlockQueueDemo {
	public static void main(String[] args) throws Exception {
		MyRource  myRource = new MyRource(new ArrayBlockingQueue<String>(10));
		new Thread(()->{
			System.out.println(Thread.currentThread().getName()+"\t 生产启动" );
			try {
				myRource.myProd();
			} catch (Exception e) {
				e.printStackTrace();
			}
		},"prod").start();
		new Thread(()->{
			System.out.println(Thread.currentThread().getName()+"\t消费启动" );
			try {
				myRource.myConsumer();
			} catch (Exception e) {
				e.printStackTrace();
			}
		},"cousm").start();
		TimeUnit.SECONDS.sleep(5);
		System.out.println("活动结束了");
		myRource.stop();
	}
}

class MyRource {
	private volatile boolean FLAG = true;// 默认开启
	private AtomicInteger atomicInteger = new AtomicInteger();

	BlockingQueue<String> blockingQueue = null;

	public MyRource(BlockingQueue<String> blockingQueue) {
		this.blockingQueue = blockingQueue;
		System.out.println(blockingQueue.getClass().getName());
	}

	public void myProd() throws Exception {
		String data = null;
		boolean retValue = true;
		while (FLAG) {
			data = atomicInteger.incrementAndGet() + "";// i++
			blockingQueue.offer(data, 2, TimeUnit.SECONDS);
			if (retValue) {
				System.out.println(Thread.currentThread().getName() + "\t插入队列" + data + "成功");
			} else {
				System.out.println(Thread.currentThread().getName() + "\t插入队列" + data + "失败");
			}
			TimeUnit.SECONDS.sleep(1);
		}
		System.out.println(Thread.currentThread().getName() + "\t停止生产");
	}

	public void myConsumer() throws Exception {
		String result = null;
		while (FLAG) {
			result = blockingQueue.poll(2, TimeUnit.SECONDS);
			if (null == result || "".equalsIgnoreCase(result)) {
				FLAG = false;
				System.out.println(Thread.currentThread().getName()+"\t 超过两秒钟没有取到，消费退出");
				return;
			}
			System.out.println(Thread.currentThread().getName() + "\t 消费队列" + result + "成功");
		}
		
		
	}
	public void stop() {
		FLAG = false;
	}
}

```

#### 8、原子类的ABA问题

​	ABA问题是指在原子操作通过cas进行比较时对目标值进行对比，但是如果在取到目标值与进行比较期间，其他线程进入并且修改多次，最终结果恢复到原值时，原线程操作可以进行。但是期间的操作不具备原子性；

​	解决方式是使用

​	1、new AtomicmarkableReference<Integer>(100,false);（布尔值）

​	2、new AtomicStampedReference<Integer>(1);版本号（推荐）



###线程

实现多线程的方法

####1、继承Thread类

####2、实现Runnable接口（没有返回值）

####3、实现Callable接口（有返回值）

```java
package juc;

import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

/**
 * @author wyg_edu
 * @date 2020年5月9日 上午8:09:30
 * @version v1.0
 * 多线程中获取新线程的方式
 */
public class CallableDemo {
	
	public static void main(String[] args) throws Exception {
		
		FutureTask<Integer> futureTask1 = new FutureTask<Integer>(new MyThread());
		FutureTask<Integer> futureTask2 = new FutureTask<Integer>(new MyThread());
		
		new Thread(futureTask1,"AA").start();
		new Thread(futureTask2,"BB").start();// 需要起两个， 否则第二个会直接复用第一个的结果
		
		int n1 = 100;
		int n2 = 0;
		while(!futureTask1.isDone()) {
			n2 = futureTask1.get();// 要求获得计算返回值，如果没有接收到返回值，则会导致阻塞知道计算完成  所以表达式一般放到最后执行
		}
		
		System.out.println("result:\t" + (n1 + n2));
	}

}

class MyThread implements Callable<Integer>{

	@Override
	public Integer call() throws Exception {
		System.out.println("******* come in Callable");
		return 1024;
	}

}

```

#### 4、使用线程池（Executor）

​	七个参数：

​	1、corePoolSize，线程数：当前线程池中的线程数（默认线程池中没有线程，当线程增加时大于corePoolSize数量的线程会被回收，但此数量不会被销毁）

​	2、maximumPoolSize，最大线程数：当前线程池中最大的线程数量（CPU密集型配置，核心数+1；IO密集型cpu核心数/(1-阻塞系数)，阻塞系数在0.8~0.9 ）

​	3、keepAliveTime：多余线程的存活时间(大于corePoolSize的线程，可以使用allowCoreThreadTimeOut(boolean)方法，使线程减少至0)

​	4、unit：keepAliveTime的单位

​	5、workQueue，阻塞队列：排队等待区域

​	6、threadFactory，线程工厂：表示线程池工作的线程工厂，用于创建线程，使用默认即可

​	7、hangdler，拒绝策略：当队列满并且工作线程满时的解决方式

注：一般禁止直接使用Executor，因为阻塞队列的默认值是Integer的最大值，会导致服务器资源被充满



#### 5、死锁的Demo

​	java查看死锁的方式

​	1、使用jps命令查询进程号，Jstack -l 8384	(8384进程Id)

​	2、使用jconsole.exe中线程的死锁检测

```java
package juc;

import java.util.concurrent.TimeUnit;

/**
 * @author wyg_edu
 * @date 2020年5月21日 上午8:23:19
 * @version v1.0
 * 死锁是两个或者两个以上的进行在执行过程中
 * 因抢夺资源而造成的相互等待的现象
 * 若无外力推动则都将无法推动下去
 */
public class DeadLockDemo {

	static String lockA = "A";
	static String lockB = "B";
	public static void main(String[] args) {
		new Thread(new HoldLockThread(lockA, lockB), "ThreadAA").start();
		new Thread(new HoldLockThread(lockB, lockA), "ThreadBB").start();

	}
}

class HoldLockThread implements Runnable{
	
	private String lockA;
	private String lockB;
	
	public HoldLockThread(String lockA, String lockB) {
		super();
		this.lockA = lockA;
		this.lockB = lockB;
	}
	@Override
	public void run() {
		synchronized (lockA) {
			System.out.println(Thread.currentThread().getName()+"\t持有"+lockA+"\t尝试获取"+lockB);
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			synchronized (lockB) {
				System.out.println(Thread.currentThread().getName()+"\t持有"+lockB+"\t尝试获取"+lockA);
			}
		}
	}
	
}

```





###JAVA内存

####1、常见的垃圾回收算法

​	1）、引用计数：从根节点算起，被引用加一，没有被引用减一，删除为0的引用；无法解决循环引用

​	2）、复制算法：将存活的对象进行复制到其他区域，清空原来区域；消耗内存

​	3）、标记清除：标记需要清除的对象，然后消除这些对象；会产生内存碎片

​	4）、标记整理：在标记清除的基础上对其压缩，使其成为连续的内存，不会产生内存碎片；时间浪费



#### 2、JMM

​	概念：JMM即Java内存模型，是为了屏蔽系统和硬件的差异，让一套代码在不同平台下能达到相同的访问结果。

​	java在线程操作资源时，并不会直接操作主物理内存，，这样对性能影响较大，因此每个线程都有自己的工作空间，工作空间的变量是主内存的一份拷贝。

#### 3、什么对象可以作为GCRoot(是一组活跃的引用)

​	1）虚拟机栈中被引用的对象

​	2）方法去中的静态属性引用的对象

​	3）方法区内常量引用的对象

​	4）本地方法栈引用的对象





### Other

#### JAVA的值传递

java所有变量都是值传递，其中基本类型是直接copy一个副本，String类型由于底层是final修饰的数组形式，其值是无法改变的，修改时相当于创建了一个新的对象；引用类型传递的是引用的复制份，指向于同一个对象，因此修改时会修改原有对象，实际上还是相当于传值，并不是真正的传递引用

无论是基本类型还是引用类型都是将堆中的值进行复制一份，然后传给下一个函数，由于引用类型的形参实际上指向的堆内存与实参的指向是相同的，所以修改后会在实参中有相应的体现；



#### 序列化

​	序什么是Serializable接口？

一个对象序列化的接口，一个类只有实现了Serializable接口，它的对象才能被序列化

什么是序列化？

将对象的状态信息转换为可以存储或传输的形式的过程，在序列化期间，对象将其当前状态写入到临时存储区或持久性存储区，之后，便可以通过从存储区中读取或反序列化对象的状态信息，来重新创建该对象

什么情况下需要序列化？

当我们需要把对象的状态信息通过网络进行传输，或者需要将对象的状态信息持久化，以便将来使用时都需要把对象进行序列化

### 幂等性

​	HTTP方法的幂等性是指一次和多次请求某一个资源应该具有同样的副作用。（我们可以把withdraw设计为幂等的。create_ticket的语义是获取一个服务器端生成的唯一的处理号ticket_id，它将用于标识后续的操作。idempotent_withdraw和withdraw的区别在于关联了一个ticket_id，一个ticket_id表示的操作至多只会被处理一次，每次调用都将返回第一次调用时的处理结果。这样，idempotent_withdraw就符合幂等性了，客户端就可以放心地多次调用。）

​	个人理解：客户端先发起获取ticket_id的请求，服务端返回ticket_id，如果客户端没有收到改id，会再次发起请求，直至收到这个id为止；收到后开始发起业务，发起的请求中携带该id，服务端处理请求，返回结果；如若客户端没有收到处理结果，则会再次进行请求，在请求中依旧会携带原id，服务端判断该id已经处理完成，这会返回第一次的处理结果。

### 其他问题

#### CSRF跨站请求伪造

​	用户访问恶意网站A，而已网站A返回给用户HTTP信息，信息中要求用户访问B，由于用户与B网站有建立信任关系(cookies)，则会像正常访问一样发起对B网站的请求。

​	防御手段：验证Http Referer字段；在请求中增加Token验证；在Http头中增加自定义属性并验证；1、尽量使用POST请求；2、增加验证码验证；3、Refere Check：防止图片盗链；4、Anti CSRF Token

​	Token，是指用户像B网站发起请求时，首先会发起一个空请求，服务器收到空请求时会返回Token信息，用户在发起正式请求时会携带Token信息，服务段验证Token（Token可以保存在cookies或者session中），如不符合则拒绝。(Token一般会由服务器返回页面，增加一个隐藏的标签，请求发起时候拼接到请求中，携带到服务器)

### synchronized 关键字

​	当在方法上加锁时，同一个实例调用会阻塞，不同实例调用则不会阻塞（对象锁）

​	同步代码块传参this，同一个实例调用会阻塞，不同实例调用则不会阻塞（对象锁）

​	同步代码块传参为变量对象：1、当变量是基本类型或者String时，全局锁；2、当变量对象是引用类型时同一实例会阻塞，不同实例不会阻塞（对象锁）；

​	同步代码快传参为class，全局锁

​	在静态方法上加锁时，全局锁

### 转发与重定向

​	转发：服务器行为：客户端发起请求后，服务端匹配servlet处理，该servlet处理后调用jsp，因此jsp与servlet共用同一个request，转发只能在同一个web应用中使用；（客户端一次请求，服务端通过多个程序调用处理）

​	重定向：浏览器行为：客户端发起请求后，服务端匹配servlet处理，处理后返回给客户端，要求客户端调用jsp，两者相互独立，此使request不再是同一个；由于不是同一个请求，url发生改变，首次传输的信息会丢失，可以重定向到任意的url（客户端发起请求，服务端处理后，返回客户端，客户端重新发起请求调用下一个程序）

​	重定向可以解决表单提交后，刷新页面导致的表单重复提交



### ConcurrentHashMap的线程安全问题

​	**jdk1.7** 是由一个Segment数组和多个HashEntry数组组成 ；使用了分段锁技术，其主干是个segMent数组，相当于每个segment都是一个小的HashTable，都存在自己的锁，只要不在同一个段上修改就不会引起并发问题；segment继承于ReenTrantLock，segment本身就是一个锁，在进行put操作时进行锁定，其他线程无法操作

​	扩容时ConcurrentHashMap只会对segment中的HashEntry数组扩容

​	**put**操作首先进行hash计算，定位Segment位置，在进行hash计算定位HashEntry位置，插入时加锁

​	**get**操作相对put操作取消加锁操作

​	实质上时降低了锁的细粒度，提高了并发能力

​	**jdk1.8**整个过程采用了Synchronized或CAS的方式保证了线程安全