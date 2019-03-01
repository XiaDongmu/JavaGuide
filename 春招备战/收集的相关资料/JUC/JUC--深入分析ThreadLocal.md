---

title: JUC--深入分析ThreadLocal
categories: JUC
date: 2018-10-02 23:26:54

---
# ThreadLocal #
1、调用Thread.currentThread()获取当前线程

2、JDK提供了ThreadLocal，在一个线程中传递同一个对象
![](https://i.imgur.com/sWKzfj8.png)
3、ThreadLocal表示线程的“局部变量”，它确保每个线程的ThreadLocal变量都是各自独立的。下面是ThreadLocal的典型使用场景
![](https://i.imgur.com/WxJKfbG.png)
4、ThreadLocal适合在一个线程处理的流程中保持上下文（避免了同一参数在所有方法内的传递）。下面是两个无参函数的示例。
![](https://i.imgur.com/S5owVFJ.png)

![](https://i.imgur.com/RovwUME.png)

![](https://i.imgur.com/6E44lp6.png)

5、使用ThreadLocal时要使用try...finally结构，原因是最后一定要把ThreadLocal给remove掉，如果不remove，线程被放回线程池的话，ThreadLocal变量会一直存在，影响其他线程。

6、可以将ThreadLocal看成全局Map<Thread，Object>：

- 每个线程获得ThreadLocal变量时，都使用Thread自身作为key

		Object threadLocalValue=threadLocalMap.get(Thread.currentThread());


# ThreadLocal源码分析 #
## ThreadLocalMap ##
ThreadLocalMap其内部利用Entry来实现key-value的存储，如下：

	  static class Entry extends WeakReference<ThreadLocal<?>> {
	            /** The value associated with this ThreadLocal. */
	            Object value;
	
	            Entry(ThreadLocal<?> k, Object v) {
	                super(k);
	                value = v;
	            }
	        }
从上面代码中可以看出Entry的key就是ThreadLocal，而value就是值。同时，Entry也继承WeakReference，所以说Entry所对应key（ThreadLocal实例）的引用为一个弱引用。

ThreadLocalMap的源码稍微多了点，我们就看两个最核心的方法getEntry()、set(ThreadLocal> key, Object value)方法。
### set(ThreadLocal<?> key, Object value) ###
	   private void set(ThreadLocal<?> key, Object value) {
	
	        ThreadLocal.ThreadLocalMap.Entry[] tab = table;
	        int len = tab.length;
	
	        // 根据 ThreadLocal 的散列值，查找对应元素在数组中的位置
	        int i = key.threadLocalHashCode & (len-1);
	
	        // 采用“线性探测法”，寻找合适位置
	        for (ThreadLocal.ThreadLocalMap.Entry e = tab[i];
	            e != null;
	            e = tab[i = nextIndex(i, len)]) {
	
	            ThreadLocal<?> k = e.get();
	
	            // key 存在，直接覆盖
	            if (k == key) {
	                e.value = value;
	                return;
	            }
	
	            // key == null，但是存在值（因为此处的e != null），说明之前的ThreadLocal对象已经被回收了
	            if (k == null) {
	                // 用新元素替换陈旧的元素
	                replaceStaleEntry(key, value, i);
	                return;
	            }
	        }
	
	        // ThreadLocal对应的key实例不存在也没有陈旧元素，new 一个
	        tab[i] = new ThreadLocal.ThreadLocalMap.Entry(key, value);
	
	        int sz = ++size;
	
	        // cleanSomeSlots 清楚陈旧的Entry（key == null）
	        // 如果没有清理陈旧的 Entry 并且数组中的元素大于了阈值，则进行 rehash
	        if (!cleanSomeSlots(i, sz) && sz >= threshold)
	            rehash();
	    }

这个set()操作和我们在集合了解的put()方式有点儿不一样，虽然他们都是key-value结构，不同在于他们解决散列冲突的方式不同。集合Map的put()采用的是拉链法，而ThreadLocalMap的set()则是采用开放定址法。



> 题外话：什么是开放地址法？

![](https://i.imgur.com/M4GXZ5e.png)
![](https://i.imgur.com/m07Z4ex.png)
![](https://i.imgur.com/IqBruh4.png)

set()操作除了存储元素外，还有一个很重要的作用，就是replaceStaleEntry()和cleanSomeSlots()，这两个方法可以清除掉key == null 的实例，防止内存泄漏。在set()方法中还有一个变量很重要：threadLocalHashCode，定义如下：

		private final int threadLocalHashCode = nextHashCode();
从名字上面我们可以看出threadLocalHashCode应该是ThreadLocal的散列值，定义为final，表示ThreadLocal一旦创建其散列值就已经确定了，生成过程则是调用nextHashCode()：

	 private static AtomicInteger nextHashCode = new AtomicInteger();
	
	    private static final int HASH_INCREMENT = 0x61c88647;
	
	    private static int nextHashCode() {
	        return nextHashCode.getAndAdd(HASH_INCREMENT);
	    }

nextHashCode表示分配下一个ThreadLocal实例的threadLocalHashCode的值，HASH_INCREMENT则表示分配两个ThradLocal实例的threadLocalHashCode的增量，从nextHashCode就可以看出他们的定义。

### getEntry() ###
		private Entry getEntry(ThreadLocal<?> key) {
		            int i = key.threadLocalHashCode & (table.length - 1);
		            Entry e = table[i];
		            if (e != null && e.get() == key)
		                return e;
		            else
		                return getEntryAfterMiss(key, i, e);
		        }
由于采用了开放定址法，所以当前key的散列值和元素在数组的索引并不是完全对应的，首先取一个探测数（key的散列值），如果所对应的key就是我们所要找的元素，则返回，否则调用getEntryAfterMiss()，如下：

	    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
	            Entry[] tab = table;
	            int len = tab.length;
	
	            while (e != null) {
	                ThreadLocal<?> k = e.get();
	                if (k == key)
	                    return e;
	                if (k == null)
	                    expungeStaleEntry(i);
	                else
	                    i = nextIndex(i, len);
	                e = tab[i];
	            }
	            return null;
	        }
这里有一个重要的地方，当key == null时，调用了expungeStaleEntry()方法，该方法用于处理key == null，有利于GC回收，能够有效地避免内存泄漏。
## get() ##
	public T get() {
	        // 获取当前线程
	        Thread t = Thread.currentThread();
	
	        // 获取当前线程的成员变量 threadLocal
	        ThreadLocalMap map = getMap(t);
	        if (map != null) {
	            // 从当前线程的ThreadLocalMap获取相对应的Entry
	            ThreadLocalMap.Entry e = map.getEntry(this);
	            if (e != null) {
	                @SuppressWarnings("unchecked")
	
	                // 获取目标值        
	                T result = (T)e.value;
	                return result;
	            }
	        }
	        return setInitialValue();
	    }
首先通过当前线程获取所对应的成员变量ThreadLocalMap，然后通过ThreadLocalMap获取当前ThreadLocal的Entry，最后通过所获取的Entry获取目标值result。

getMap()方法可以获取当前线程所对应的ThreadLocalMap，如下：

	 ThreadLocalMap getMap(Thread t) {
	        return t.threadLocals;
	    }

## set(T value) ##
	 public void set(T value) {
	        Thread t = Thread.currentThread();
	        ThreadLocalMap map = getMap(t);
	        if (map != null)
	            map.set(this, value);
	        else
	            createMap(t, value);
	    }
获取当前线程所对应的ThreadLocalMap，如果不为空，则调用ThreadLocalMap的set()方法，key就是当前ThreadLocal，如果不存在，则调用createMap()方法新建一个，如下：

	 void createMap(Thread t, T firstValue) {
	        t.threadLocals = new ThreadLocalMap(this, firstValue);
	    }

## initialValue() ##
	  protected T initialValue() {
	        return null;
	    }
该方法定义为protected级别且返回为null，很明显是要子类实现它的，所以我们在使用ThreadLocal的时候一般都应该覆盖该方法。该方法不能显示调用，只有在第一次调用get()或者set()方法时才会被执行，并且仅执行1次。

由于ThreadLocal对象的set()方法设置的值只对当前线程可见，那有什么方法可以为ThreadLocal对象设置的值对所有线程都可见。

为此，我们可以通过ThreadLocal子类的实现，并覆写initialValue()方法，就可以为ThreadLocal对象指定一个初始化值。如下所示:

		private ThreadLocal myThreadLocal = new ThreadLocal<String>() {
		   @Override protected String initialValue() {
		       return "This is the initial value";
		   }
		};
此时，在set()方法调用前，当调用get()方法的时候，所有线程都可以看到同一个初始化值。

## remove() ##
	  public void remove() {
	        ThreadLocalMap m = getMap(Thread.currentThread());
	        if (m != null)
	            m.remove(this);
	    }
该方法的目的是减少内存的占用。当然，我们不需要显示调用该方法，因为一个线程结束后，它所对应的局部变量就会被垃圾回收。（但是使用的时候，为了防止内存泄漏以及线程被放回线程池等情况的发生，肯定要在try...finally中对ThreadLocal进行remove）

# ThreadLocal内存泄漏相关问题分析 #
> 问题：ThreadLocal为什么会内存泄漏？

答：首先上图

![](https://i.imgur.com/J4dkzlt.png)

前面提到每个Thread都有一个ThreadLocal.ThreadLocalMap的map，该map的key为ThreadLocal实例，它为一个弱引用，我们知道弱引用有利于GC回收。当ThreadLocal的key == null时，GC就会回收这部分空间，但是value却不一定能够被回收，因为他还与Current Thread存在一个强引用关系。

由于存在这个强引用关系，会导致value无法回收。如果这个线程对象不会销毁那么这个强引用关系则会一直存在，就会出现内存泄漏情况。所以说只要这个线程对象能够及时被GC回收，就不会出现内存泄漏。如果碰到线程池，那就更坑了。

那么要怎么避免这个问题呢？

在前面提过，在ThreadLocalMap中的setEntry()、getEntry()，如果遇到key == null的情况，会对value设置为null。当然我们也可以显示调用ThreadLocal的remove()方法进行处理。

