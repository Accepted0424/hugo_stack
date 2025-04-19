---
title: BUAA-OO-Unit2
description: 电梯调度
slug: OO-unit2
date: 2025-04-19 18:28:00+0000
image: cover.jpg
categories:
    - BUAA-OO
tags:
    - BUAA-OO
    - Java
    - 面向对象
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# BUAA-OO-Unit2总结

## 三次迭代回顾

**第五次作业**

本单元以某公司大楼的**目的选层电梯系统**为背景。

> 目的选层电梯系统的运行流程：
>
> 1. 进入电梯前，乘客输入目标楼层；
> 2. 系统根据设定的策略指定最合适的电梯来服务，将结果告知乘客；
> 3. 乘客进入指定电梯后，无需再次按楼层按钮，电梯会自动将其送至目标楼层。

与普通的目的选层电梯系统有所区分的是，该公司为提高运行效率，充分考虑不同职位的员工所从事工作的时间急迫性，为不同类型的乘客请求赋予了**优先级指数**。优先级指数描述了乘客到达目的楼层的急迫程度。

**第六次作业**

>  本次作业的新增内容有：
>
> 1. 取消“乘客请求指定电梯”约束，需要自行设计电梯分配策略
> 2. 新增“临时调度”请求；新增“RECEIVE”约束
> 3. 修改乘客请求格式；修改“OUT”指令输出格式；修改测试数据约束；修改“实现提示”“提示与警示”部分

**第七次作业**

> 本次作业的新增内容有：
>
> - 新增双轿厢电梯改造
> - 修改"RECEIVE"约束；修改临时调度约束

**性能要求**

> 抛开麻烦的公式，简单来说就是
>
> - 运行时间尽量短
> - 尽量满足乘客请求的优先级
> - 尽量减少系统的无效运行



## 生产者-消费者模型

程序在运行中任何时刻都有可能收到用户请求，我们不能收到一个请求就立刻将其分配给某个电梯去处理，因为电梯处理一个请求需要时间，如果在处理一个请求的过程中接收到了一个请求该怎么办呢？是放弃当前请求还是放弃新接受到的请求？显然都不合理。

于是我们想到可以给每部电梯新建一个待处理的容器（队列、数组等等），里面放着分配给该部电梯的乘客请求。我们收到一个请求只需要把该请求放入对应电梯的待处理容器里即可，而对于电梯来说，它的任务只有一个，那就是把这个容器里的请求全部处理完即可。于是我们很自然的实现了一个多线程中的经典设计模型**"生产者-消费者模型"**。

在第一次作业中，由于每一个请求都有指定的电梯，所以我们可以直接把请求放在对应电梯的待处理容器里。而在后续两次迭代中，需要我们自己决定乘客请求分配给哪个电梯，并且已经分配的请求有可能被取消需要被再次分配，因此我们还需要一个分配线程（Dispatch Thread）。

综上，在本次电梯单元，我们需要用到两个**生产者-消费者模型**

- **输入线程**作为生产者，**分配线程**作为消费者。输入线程负责从外部读取用户请求，这些请求被放入⼀个输入缓冲区（请求队列）中。分配线程持续从该缓冲区中取出请求，进行传递sche和update信号、选择电梯等操作，并将调度结果发送到另⼀个调度缓冲区（电梯的待分配队列）。

- **分配线程**作为生产者，六个**电梯线程**作为消费者。分配线程将这些任务传递给具体的**电梯线程**，后者执行具体的操作，如移动电梯、开关门、响应请求等。



## 多线程中的同步块与锁

在学习了OS之后，我们发现 Java 中的`synchronized` 关键字实现的就是一种管程（Monitor）机制，更准确地说，它实现的是一种**MESA风格的管程**。相比于Hoare管程，MESA管程的特点是当一个线程被 `notify()` 或 `notifyAll()` 唤醒时，它只是被放入了就绪队列，还要等待重新竞争锁，我们在设计时需要考虑这个问题。

synchronized是Java中的关键字，告诉JVM要对这个语句块进行线程互斥控制，其后的括号内需要一个对象用作锁，获取到锁才会执行代码块内的代码，执行结束后会释放锁。

```java
synchronized(object) { // 获取锁
    ...
} // 释放锁
```

