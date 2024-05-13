---
title: Semaphore深入浅出
date: 2018-05-15 23:22:59
tags: [Java,多线程]
categories: Java
---

### 1. Semaphore定义

**个人理解**：*同一时间内，限制指定数量线程通过*


### 2. Semaphore的同步性

``` java
package com.hc.thread.chapterOne.SemaPhore;

import java.util.concurrent.Semaphore;

/**
 * 同一时间内  限制多个线程通过
 */
public class SemaPhoreT {

    private Semaphore semaphore = new Semaphore(1);

    public void testMethod() {

        try {
            //限制
            semaphore.acquire();

            System.out.println(Thread.currentThread().getName() + " begin time:" + System.currentTimeMillis());
            Thread.sleep(1000);
            System.out.println(Thread.currentThread().getName() + " end time:" + System.currentTimeMillis());

            //释放
            semaphore.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

<img src="/img/thread/permits.png">

  类Semaphore的构造函数permits可以理解为同时刻通过的线程许可数,代表同一时间内最多允许多少个线程同时执行
  acquire()和release()之间的代码
  eg: 无参方法的作用是使用1个许可

``` java
public static void main(String[] args) {

        SemaPhoreT semaPhoreT = new SemaPhoreT();

        new Thread(new Runnable() {
            @Override
            public void run() {
                semaPhoreT.testMethod();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                semaPhoreT.testMethod();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                semaPhoreT.testMethod();
            }
        }).start();

    }
```

print：
``` bash
Thread-0 begin time:1526479020027
Thread-0 end time:1526479021028
Thread-1 begin time:1526479021028
Thread-1 end time:1526479022028
Thread-2 begin time:1526479022028
Thread-2 end time:1526479023029
```

可以看到打印信息依次输出，如果给为1个许可相当于这一段的时候是单线程的

我们改改: **private Semaphore semaphore = new Semaphore(2);**
print：
``` bash
Thread-0 begin time:1526479290849
Thread-1 begin time:1526479290850
Thread-0 end time:1526479291849
Thread-2 begin time:1526479291849
Thread-1 end time:1526479291850
Thread-2 end time:1526479292850
```
这个打印结果说明同一时刻是有0跟1两个线程通过acquire()和release()之间的

### 3. Semaphore实现生产者、消费者

Semaphore实现生产者、消费者模式的话还是比较简单的
我们以厨师、顾客来进行模拟这样一个场景，废话不多说，直接上代码：
``` java
package com.hc.thread.chapterOne.SemaPhore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 *  Semaphore实现生产者、消费者
 *  created by cheng on 2018/5/16
 */
public class RepastService {

    /**
     * 设置5个厨师（生产者）
     */
    private volatile Semaphore setSemaphore = new Semaphore(5);

    /**
     * 设置10个顾客（消费者）
     */
    private volatile Semaphore getSemaphore = new Semaphore(10);

    private volatile ReentrantLock reentrantLock = new ReentrantLock();
    private volatile Condition setCondition = reentrantLock.newCondition();
    private volatile Condition getCondition = reentrantLock.newCondition();

    /**
     * 最多一次上4盘菜
     */
    private volatile Object[] producePosition = new Object[4];

    private boolean isEmpty() {

        boolean isEmpty = true;
        for (int i = 0; i < producePosition.length; i++) {
            if (producePosition[i] != null) {
                isEmpty = false;
                break;
            }
        }
        return  isEmpty;

    }

    /**
     * 判断有没有空盘子
     * @return true：没有空盘子 反之则有
     */
    private boolean isFull() {

        boolean isFull = true;
        for (int i = 0; i < producePosition.length; i++) {
            if (producePosition[i] == null) {
                isFull = false;
                break;
            }
        }
        return  isFull;

    }

    public void set() {
        try {
            setSemaphore.acquire();
            reentrantLock.lock();
            while (isFull()) {
                //没有空盘子  厨师要等待
                setCondition.await();
            }

            for (int i = 0; i < producePosition.length; i++) {

                if (producePosition[i] == null) {

                    //发现有空盘子了  可以上菜
                    producePosition[i] = "xx菜";
                    System.out.println(Thread.currentThread().getName() + " 生产了 " + producePosition[i]);
                    break;

                }

            }
            //上菜
            getCondition.signalAll();
            reentrantLock.unlock();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            setSemaphore.release();
        }
    }

    public void get() {
        try {
            getSemaphore.acquire();
            reentrantLock.lock();
            while (isEmpty()) {
                //菜已经上齐了  没盘子装了  吃完了才能上
                getCondition.await();
            }

            for (int i = 0; i < producePosition.length; i++) {
                if (producePosition[i] != null) {
                    //发现有菜上来  可以开饭了
                    System.out.println(Thread.currentThread().getName() + " 消费了 " + producePosition[i]);
                    producePosition[i] = null;
                    break;
                }
            }
            //端盘子下去
            setCondition.signalAll();
            reentrantLock.unlock();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            getSemaphore.release();
        }
    }

}

```
**启动类Run.java**
``` java
package com.hc.thread.chapterOne.SemaPhore;

public class Run {

    public static void main(String[] args) throws InterruptedException {

        RepastService repastService = new RepastService();
        ThreadP[] arrayP = new ThreadP[20];
        ThreadC[] arrayC = new ThreadC[20];

        for (int i = 0; i < 20; i++) {
            arrayP[i] = new ThreadP(repastService);
            arrayC[i] = new ThreadC(repastService);
        }

        Thread.sleep(2000);

        for (int i = 0; i < 20; i++) {
            arrayP[i].start();
            arrayC[i].start();
        }

    }

}
```

**线程类ThreadP.java**
``` java
package com.hc.thread.chapterOne.SemaPhore;

public class ThreadP extends Thread {

    private RepastService service;

    public ThreadP(RepastService service) {
        super();
        this.service = service;
    }

    @Override
    public void run() {
        service.set();
    }
}

```
**ReentrantLock**跟**Condition**大家不明白的先自行度娘，后面我抽空再补上，emmmm...
ReentrantLock在这里再做一重锁的判断，确保生产者跟消费者都是平衡的
如果不加ReentrantLock会怎么样，因为我们是用的Condition进行一个相互唤醒的操作，不用ReentrantLock的话可能会报**IllegalMonitorStateException**的异常

输出的话就给大家截个图：
<img src="/img/thread/out.png">


### 4.总结

Semaphore semaphore = new Semaphore(1)其实只是初始化多少个许可
acquire()相当于动态的减少许可，相应的release()可以动态的增加许可
Semaphore提供的限制并发线程的功能，此功能在默认的synchronized种是不提供的

[源码地址](https://github.com/Yiaichen/javaDemo/tree/master/javaDemo)
源码在thread目录下,还有一些其他demo,大家可自行学习

