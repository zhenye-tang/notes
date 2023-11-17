## wait queue

* 等待队列
* 某个线程可以添加到等待队列中，rt_wqueue_wakeup 时，会唤该队列上挂在的线程
* 比如 poll 的实现原理
* 将想要监控的文件描述符，添加到 pollfd 中，调用 poll 时，如果该设备添加了 poll 接口，此时会调用到设备的 poll 接口中，在设备的 poll 接口中，会将当前线程添加到设备层创建的等待队列中
* 当某一设备满足某一条件时，设备会主动调用 rt_wqueue_wakeup，此时就会唤醒该设备的等待队列上的所有线程
* poll 的实现就利用了等待队列的机制，因此，设备层需要存在一个等待队列，并且设备层提供了 poll 接口供 poll 函数调用。通过这一机制巧妙的实现了一个线程，监控多个 IO设备
* 问题：如果一个 poll 监控了 3 个设备，一个设备唤醒（调用 rt_wqueue_wakeup）了，此时调用 poll 的线程被唤醒，遍历 pollfd 查看到底是那个设备，途中，poll 监控的第二个设备也调用了 rt_wqueue_wakeup，此时会咋样呢？？

## work queue

## ring buffer

## pipe

## poll

## completion

## ring block buffer