> **为什么在Object.java中找不到和锁相关的代码？**
>
> 在 Java 里，每个对象都自动带有一个对象监视器（Monitor），这个机制由 JVM 层面（C++ 代码）实现，而不是在 `Object.java` 源码中。
>
> Java 的 `synchronized` 依赖于 **对象头（Object Header）** 中的 **Mark Word** 来存储锁状态。
>
> 每个 Java 对象在 JVM 内存中存储结构如下：
>
> | **对象头** (Header)                     | **实例数据** | **对齐填充** |
> | --------------------------------------- | ------------ | ------------ |
> | Mark Word（存储锁状态） + Class Pointer | 成员变量     | 对齐数据     |
>
> 其中：
>
> - **Mark Word** 记录了**对象的锁状态**、线程 ID、GC 信息等。
> - 当 `synchronized` 代码块运行时，JVM 会修改 `Mark Word`，进行加锁和解锁操作。

synchronized的使用方法有以下三种：

```java
// 关键字在实例方法上，锁为当前实例
public synchronized void instanceLock() {
    // code
}

// 关键字在静态方法上，锁为当前Class对象
public static ssynchronized void classLock() {
    // code
}

// 关键字在代码块上，锁为括号里面的对象
public void blockLock() {
    Object o = new Object();
    synchronized (o) {
        // code
    }
}
```

在三次作业中，我的所有同步设计都是通过synchronized关键字实现的，其实为了更好的性能应该使用读写锁分开处理读操作和写操作，但是由于时间原因就没有去重构。

什么时候需要加锁是一个很关键的问题，笔者认为如果一个对象或数据可能同时被多个线程修改，或者多个线程同时读写该对象的状态，就应该加锁，或者使用线程安全的数据结构或机制。在三次作业中，RequestQueue（也就是电梯的待处理队列）就是需要重点关注的对象，因为分配线程和电梯线程都会对其进行访问。我们对其中可能造成数据冲突的方法都加上synchronized关键字，确保这些方法之间的互斥，这样我们就可以称RequestQueue类是**线程安全**的，这样我们在使用该类的方法时就无需考虑同步问题。

当然也需要注意不能无脑加锁，把所有方法都加上synchronized关键字固然能保证没有数据冲突，但这样会大幅降低并发效率，可能会在一些情况下超出课程组给出的一些时间限制。



## 策略设计

### 电梯调度策略

在三次作业中，每部电梯最多容纳6个乘客，我们该怎么保证在所有乘客都能到达目的地的情况下电梯运行时间尽可能少，优先级高的乘客能尽快到达目的地，并且尽可能减少耗电量。

最常规的电梯调度策略是**Look算法**。

> **SCAN 算法**（也称电梯算法）是一种磁盘调度算法，用于确定磁盘臂和磁头在处理读写请求时的运动。它让电梯在最底层和最顶层之间连续往返运行，在运行过程中响应处在于电梯运行方向相同的各楼层上的请求。
>
> **LOOK算法**是对SCAN算法的改进，也是一种硬盘调度算法。当LOOK算法发现电梯所移动的方向上不再有请求时立即改变运行方向，而扫描算法则需要移动到最底层或者最顶层时才改变运行方向。

现实中大多数电梯都采用LOOK算法或优化后的LOOK算法，但我们发现这个算法并没有考虑乘客的优先级。假设当前电梯在F1下行，电梯里有一个优先级为1的乘客正在前往B4，而这是有一个优先级为100的乘客在F2发出请求想要前往F7，显然这时候我们应该优先让电梯服务优先级为100的乘客。

于是笔者尝试自己改造LOOK算法。仔细分析后，我们发现电梯调度问题的关键其实就是**何时反转方向**。不过需要注意的是我们在设计算法时不能针对某种特殊情况去优化，这样容易过拟合（Overfitting），即特殊情况下性能优越，但是平均性能有所下降。

以下是笔者的一次算法优化尝试：

- 对于电梯内的乘客，我们应该优先送优先级足够大的乘客，确保优先级大的乘客能尽快到达目的地，而不会被优先级太小的乘客拖长到达时间。具体实现方法为如果电梯里非空，比较向上的优先级和向下的优先级来确定下一步的移动方向。

$$
Priority\_up = \sum_{toFLoor > curfloor} \frac{priority}{toFloor - curFloor}(对于电梯内所以目标楼层大于当前楼层的乘客)
$$

$$
Priority\_up = \sum_{toFLoor < curfloor} \frac{priority}{curFloor-toFloor}(对于电梯内所以目标楼层小于当前楼层的乘客)
$$

- 对于电梯外的请求，我们应该先接优先级大且行程（ $|toFloor - fromFloor|$ ）较大的乘客。具体实现方法为如果电梯内部没人且电梯的待处理队列非空，则遍历待处理队列，找出综合优先级最大的乘客，根据该乘客的楼层确定电梯运行方向。

