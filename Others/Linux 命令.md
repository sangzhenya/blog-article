---
title: "Linux 命令"
tags: ["Linux"]
categories: ["其他"]
date: "2019-04-21T09:00:00+08:00"
---

强制卸载软件：

```shell
sudo mv /var/lib/dpkg/info/{packagename}.* /tmp/
sudo dpkg --remove --force-remove-reinstreq {packagename}
sudo apt-get remove {packagename}
sudo apt-get autoremove && sudo apt-get autoclean
```



常用命令：

`top` 命令：

![top 命令](https://i.loli.net/2019/06/25/5d120a6fe694016796.png)

load average: 三个值分别是系统 1分钟，5分钟，15分钟系统的负载平均值。如果三个值相加，除以3，乘以 100%的结果大于 60%，则表明系统压力过大。

---

`uptime` 命令精简版，显示系统负载情况。

![uptime 命令](https://i.loli.net/2019/06/25/5d120b3fd6d7715038.png)

---

`vmstat` 命令； `vmstat -n 2 3` 每 2 秒查询一次，一共查询 3 次。

![vmsta 命令](https://i.loli.net/2019/06/25/5d120bbb1295d91056.png)

+ procs

  r： 运行和等待 CPU 时间片的进程数，原则上 1核的 CPU 的运行队列不要超过 2，整个系统的运行队列不要超过总核数的 2倍。

  b: 等待资源的进程数，比如正在等待磁盘 IO。网络 IO 等

+ cpu

  us: 用户进程消耗 CPU的时间比，us  值高，用户进程消耗 CPU 时间多，如果长期大于 50%，则需要优化程序。

  sy: 内核内核进程消耗 CPU 时间百分比。

  us + sy 参考值为 80%，如果 大于 80%，则说明存在 CPU 不足。

  id: 处于空闲的 CPU 百分比。

  wa: 系统等待 IO 的CPU 时间百分比。

  st: 来自于一个虚拟机偷取的 CPU 时间的百分比。

---

`mpstat -P ALL 2` 每两秒打印一次 CPU 使用情况。

---

`pidstat -u 1 -p 11581` 查看某个进程的 CPU 使用情况，每秒钟采样一次。

---

`free`: 查看内存使用情况, 应用程序可用内存/系统物理内存 > 70% 内存充足。如果 < 20%，需要加内存。20% ~ 70% 之间，表示基本够用。

`free -g`, `free -m`  单位分别是 按照 g 和 m 来查看。

---

`pidstat -p 进程号 -r 采样间隔秒数` 查看某个应用的内存占用。

---

`df -h` 查看系统磁盘剩余空间。

---

`iostat -xdk 2 3` 查看网络访问情况，每2秒采集一次，一共采集3次。

+ rKB/s 每秒读取数据量 kB
+ wKB/s 每秒写入数据量 kB
+ svctm I/O 请求平均等待服务时间，单位是毫秒。
+ await I/O 请求的平均等待时间，单位是毫秒。值越小性能越好。
+ util 一秒中有百分之几的时间用于 I/O。接近 100% 时，表示磁盘宽带跑满，需要优化程序或者增加磁盘。
+ rKB/s、wKB/s 根据系统应用不同会用不同的值，但有规律遵循：长期、超大数据的读写，肯定不正常。需要优化程序读取。
+ svctm 的值与 await的值很接近表示几乎没有 IO 等待，磁盘性能好。如果 await的值远高于 svctm 的值，则表示 IO 队列等待太长，需要优化程序或更换更快磁盘。

---

`pidstat -d 采样间隔秒数 -p 进程号`  查看磁盘使用情况。

---

`ifstat 1`  每秒采样一次，查看网络使用情况。

---

`ps -mp 5101 -o THREAD,tid,time`  查看具体线程的占用情况。

+ m 显示所有线程
+ p pid 进程使用 CPU 的时间
+ o 参数后是用户自定义格式

将需要的线程 ID 转换为 16 进制格式，英文小写格式

---

`jstack 进程ID | grep tid(英文小写格式)  -A60`  打印线程使用信息的前 60行。

---

```bash
ps aux | grep java # 查看某个线程
netstat -ap | grep 8004 # 查看端口占用情况
```