---
title: Concurrent Programming
date: 2019-07-24 23:03:42
tags:
- Java
categories: Web
---

# 1. Basic concepts

Processes run in separate memory spaces. Threads of the same process run in a shared memory spaces. They both are independent sequences of execution.

## Concurrency vs parallelism

{% asset_img concurrency-vs-parallelism.jpeg concurrency vs parallelism %}

```java
 public static void main(String[] args) {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos =
                threadMXBean.dumpAllThreads(false, false);
        for (ThreadInfo threadInfo: threadInfos) {
            System.out.println("[" + threadInfo.getThreadId()
            + "]" + " " + threadInfo.getThreadName());
        }
    }
```
<!--more-->
output:
```
[1] main
[2] Reference Handler
[3] Finalizer
[4] Signal Dispatcher
[10] Common-Cleaner
[11] Monitor Ctrl-Break
```

## Three ways to start a new thread
* extends Thread
* implement Runnable
* implement Callable

```java
    private static class UseRun implements Runnable {
        @Override
        public void run() {
            System.out.println("Use Runnable");
        }
    }

    private static class UseCall implements Callable<String> {

        @Override
        public String call() throws Exception {
            System.out.println("Use Callable");
            return "CallResult";
        }
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        UseRun useRun = new UseRun();
        new Thread(useRun).start();

        UseCall useCall = new UseCall();
        FutureTask<String> futureTask = new FutureTask<>(useCall);
        new Thread(futureTask).start();
        System.out.println(futureTask.get());
    }
```

## Stop Thread
* interrupt()
* isInterrupted()
Tests whether this thread has been interrupted.
* static interrupted(): Tests whether the current thread has been interrupted.

```java
    private static class UseThread extends Thread {
        public UseThread(String name) {
            super(name);
        }

        @Override
        public void run() {
            String threadName = Thread.currentThread().getName();
            while (!isInterrupted()) {
                System.out.println("thread name " + threadName + " is running");
            }
            System.out.println("thread name intrrupted flag is " + isInterrupted());
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread endThread = new UseThread("endThread");
        endThread.start();
        Thread.sleep(20);
        endThread.interrupt();
    }
```

## share variables between threads

* synchronized
* volatile
* ThreadLocal

## wait/notify/notifyAll

wait side:
1) obtain lock
2) check if condition is met in a loop, if not, wait
3) if yes, process

notify side:
1) obtain lock
2) update condition
3) notify all waiting threads

wait with timeout:
```java
long overtime = now + time;
long remain = time;
while (condition not met && remain > 0) {
    wait(remain);
    remain = overtime - now;
}
return result;
```

# Tools

## Fork/Join

A problem of size N. If N is smaller than n, divide N to k smaller size problem. Each subproblem is independent.

1) pool = new ForkJoinPool();
2) MyTask myTask = new ForkJoinTask(); --> 
3) pool.invoke(myTask);
4) result = myTask.join()

MyTask extends RecursiveTask/RecursiveAction/ForkJoinTask and override compute method:
if (condition is met) {
    do work;
    submit result;
} else {
    divide task to subtasks;
    invoke all subtasks;
    join all results;
}

```java
        @Override
        protected Integer compute() {
            if (toIndex - fromIndex < THRESHOLD) {
                int count = 0;
                for (int i = fromIndex; i <= toIndex; i++) {
                    count = count + src[i];
                }
                return count;
            } else {
                int mid = (fromIndex + toIndex) / 2;
                SumTask left = new SumTask(src, fromIndex, mid);
                SumTask right = new SumTask(src, mid + 1, toIndex);
                invokeAll(left, right);
                return left.join() + right.join();
            }
        }
    }
```