$$
Priority\_complex = priority \times |toFloor - fromFloor|
$$

- 电梯内部满员时，我们可以让优先级足够小的乘客中途下电梯，让优先级足够大的乘客上去。具体实现方法为每到一层，先让到达目的地的乘客下来，如果电梯满员且该层存在未处理请求，则依次比较该层的所有请求乘客，如果其优先级为电梯内优先级最小的乘客的**5倍**以上，则让电梯内优先级最小的乘客下来，让对应的外部请求乘客上电梯。

其实这是一次比较失败的算法优化，在一些极端情况性能确实优于LOOK算法，在中测时性能和LOOK相当，但在强测数据量较大时普遍不如LOOK算法。



### 分配策略

在第六和第七次作业中，我们需要自己决定一个请求应该分配给哪部电梯。这就涉及到一个问题，分配给哪部电梯才能保证性能分尽可能高。

- 如果其他5部电梯都正在sche或update，如果这时突然来了100个请求，这时100个请求全部分配给一个电梯会导致整体运行时间过长。我们应该给电梯接收乘客请求的数量设置一个最大值，在所有电梯都不接收请求时分配线程进入等待状态，某个电梯空闲时唤醒分配线程。
- 对于第七次作业，电梯的可达楼层范围发生了变化，我们在分配时的条件也应该发生变化。我们应该首选可达fromFloor和toFloor的电梯接收请求，减少换乘带来的时间和电量损失，次选可达fromFloor，确保电梯可以接到对应乘客。



## 调度器（Dispatch）设计

### 结束和等待条件

- 未分配队列为空，但输入还未结束，应该进入等待状态，避免轮询。
- 因为所有电梯暂时无法接收乘客请求而无法继续分配，但是未分配队列非空，应该进入等待，避免轮询。
- 输入结束且未分配队列，此时应该跳出循环结束dispatch线程。

```java
public void run() {
        while (true) {
            // 输入未结束，还有可能获取请求
            while (isEmpty() && !isEnd()) {
                try {
                    dispatchWait();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
            // 所有电梯处于忙碌状态，但是还有未分配请求，等待空闲电梯
            while (allElevatorsBusy && !isEmpty()) {
                synchronized (busyLock) {
                    try {
                        busyLock.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
            }
            // 输入结束，且没有未分配队列，告知电梯的已分配队列不会再有来自dispatch的分配
            if (isEnd() && isEmpty()) {
                for (int i = 1; i <= 6; i++) {
                    elevators[i].getRequestQueue().setEnd();
                }
                // 等待所有子任务结束
                for (Future<?> f : futures) {
                    try {
                        f.get();
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                }
                // 关闭线程池
                executor.shutdown();
                break;
            }
            // 分配操作
            try {
                dispatch();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }
```

其实分配线程有点类似 Java 中的**守护（Daemon）线程**，其必须要等到其他所有线程都结束之后才能结束，其必须要在所有电梯线程都结束之后才能结束。



### 接收乘客请求

提供统一的`offer`接口，通过传参区分不同行为。

对于乘客请求，其所在楼层是会发生变化的，由于我们无法直接修改personRequest实例的fromFloor，我们可以通过一个map来记录每个乘客的当前楼层。

请求有以下分类：

- 乘客请求（PersonRequest）
  - 首次请求，nowFloorMap记录的楼层即为请求的fromFloor，需要记入总请求数
  - 被取消的请求，nowFloorMap记录的楼层为乘客当前的楼层（通过参数传入），无需记入总请求数
  - 中途下电梯的乘客请求，nowFloorMap记录的楼层为乘客当前的楼层（通过参数传入），无需记入总请求数
- 调度请求（ScheRequest）
- 双轿厢改造请求（UpdateRequest）

```java
public synchronized void offer(Request r, Boolean isRearrange, Boolean first, int nowFloor) {
    if (r instanceof PersonRequest) {
        PersonRequest pr = (PersonRequest) r;
        unDispatchQueue.offer(pr);
        if (first) {
            nowFloorMap.put(pr, intOf(pr.getFromFloor()));
        }
        if (isRearrange) {
            nowFloorMap.put(pr, nowFloor);
        }
        if (!isRearrange && first) {
            personRequestReceive++;
        }
    } else if (r instanceof ScheRequest) {
        ScheRequest sr = (ScheRequest) r;
        unDispatchSche.offer(sr);
    } else if (r instanceof UpdateRequest) {
        UpdateRequest ur = (UpdateRequest) r;
        try {
            elevators[ur.getElevatorAId()].acceptUpdate(ur);
            elevators[ur.getElevatorBId()].acceptUpdate(ur);
        } catch (Exception e) {
            e.printStackTrace();
        }
        unDispatchUpdate.add(ur);
        hasFreeElevator();
    }
    notifyAll();
}
```



