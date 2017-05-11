```
package com.example.zhanghaohao089.mytest;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 实现一个消费队列,有同步和异步方法
 *
 * @author ZHANGHAOHAO
 * @date 2017/5/11
 */

public class ProduceQueue<T> {
    private T[] items = null;
    private int head;//头部
    private int last;//尾部
    private Lock lock = new ReentrantLock();
    private Condition notFull = lock.newCondition();
    private Condition notEmpty = lock.newCondition();
    private int count;//目前的count
    private int max;//最大内容

    public ProduceQueue() {
        this(10);
    }

    public ProduceQueue(int max) {
        items = (T[])new Object[max];
        this.max = max;
    }

    //堵塞的
    public void put(T item) {
        lock.lock();
        try {
            if (count == max) {
                notFull.await();
            }
            enqueue(item);
            notEmpty.signalAll();
            count++;
        }catch (Exception e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    //非堵塞的
    public boolean offer(T item) {
        lock.lock();
        try {
            if (count == max) {
                return false;
            }
            enqueue(item);
            notEmpty.signalAll();
            count++;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return true;
    }

    //只堵塞一定时间
    public boolean offer(T item, long timeout, TimeUnit unit) {
        lock.lock();
        long nanos = unit.toNanos(timeout);
        try {
            if (count == max) {
                if (nanos <= 0)
                    return false;
                notFull.awaitNanos(nanos);
            }
            enqueue(item);
            notEmpty.signalAll();
            count++;
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
        return true;
    }

    //非堵塞的
    public T poll() {
        T item = null;
        lock.lock();
        try {
          if (count == 0) {
                return null;
            }
            item = dequeue();
            notFull.signalAll();
            count --;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return item;
    }

    //非堵塞的
    public T poll(long timeout, TimeUnit unit) {
        T item = null;
        lock.lock();
        long nanos = unit.toNanos(timeout);
        try {
            if (count == 0) {
                if (nanos <= 0)
                    return null;
                notEmpty.awaitNanos(nanos);
            }
            item = dequeue();
            notFull.signalAll();
            count --;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return item;
    }

    //堵塞的
    public T take() {
        T item = null;
        lock.lock();
        try {
            if (count == 0) {
                notEmpty.await();
            }
            item = dequeue();
            notFull.signalAll();
            count --;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return item;
    }

    //入队列操作
    private void enqueue(T item) {
        if (last == max) {
            last = 0;
        }
        items[last++] = item;
    }

    //出队列操作
    private T dequeue() {
        T item = items[head];
        items[head] = null;
        if (++head == max) {
            head = 0;
        }
        return item;
    }

    public int getSize() {
        return count;
    }
}


```
