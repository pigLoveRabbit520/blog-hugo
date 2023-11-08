---
title: go并发之WaitGroup使用
author: Salamander
tags:
  - Go
categories:
  - Go
date: 2020-06-15 20:00:00
---
## 需求
有时候我们会开启很多线程（go中是协程）去做一件事件，然后希望主线程等待这些线程都完成后才结束，一个简单的想法是，我在主线程sleep一段时间，譬如3s钟，但是明显这样的做法不科学，因为这些任务很有可能在200ms内就都完成了。如果你用过Java的话，那你很快就会想到`CountDownLatch`类，在Go中，也有类似的结构，就是本文要讨论的`WaitGroup`。

<!-- more -->


## 使用
示例代码
```
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	learnWaitGroup()
}

func learnWaitGroup() {
	num := 10
	wg := sync.WaitGroup{}
	wg.Add(num)

	for i := 0; i < num; i++ {
		go func(idx int) {
			fmt.Printf("%d Doing something...\n", idx)
			time.Sleep(time.Second)
			wg.Done()
		}(i)
	}

	wg.Wait()
	fmt.Println("All is done...")
}
```
`WaitGroup` 对象内部有一个计数器，最初从0开始，它有三个方法：`Add()`, `Done()`, `Wait()` 用来控制计数器的数量。`Add(n)` 把计数器设置为n ，`Done()` 每次把计数器-1 ，`Wait()` 会阻塞代码的运行，直到计数器地值减为0。  


## 注意问题
`WaitGroup`对象不是一个引用类型，所以在作为参数的时候，你应该要使用指针。在上面的示例提取一个任务函数
```
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	learnWaitGroup()
}

func learnWaitGroup() {
	num := 10
	wg := sync.WaitGroup{}
	wg.Add(num)

	for i := 0; i < num; i++ {
		go runTask(i, &wg)
	}

	wg.Wait()
	fmt.Println("All is done...")
}

func runTask(idx int, wg *sync.WaitGroup) {
	fmt.Printf("%d Doing something...\n", idx)
	time.Sleep(time.Second)
	wg.Done()
}
```

## Java类比
Java中可以使用`CountDownLatch`类实现这个功能，它暴露出三个方法：
```
// 调用此方法的线程会被阻塞，直到 CountDownLatch 的 count 为 0
public void await() throws InterruptedException

// 会将 count 减 1，直至为 0
public void countDown() 
```
`countDown()` 跟`WaitGroup`的`Done()`函数类似，我们还是很容易实现的
```
public class Main {
    public static void main(String[] args) {
        try {
            testCountDownLatch();
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
    
    static class TaskThread extends Thread {
        
        CountDownLatch latch;
        
        public TaskThread(CountDownLatch latch) {
            this.latch = latch;
        }
        
        @Override
        public void run() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.println(getName() + " Task is Done");
                latch.countDown();
            }
        }
    }

    public static void testCountDownLatch() throws InterruptedException {
        final int threadNum = 10;
        CountDownLatch latch = new CountDownLatch(threadNum);
        for(int i = 0; i < threadNum; i++) {
            TaskThread task = new TaskThread(latch);
            task.start();
        }
        
        System.out.println("Task Start!");
        latch.await();
        System.out.println("All Task is Done!");
    }
}
```