### 分配请求

- 对于调度请求和乘客请求，我们直接将请求放入电梯的未分配队列即可
- 对于双轿厢改造请求，我们通过线程池来实现异步操作（后续详细解释）。

```java
private synchronized void dispatch() throws InterruptedException {
        while (!unDispatchSche.isEmpty()) {
            ScheRequest sr = unDispatchSche.peek();
            elevators[sr.getElevatorId()].getRequestQueue().offer(unDispatchSche.poll(),  0);
        }
        while (!unDispatchUpdate.isEmpty()) {
            UpdateRequest ur = unDispatchUpdate.poll();
            Future<?> future = executor.submit(() -> {
                try {
                    elevators[ur.getElevatorAId()].wait2clearInside();
                    elevators[ur.getElevatorBId()].wait2clearInside();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                TimableOutput.println(String.format("UPDATE-BEGIN-%d-%d",
                    ur.getElevatorAId(), ur.getElevatorBId()));
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                TimableOutput.println(String.format("UPDATE-END-%d-%d",
                    ur.getElevatorAId(), ur.getElevatorBId()));
                elevators[ur.getElevatorAId()].updateDone();
                elevators[ur.getElevatorBId()].updateDone();
            });
            futures.add(future);
        }
        // 分配给最近的空闲电梯
        if (!unDispatchQueue.isEmpty()) {
            PersonRequest pr = unDispatchQueue.peek();
            // 获取被分配电梯的序号
      		......
            TimableOutput.println(
                String.format("RECEIVE-%d-%d", pr.getPersonId(), elevators[target].getId()));
            elevators[target].getRequestQueue().offer(unDispatchQueue.poll(), nowFloorMap.get(pr));
        }
        notifyAll();
    }
```



## 双轿厢改造

"将一部电梯移动另一部电梯的电梯井中"的操作并不需要体现在代码中，我们只需要改变两部电梯的可达楼层范围即可。

### 输出同步

同步输出是双轿厢电梯改造中比较麻烦的一点，因为其要求是两部电梯分别改造，但是需要统一输出。

具体有两种实现方式：

- 利用信号量进行线程间通信，当两个电梯都完成对应操作后统⼀输出

- 异步处理，让电梯在处理完之后sleep一段时间确保二者都完成了对应操作之后由分配线程输出。

笔者采用的第二种方法：

- 在电梯线程接收到update请求后，立即告知两部电梯，让其尽快停下来，并且不再接收请求，因为update请求优先级大于乘客请求，所以要让电梯从等待中唤醒（如果所有电梯都不能接收乘客请求了）。
- 分配时，因为处理过程中涉及到分配线程的sleep，为了避免sleep期间不能接收请求的问题，我们创建一个子线程处理，子线程sleep并不影响分配线程。
- 等待两部电梯线程处于静止状态并清空内部所有的乘客之后由分配器子线程统一输出BEGIN，接着子线程会sleep一秒，确保满足改造时间限制。
- 电梯线程在清理完内部乘客之后也应该sleep1秒（最好大于1秒），因为BEGIN-END之间电梯是不能移动的，要等到输出END之后再输出。

```java
// Dispatch.java

private ExecutorService executor = Executors.newFixedThreadPool(5);

public synchronized void offer(Request r, Boolean isRearrange, Boolean first, int nowFloor) {
        if (r instanceof PersonRequest) {
            ...
        } else if (r instanceof ScheRequest) {
            ...
        } else if (r instanceof UpdateRequest) {
            UpdateRequest ur = (UpdateRequest) r;
            try {
                elevators[ur.getElevatorAId()].acceptUpdate(ur);
                elevators[ur.getElevatorBId()].acceptUpdate(ur);
            } catch (Exception e) {
                e.printStackTrace();
            }
            unDispatchUpdate.add(ur);
            hasFreeElevator();
        }
        notifyAll();
    }


private synchronized void dispatch() throws InterruptedException {
        while (!unDispatchSche.isEmpty()) {...}
    
        while (!unDispatchUpdate.isEmpty()) {
            UpdateRequest ur = unDispatchUpdate.poll();
            Future<?> future = executor.submit(() -> {
                try {
                    elevators[ur.getElevatorAId()].wait2clearInside();
                    elevators[ur.getElevatorBId()].wait2clearInside();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                TimableOutput.println(String.format("UPDATE-BEGIN-%d-%d",
                    ur.getElevatorAId(), ur.getElevatorBId()));
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                TimableOutput.println(String.format("UPDATE-END-%d-%d",
                    ur.getElevatorAId(), ur.getElevatorBId()));
                elevators[ur.getElevatorAId()].updateDone();
                elevators[ur.getElevatorBId()].updateDone();
            });
            futures.add(future);
        }
    
        // 分配给最近的空闲电梯
        if (!unDispatchQueue.isEmpty()) {...}
        notifyAll();
    }
```

