### Houserush抢房项目Jmeter压测



脚本使用：

- 引入fastjson库
- 设置ip和端口gateway_ip,gateway_port
- 设置秒杀开始时间startTime
- 设置线程数threadNum和启动时间rampUpTime
- 设置读取用户登陆账号和密码文件路径
- 

服务器配置： 16G服务器，单核8线程

服务实例配置：

- 每个微服务都只开一个实例

模拟场景流程：

​	1.设置一个n个线程（用户数）,m秒完全启动。（m稍微大点，保证每个用户登录成功。真实的场景也是如此，用户并不是同时登陆）

​	2.查询一次所有的开售活动、查询一次当前要抢的开售活动

​	3.查询一次当前要抢的开售活动，随机获取一个房子订单id

​	4.设置时刻定时器，到达指定时刻，即开始第5步下单，保证所以用户同时开抢

​	5.抢购第三步中随机获取的房子，如果抢购成功则当前用户完成，否则到6

​	6.再次下单该房子，如果再次不成功，则重新查询开售活动，获取一个还没有被抢的房子订单继续下单，最多循环4次。

​	7.结束

##### 1. zuul gateway

使用默认配置，不做额外配置。每个微服务都只开一个实例

结果：

- 30并发量，抢购成功为25，26
- 40并发量，抢购成功0（大量转发失败信息）

综上，当前设置下zuul的并发量大概30左右（没进行调优配置）



##### 2.edge-service 

使用默认配置，不做额外配置。每个微服务都只开一个实例

- 线程数40，抢购成功40
- 线程数100，抢购成功100
- 线程数150，抢购成功137，数据库开始拒绝连接（mysql5.6默认最大连接数151）

```shell
show variables like 'max_connection' #查看当前最大连接数
set global max_connections = 10000     #修改最大连接数为10000
```

- 线程数200，抢购成功199

设置mysql max_connection=10000

- 线程数800，抢购成功498

- 线程数1000，抢购成功498

  ![聚合报告](C:\Users\linzibin\Desktop\jmeter\result_img\1000_1.PNG)

  ![图表](C:\Users\linzibin\Desktop\jmeter\result_img\1000_2.PNG)

可知，除了响应时间接近20多秒接近30，无异常情况，服务器能完美抗下1000的并发。

- 线程数1500，抢购成功377，大部分请求超时挂了（30秒超时了）

  - jmeter出现java.lang.OutOfMemoryError: Java heap space，而jmeter的heap默认配置为-Xms1g -Xmx1g -XX:MaxMetaspaceSize=256m

  把jmeter的heap设置为-Xms4g -Xmx4g -XX:MaxMetaspaceSize=1024m

- 线程数1500，抢购成功数498

  ![1500聚合报告](C:\Users\linzibin\Desktop\jmeter\result_img\1500_1.PNG)

  ![1500响应时间](C:\Users\linzibin\Desktop\jmeter\result_img\1500_2.PNG)

  ![TPS](C:\Users\linzibin\Desktop\jmeter\result_img\1500_3.PNG)

  TPS接近1200
  
  抢购瞬间，CPU满负载，响应时间有接近30s,几乎到了极限了。
  
- 线程数2000
  
  ![聚合报告](C:\Users\linzibin\Desktop\jmeter\result_img\2000_1.PNG)
  
  ![2000_2](C:\Users\linzibin\Desktop\jmeter\result_img\2000_2.PNG)
  
  ![2000_3](C:\Users\linzibin\Desktop\jmeter\result_img\2000_3.PNG)
  
  TPS反而降低了
  
  **综上可知，在当前机器配置（16G,单核心8线程）下，一个实例能正常处理的并发请求大概为1500左右**
  
  性能瓶颈：
  
  ​	主要是因为超时而导致异常（30s超时）
  
  后续：
  
  ​	分析分布式调用链追踪可以知，超时调用主要在edge-service start 和 house-order stat之间花费了29s左右。找出导致超时的原因，进而优化。
  
  
