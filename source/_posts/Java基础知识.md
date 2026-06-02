---
title: Java 基础知识
date: 2025-03-11
tags:
  - Java
categories:
  - 学习
---

## 整理一些不熟悉但重要的 Java 基础知识（自用）

### 移位运算符
移位运算符是最基本的运算符之一，几乎每种编程语言都包含这一运算符。
移位操作中，被操作的数据被视为二进制数，移位就是将其向左或向右移动若干位的运算。

移位运算符有两种: >>/<< 或者 >>>
* >>/>>>: >> 将被操作数的2进制位全体右移1位，相当于进行 /2 操作，>>> 的作用基本和 >> 一样，只不过如果被操作数是负数，>> 和 >>> 在高位右移时被操作数会有不同的表现
  * >>: 被操作数如果是正数，则高位补0；否则高位补1
  * >>>: 无符号右移，管你被操作数正数负数，直接补0
  * 由于 >> 和 >>> 对负数的操作不同，这就导致了被操作数是负数的场景，这两个操作得出的结果截然不同
* <<: << 将被操作数的2进制位全体左移1位，相当于进行 *2 操作
  * 由于 << 是左移操作，有可能会遇到溢出的情况
  * 例如对于正整数1，其二进制表达式为 00000000000000000000000000000001，如果对1进行左移操作： 1 << 31
  * 最后就会出现最高位为1的情况：10000000000000000000000000000000，此时变为了 -2147483648，从正整数变成了负整数，也就是溢出
  * 对于负整数左移，同样也会出现溢出的情况，就拿上面的 -2147483648 来说，-2147483648 << 1 = 0，也是因为溢出
针对于移位运算来说，移位的次数如果大于被移位的数的位数，则进行取模操作。如 Integer 是32位，对一个 Integer << 48，相当于 << (48 % 32) = << 16

```java
public class Test {
    public static void main(String[] args) {
        int a;
        
        // 正整数无符号右移（高位补0）
        a = 4;
        System.out.println("正整数无符号右移（高位补0）");
        System.out.println("a = " + a + " binaries: " + Integer.toBinaryString(a));   // 4 | 00000000 00000000 00000000 00000100 
        System.out.println("a = " + (a >>> 1) + " binaries: " + Integer.toBinaryString(a >>> 1));  // 2 | 00000000 00000000 00000000 00000010
  
        // 负数无符号右移（高位补0）
        a = -4;
        System.out.println("负数无符号右移（高位补0）");
        System.out.println("a = " + a + " binaries: " + Integer.toBinaryString(a));  // -4 | 11111111 11111111 11111111 11111100 
        System.out.println("a = " + (a >>> 1) + " binaries: " + Integer.toBinaryString(a >> 1));  // 2147483646 | 01111111 11111111 11111111 11111110
      
        // 正整数右移（高位补0）
        a = 4;
        System.out.println("正整数右移（高位补0）");
        System.out.println("a = " + a + " binaries: " + Integer.toBinaryString(a));   // 4 | 00000000 00000000 00000000 00000100 
        System.out.println("a = " + (a >> 1) + " binaries: " + Integer.toBinaryString(a >> 1));  // 2 | 00000000 00000000 00000000 00000010
      
        a = -4;
        // 负数右移（高位补1）
        System.out.println("负数右移（高位补1）");
        System.out.println("a = " + a + " binaries: " + Integer.toBinaryString(a));  // -4 | 11111111 11111111 11111111 11111100 
        System.out.println("a = " + (a >> 1) + " binaries: " + Integer.toBinaryString(a >> 1));  // -2 | 11111111 11111111 11111111 11111110

        // 正整数左移
        a = 4;
        System.out.println("正整数左移");
        System.out.println("a = " + a + " binaries: " + Integer.toBinaryString(a)); // 4 | 00000000 00000000 00000000 00000100
        System.out.println("a = " + (a << 1) + " binaries: " + Integer.toBinaryString(a << 1)); // 8 | 00000000 00000000 00000000 00001000
      
        // 负数左移
        a = -4;
        System.out.println("负数左移");
        System.out.println("a = " + a + " binaries: " + Integer.toBinaryString(a)); // -4 | 11111111 11111111 11111111 11111100
        System.out.println("a = " + (a << 1) + " binaries: " + Integer.toBinaryString(a << 1)); // -8 | 11111111 11111111 11111111 11111000

        // 正整数左移溢出
        a = 1;
        System.out.println("正整数左移溢出");
        System.out.println("a = " + a + " binaries: " + Integer.toBinaryString(a)); // 1 | 00000000 00000000 00000000 00000001
        System.out.println("a = " + (a << 31) + " binaries: " + Integer.toBinaryString(a << 31)); // -2147483648 | 10000000 00000000 00000000 00000000

        // 负整数左移溢出
        a = -2147483648;
        System.out.println("负整数左移溢出");
        System.out.println("a = " + a + " binaries: " + Integer.toBinaryString(a)); // -2147483648 | 10000000 00000000 00000000 00000000
        System.out.println("a = " + (a << 1) + " binaries: " + Integer.toBinaryString(a << 1)); // 0 | 00000000 00000000 00000000 00000000
    }
}
```

