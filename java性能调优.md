# note_java-performance tunning
java性能检测，死锁，线程阻塞，内存，cpu过高排查

主要有以下几个步骤：(以端口9001为例，以下命令都在mac上演示)
## 1.查看端口对应的进程
输入命令：
```
lsof -i tcp:9001
```
结果：
```
COMMAND  PID      USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    7388 zhengjian  154u  IPv6 0x6710d3c527b09019      0t0  TCP *:etlservicemgr (LISTEN)
```

## 2. 拿到当前堆栈快照
```
jstack -l 7388 >>dump.out
```
堆栈信息打印如下：
```
"[ThreadPool Manager] - Idle Thread" daemon prio=6 tid=0x069a3400 nid=0x84 in Object.wait() [0x0795f000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x239102b8> (a org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor)
	at java.lang.Object.wait(Object.java:503)
	at org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor.run(Executor.java:106)
	- locked <0x239102b8> (a org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor)

   Locked ownable synchronizers:
	- None

```
dump.out内容如下：
```
2019-05-17 11:34:03
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.40-b25 mixed mode):

"AsyncResolver-bootstrap-executor-0" #66 daemon prio=5 os_prio=31 tid=0x00007f9ac8b84000 nid=0x460b waiting on condition [0x000070000e4b2000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000007a0725448> (a java.util.concurrent.SynchronousQueue$TransferStack)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:458)
	at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
	at java.util.concurrent.SynchronousQueue.take(SynchronousQueue.java:924)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
	- None
```
上面文字翻译：
```
1.AsyncResolver-bootstrap-executor-0 :线程名
2.prio优先级
3.tid ，nid 为16进制，目前不知道是什么（mac不能找到对应的进程）
4.java.lang.Thread.State: WAITING线程等待状态
5.LockSupport.java:175   ：栈顶
5.Locked ownable synchronizers:
	- None
  说明不在同步代码快中   （注：此处没有验证，不知是否正确）

```

## 3.获取cpu,内存占用等情况·
```
//top 帮助命令，单独输入top，显示所有进程情况
top -h  

```
top 查看指定进程
```
top -pid 7388
```
结果如下：
```
Processes: 326 total, 2 running, 324 sleeping, 1657 threads                                                                                                                 14:53:22
Load Avg: 1.70, 2.01, 2.09  CPU usage: 8.91% user, 6.74% sys, 84.33% idle    SharedLibs: 170M resident, 50M data, 21M linkedit.
MemRegions: 65343 total, 2456M resident, 54M private, 709M shared. PhysMem: 8082M used (1872M wired), 109M unused.
VM: 1463G vsize, 1098M framework vsize, 25147559(0) swapins, 26053071(0) swapouts. Networks: packets: 17416121/17G in, 16281327/13G out.
Disks: 8966763/230G read, 6699987/219G written.

PID   COMMAND      %CPU TIME     #TH  #WQ  #POR MEM   PURG CMPRS PGRP  PPID  STATE    BOOSTS     %CPU_ME %CPU_OTHRS UID  FAULTS  COW  MSGS MSGR SYSBSD  SYSMA CSW     PAGE IDLEW 
7388  java         0.1  00:17.37 47   1    131  114M  0B   226M  78560 78560 sleeping *0[1]      0.00000 0.00000    501  151962  384  281  105  398246+ 1969  213563+ 38   69115+ 
```