```java
// Elevator.java

private void executeUpdate() throws InterruptedException {
        inUpdate = true;
        updateHasBegin = true;
        if (!insideQueue.isEmpty()) {
            TimableOutput.println(String.format("OPEN-%s-%d", formatFloor(curFloor), id));
            Thread.sleep(minTimeOpen2Close);
            allPersonOut();
            TimableOutput.println(String.format("CLOSE-%s-%d", formatFloor(curFloor), id));
        }
        insideHasClear();
        hasAcceptUpdate = false;
        Thread.sleep(1100);
        updateParam();
        removeAllReceive();
        inUpdate = false;
    }
```



### 换乘层限制

- 线程间通信确保在同一电梯井的电梯的不会同时出现在换乘层。
  - 需要注意的是，在电梯前往换乘层的过程中就应该判定换乘层已经被占用。

- 电梯不应在换乘层等待，到达换乘层完成对应操作后应尽快离开。

```java
		Status status = update();
        if (status == Status.WAIT && curFloor == transferFloor && afterUpdate) {
            status = canMove() ? Status.MOVE : canReverse() ? Status.REVERSE : Status.WAIT;
        }
        switch (status) {
            case UPDATE:
                ...
                break;
            case OPEN:
                ...
                break;
            case MOVE:
                if ((curFloor + 1 == 0 && curFloor + 2 == transferFloor) ||
                    curFloor + 1 == transferFloor) {
                    transferFloorIsOccupied = true;
                }
                modifyFloor(true, false, false, 0);
                Thread.sleep(timePerFloor);
                TimableOutput.println(String.format("ARRIVE-%s-%d", formatFloor(curFloor), id));
                if (curFloor != transferFloor) {
                    transferFloorIsOccupied = false;
                }
                break;
            case REVERSE:
                if ((curFloor - 1 == 0 && curFloor - 2 == transferFloor) ||
                    curFloor - 1 == transferFloor) {
                    transferFloorIsOccupied = true;
                }
                modifyFloor(false, true, false, 0);
                Thread.sleep(timePerFloor);
                TimableOutput.println(String.format("ARRIVE-%s-%d", formatFloor(curFloor), id));
                if (curFloor != transferFloor) {
                    transferFloorIsOccupied = false;
                }
                break;
            default:
                break;
        }
```



## 架构设计

整体架构比较简单，只有五个类

- MainClass主线程接收输入
- Dispatch线程对不同的请求进行分类处理
- RequestQueue类是电梯的待处理队列，每个电梯实例都有一个RequestQueue实例作为缓冲区
- Elevator线程对请求进行处理

笔者在设计的过程中找不到理由再抽象出一个类（可能OOP的思想还没学透彻吧），所以大量的处理逻辑集中在Elevator类中，导致Elevator类的代码行数接近500行。

**HW5** 无分配线程，输入线程直接分配给电梯的待处理队列，当时认为完全没有必要引入分配线程，没能考虑后续迭代问题

**HW6** 新增分配线程，在分配线程和电梯线程中增加调度逻辑

**HW7** 在分配线程和电梯线程中新增Update异步处理逻辑

三次迭代中整体架构设计变化不大，所以这里只放最后一次的设计架构：

