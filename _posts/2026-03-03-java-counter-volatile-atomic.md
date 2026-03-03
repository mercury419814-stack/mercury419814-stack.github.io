---
title: Java 并发代码 snippet 分析：Counter 与 volatile
categories: [工作总结]
tags: [并发, java, volatile]
---

```java
public class Counter {
	private int count = 0;
	
	private boolean stopped = false;
	
	public void increment() {
		if (!stopped) {
			count = count + 1;
		}
	}
	
	public void stop() {
		stopped = true;
	}
	
	public int getCount() {
		return count;
	}

}
```

**1. 关于 count 的竞态条件（Race Condition）**

count++ 实际上包含了三个步骤：

1. **Read**: 从主内存读取 count 到工作内存。
2. **Modify**: 在 CPU 寄存器中执行 +1 操作。
3. **Write**: 将新值写回工作内存。
    
如果没有锁，两个线程可能同时读取到 0，都计算出 1，写入后 count 变成了 1（本该是 2）。即多线程并发时会出现丢失更新，导致结果比预期小的情况。

**2. 关于 stopped 的可见性与重排序（Visibility & Reordering）**

stopped 变量在没有同步措施时，修改可能对其他线程不可见（读不到最新的值），或者因为指令重排序导致读取逻辑错误。建议加 volatile。

在 Java 内存模型（JMM）中，没有 volatile 修饰的变量，线程可能会一直从自己的**CPU 缓存（工作内存）中读取 stopped 的旧值（false），导致 stop() 方法虽然被调用了，但循环或判断迟迟停不下来。volatile 保证了可见性（立即刷新回主内存）和有序性（禁止指令重排序）。

更优解是使用 Atomic 类，AtomicInteger 使用 CAS（Compare-And-Swap）非阻塞算法，性能通常优于 synchronized。

即，修改成这样就行：

```java
// 使用原子类保证 increment 的原子性
private AtomicInteger count = new AtomicInteger(0);
    
// 使用 volatile 保证可见性
private volatile boolean stopped = false;
```

要区分 **“赋值操作”** 和 **“复合操作（Read-Modify-Write）”**。

1. **复合操作（危险）**：
    
    比如 count++ 或者 flag = !flag（取反）。
    
    - 这种操作依赖于变量的**旧值**。
    - 多线程同时做这个操作，会出现竞态条件。
    - **结论**：这种场景下，volatile 确实不够，必须用 Atomic 类或锁。
2. **纯赋值操作（安全）**：
    
    看一下你的代码：
    
    `public void stop() {
        stopped = true; // 这是一个单纯的赋值，不依赖 stopped 原来的值
    }`
    
    - 无论多少个线程同时调用 stop()，它们做的动作都是把 stopped 变成 true。
    - 如果不依赖旧值（即不需要“先读旧值，再计算，再写新值”），那么这个操作本身就是**原子**的（Java 规范保证了对 boolean、int 等引用的赋值是原子的）。
    - 既然赋值已经是原子的，我们唯一剩下的问题就是**可见性**（其他线程能不能立马看到）。
    - **结论**：volatile 解决了可见性问题，所以这就够了。