使用移位运算符的主要原因：
* 高效：移位运算符直接对应于处理器的移位指令。
  现代处理器具有专门的硬件指令来执行这些移位操作，这些指令通常在一个时钟周期内完成。
  相比之下，乘法和除法等算术运算在硬件层面上需要更多的时钟周期来完成。
* 节省内存：通过移位操作，可以使用一个整数（如 int 或 long）来存储多个布尔值或标志位，从而节省内存。

### try-catch-finally
尽量不要在 finally 块中使用 return 语句，否则在 try 和 catch 块中定义的 return 语句不会生效

```java
public static String test() {
    try {
        // throw new Exception("");
        return "111";
    } catch (Exception e) {
        return "222";
    } finally {
        return "333";
    }
}
```

上述代码中不管抛不抛出异常，都只会输出 333

### HashMap 链表转红黑树机制
当 HashMap 中某个节点的链表长度 > 8 时，会进行当前节点是否需要将链表转为红黑树的判断
* 当数组长度 < 64 时，将当前数组长度 * 2
* 当数组长度 >= 64 时，将当前链表转红黑树

也就是说，假如有以下代码
```java
public void test() {
    HashMap<T, String> hashMap = new HashMap<>();
    for (int i = 0; i < 12; i++) {
        T t = new T();
        hashMap.put(t, "2");
    }
}

static class T implements Delayed {

    // 重写 hashCode 方法，使得在 hashMap 中所落的节点都是同一个
    @Override
    public int hashCode() {
        return 1;
    }
    
}
```

当 i = 0 时，hashMap.put() 方法会触发 resize() 方法，此时由于未初始化 hashMap 长度，默认会将数组长度初始化成 16
当 i = 8 时，hashMap 中某节点的链表长度正好为 9，此时会触发转红黑树判断，由于数组长度 = 16 < 64，此时触发 resize() 方法将数组长度 * 2 = 32
当 i = 9 时，hashMap 中某节点的链表长度为 10，此时会触发转红黑树判断，由于数组长度 = 32 < 64，此时触发 resize() 方法将数组长度 * 2 = 64
当 i = 10 时，hashMap 中某节点的链表长度为 11，此时会触发转红黑树判断，由于数组长度 64 不小于 64，此时触发链表转红黑树的机制
当 i = 11 时，hashMap 中某节点的链表已经转成了红黑树，debug 看确实也走的红黑树添加节点方法

### Condition 的使用
```java
private static final ReentrantLock lock = new ReentrantLock();

private static final Condition lock1 = lock.newCondition();

private static final Condition lock2 = lock.newCondition();

private static final Condition lock3 = lock.newCondition();

public static void main(String[] args) {
    Thread thread1 = new Thread(() -> {
        try {
            Thread.sleep(2000);
        } catch (Exception ignored) {}

        while (true) {
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + ": ready sleep");
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getName() + ": after sleep");
            } catch (Exception e) {
                e.printStackTrace();
            }

            try {
                System.out.println(Thread.currentThread().getName() + ": ready notify lock2");
                lock2.signal();
                System.out.println(Thread.currentThread().getName() + ": notify lock2, ready wait");
                lock1.await();
                System.out.println(Thread.currentThread().getName() + ": after wait");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }, "thread1");

    Thread thread2 = new Thread(() -> {
        try {
            Thread.sleep(2000);
        } catch (Exception ignored) {}

        while (true) {
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + ": ready sleep");
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getName() + ": after sleep");
            } catch (Exception e) {
                e.printStackTrace();
            }

            try {
                lock.lockInterruptibly();
                System.out.println(Thread.currentThread().getName() + ": ready notify lock3");
                lock3.signal();
                System.out.println(Thread.currentThread().getName() + ": notify lock3, ready wait");
                lock2.await();
                System.out.println(Thread.currentThread().getName() + ": after wait");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }, "thread2");

    thread1.start();
    thread2.start();

    while (true) {
        lock.lock();

        try {
            System.out.println(Thread.currentThread().getName() + ": ready sleep");
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getName() + ": after sleep");
        } catch (Exception e) {
            e.printStackTrace();
        }

        try {
            System.out.println(Thread.currentThread().getName() + ": ready notify lock1");
            lock1.signal();
            System.out.println(Thread.currentThread().getName() + ": notify lock1, ready wait");
            lock3.await();
            System.out.println(Thread.currentThread().getName() + ": after wait");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### interrupt
```java
public static void main(String[] args) {
    Thread thread = new Thread(() -> {
        while (!Thread.currentThread().isInterrupted()) {
            System.out.println("interrupted");
        }
    });

    thread.start();
    try {
        Thread.sleep(2000);
    } catch (Exception ignored) {}
    thread.interrupt();
}
```