![exported_from_idea.drawio (2)](https://lhy-blog-image.oss-cn-beijing.aliyuncs.com/undefinedexported_from_idea.drawio%20(2).png)

时序图（sequence diagram）：

![sequenceDiagram.drawio](https://lhy-blog-image.oss-cn-beijing.aliyuncs.com/undefinedsequenceDiagram.drawio.png)

## Debug

**输出顺序**

在多线程中print的顺序十分重要，因为当前线程的操作会影响到其他线程的运行，且题目中有着严格的输出循序要求。一些print需要在操作之后，一些需要在操作之前。

![](https://lhy-blog-image.oss-cn-beijing.aliyuncs.com/undefined%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-04-19%20111357.png)

![](https://lhy-blog-image.oss-cn-beijing.aliyuncs.com/undefined%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-04-19%20110913.png)



**楼层计数**

把楼层转化为int类型后，使用时不能直接用floor+1或floor-1，因为F1（+1）的下一层是B1（-1）而不是0。

![](https://lhy-blog-image.oss-cn-beijing.aliyuncs.com/undefined%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-04-19%20111931.png)



### debug方法

由于程序的输出需要通过官方程序，调试起来很麻烦，所以我大部分的debug都是通过添加输出来实现的。

由于Java好像没有全局变量，只好在MainClass里声明一个public static final类型的变量来近似实现全局变量的效果。

声明一个全局debug控制开关，用来控制是否显示调试信息。

```java
// MainClass.java

public class MainClass {
    // debug info
    public static final boolean debug = false;
    public static final String RESET = "\u001B[0m";  // 重置颜色
    public static final String YELLOW = "\u001B[33m"; // 黄色
    public static final String BLUE = "\u001B[34m";  // 蓝色

    public static void main(String[] args) throws Exception {
   	...
    }
}
```

```java
// Elevator.java

if (MainClass.debug) {
    TimableOutput.println(MainClass.BLUE + "WAITING..." + MainClass.RESET);
}

```



## 评测机

由于官方的数据投喂包使用起来比较麻烦，所以自己写了一个自动化程序，能够实现快速的输入数据，再加上对输出的逻辑判断和性能指标计算即可构成一个简易评测机。

hw7评测机源码：https://github.com/Accepted0424/quickHacker/blob/master/utils/quickInput_elevator3.py

- 支持windows和mac平台。
- 给定java项目文件夹，每次运行时会遍历文件夹下所有`.java`文件并自动引入官方依赖包生成`.jar`文件。操作简便，编辑完 java文件之后只需要直接`Run Code`即可，无需额外操作。
- **可能**比较完善的输出逻辑判断。
- 支持输出三项性能指标。
- 重要信息特殊颜色突出（SCHE、UPDATE、RECEIVE）。

- 将输入输出保存至log，便于查找。



## 心得体会

### 线程安全

在接触多线程编程之前，我一直以为“让多个线程一起工作”只是提升程序效率的一种手段。然而，真正深入学习之后才发现：线程安全是多线程编程中最难也最核心的问题之一。它不像语法错误那样容易察觉，往往潜伏在系统中，直到某一次并发冲突才暴露出严重的后果。线程安全并不是“写代码时加几把锁”那么简单，而是一种思维方式，它需要我们从程序整体考虑，而不是局限于单一线程。写好多线程程序需要大量的经验积累！

### 层次化设计

在这个单元中，我并没有很好的关注”层次化设计“，大量的逻辑写在了Elevator类中，各种逻辑的耦合导致debug的时候十分痛苦，往往是牵一发而动全身，改好了一个bug可能导致了更多的bug。这也是我需要反思学习、积累经验的，很多时候一个优秀的架构设计往往能避免很多问题。

### 其他

- 多线程编程需要摒弃之前的编程思维，更多的考虑时序关系。单线程程序的执行是线性的，开发者只需要关注代码从上到下的执行逻辑即可；但在多线程环境中，代码的执行顺序不再可控，线程之间可能并发运行、抢占资源，造成不可预测的结果。
- 很多时候本地无法跑出官方评测机的结果，甚至在本地跑的结果也千奇百怪，这主要是因为我们只能让线程进入**就绪状态**，而无法控制线程何时执行，这就带来了很多不确定性。最理想的情况当然是确保逻辑严密，JVM无论怎么调度都能保证正确性，但是这需要大量的经验累积和对底层知识的深入理解。
- 在第三次作业的处理中引入了异步操作导致不确定性进一步增加，而且也有点不符合实际，个人认为是一次比较失败的设计，但是由于时间原因确实没精力再重构了。
- 在刚学习Java的多线程编程时很多地方都感到困惑，不理解其背后的原理，但是在OS也学到线程和进程后，对很多东西都感到豁然开朗。
- 原本以为手机上的软件开发没什么难度，但是自己实操了之后才明白，写一个软件不难，但是想写一个支持高并发，能够让几亿人同时使用的软件确实绝非易事，毕竟自己处理6个电梯线程就出现了数不清的bug。