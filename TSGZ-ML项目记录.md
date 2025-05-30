# TSGZ-ML项目

## 1.命令记录

1.打流命令

```
cd /home/dptech/mutl_pcap
./replay.sh

tcpreplay -M 10 -i eno2
```

2.关闭打流命令

```
 killall tcpreplay
```

3.查看rpm包的信息

```
rpm --info -q tsgz-ml
```

4.安装rpm包

```
rpm -Uvh --force tsgz-ml-*.rpm
```

5.调试coredump

```
（1）coredumpctl list 查看core dump 文件
（2）coredumpctl debug <pid>调试进程
（3）bt查看进程调用堆栈
```

6.查看rpm包的内容

```
rpm -ql dptech_sqli_sigs
```

## 2.原理图

![](D:\sql注入项目\项目图\TSGZ-ML.png)

1.关于udp和tcp读写数据加锁：

- 对于 UDP，多线程读写同一个 socket 不用加锁，不过更好的做法是每个线程有自己的 socket，避免 竞争，可以用SO_REUSEPORT来实现这一点。
- 对于 TCP，通常多线程读写同一个 socket 是错误的设计，因为有 [short write](https://zhida.zhihu.com/search?content_id=55908446&content_type=Answer&match_order=1&q=short+write&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDg3NjIyMzAsInEiOiJzaG9ydCB3cml0ZSIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjU1OTA4NDQ2LCJjb250ZW50X3R5cGUiOiJBbnN3ZXIiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.KlOXJo_cnRoBVDGH8mMCl0P9t-W5sBXGDdHEhunWeKg&zhida_source=entity) 的可能。假如你加锁，而又发生 short write，你是不是要一直等到整条消息发送完才解锁（无论阻塞IO还是非阻塞IO）？如果这样，临界区长度由客户端什么时候接收数据来决定，一个慢的客户端 就把程序搞死了。

总结：对于 UDP，加锁是多余的；对于 TCP，加锁是错误的。

2.无论是tcp还是udp，多线程读写socket是否线程安全？

- 无论是tcp还是udp，多线程读写socket都是线程安全的。

- 但是tcp是流式套接字，虽然读写是线程安全的，但是如果不加控制，多线程同时读写tcp套接字会造成数据的错乱，而udp是数据报套接字，每一个数据都是一个完整的数据报，因此，针对tcp套接字，通常来说由一个线程负责tcp套接字的写，别的线程只是负责将处理好的数据放到一个队列中，负责写的线程从这个队列中取数据然后发送，同样针对读操作也是类似的。

- 针对数据报套接字，虽然说数据是完整的一个数据报，但是多个线程同时读写一个套接字会存在竞态，可能导致性能瓶颈（例如锁竞争）或者操作系统调度开销，因此通常的做法是将套接字设置SO_REUSEPORT，允许多个套接字绑定到同一个端口上，这样，即使不同的线程或进程使用不同的套接字，只要这些套接字绑定到了相同的端口，操作系统会负责将传入的 UDP 数据包分配给这些套接字，而每个线程可以独立地处理其绑定的套接字，避免了线程间的竞争和锁的开销。

  补充：此处涉及的负载均衡主要是四层负载均衡：

  操作系统在接收到数据包时，会根据一定的负载均衡策略将数据包均匀地分配给每个套接字。这种分配策略通常基于以下几种方法：

  - **轮询**：操作系统会按照轮询的方式将数据包分发给每个已绑定套接字。每次接收到一个数据包时，系统会将其交给下一个套接字。
  - **基于负载的策略**：操作系统可能根据每个套接字的负载（如当前线程的处理能力）来决定将数据包分配给哪个套接字。这样可以平衡各线程的负载，避免某个线程过载。
  - **哈希算法**：某些操作系统可能使用哈希算法，基于数据包的一些特征（如源 IP 地址或端口号）来决定哪个套接字接收该数据包。

