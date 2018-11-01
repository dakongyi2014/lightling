## codahale-metrics

Metric的中心部件是`MetricRegistry`。 它是程序中所有度量metric的容器

Reporter类是将metrics按照一定的时间间隔输出 

```
ConsoleReporter reporter = ConsoleReporter.forRegistry(registry)
      .convertRatesTo(TimeUnit.SECONDS) //输出的rate的时间单位  rateUnit
      .convertDurationsTo(TimeUnit.MILLISECONDS)//设置duration的单位 durationUnit
      //Timer才会使用到该单位
        //调用代码示例子：
       // output.printf(locale, "               min = %2.2f %s%n", convertDuration(snapshot.getMin()), getDurationUnit());
      .build();
reporter.start(1, TimeUnit.SECONDS);//从启动后的1s后开始（所以通常第一个计数都是不准的，从第二个开始会越来越准），每隔一秒从MetricRegistry钟poll一次数据
System.out.println("执行与业务逻辑");
         
while(true){
            meterTps.mark();//总数以及m1,m5,m15的数据都+1
            Thread.sleep(500);
}
```

```
count = 14
         mean rate = 2.00 events/second//该second与convertRatesTo(TimeUnit.SECONDS)的单位相同
     1-minute rate = 2.00 events/second
     5-minute rate = 2.00 events/second
    15-minute rate = 2.00 events/second
```

   report.start()方法解释：

1. ​          在服务启动的initialDelay unit（这里就是1s）后开始每隔period unit执行一次command（所以，通常第一次统计都不准确，从第二次开始变得准确）
2. ​          reporter值主动从MetricRegistry中poll数据的
3. ​          真正的report是被synchronized块包起来的（也就是线程安全的），而report的内部逻辑随着report的类型不同而不同（例如，ConsoleReporter就是将四种数据打印到console） 

####  meter类metrics

​        作用：统计最近1分钟(m1)，5分钟(m5)，15分钟(m15)，还有全部时间的速率（速率就是平均值）

​         例如：qps

​         线程安全：mark()方法中的四个操作都是基于CAS实现，统计线程安全。

​        注意：

- MetricRegistry是一个所有metrics的容器（通常设为单例）
- ConsoleReporter根据指定的打印速率（在start方法中指定）将metrics打印到console
- metrics name需要指定，这对于在statsd的统计部分以及聚合函数的选择都有用，上边的name()方法实际上是将类的全类名与后续的不定参数以"."拼接而成，这里metric name就是"com.xxx.secondboot.metrics.TestMeter.request.tps"
- mark方法：总数count和最近1分钟(m1)，5分钟(m5)，15分钟(m15)的数据都+1



####    gauge类metrics

​          作用：返回一个瞬时值（就是一个具体值）

​          例如：某一时刻的队列size

​          线程安全：只是做读操作，线程安全

​        注意：

-    在registry()的时候，可以直接将一个类型的Metric直接注入到容器中，其name就是registry()的第一个参数

####      counter类metrics

​              作用：gauge的AtomicLong实例（Counter 只是用 Gauge 封装了 AtomicLong），可用于加（inc()）减（dec()）

​              例如：获得队列长度（此处的获取要比使用gauge通过size()方法获取高效很多，后者size()方法的获取大多数是O(n)），方法执行成功失败次数（这个就是gauge无法做的）

​              作用：AtomicLong基于CAS，线程安全

####      histogram类metrics（使用较少）

​            作用：计算执行次数count、最小值min，最大值max，平均值mean，方差stddev，中位数median，75              百分位, 90百分位, 95百分位, 98百分位, 99百分位, 和 99.9百分位的值 

​             例如：统计某个函数的执行耗时，以上这些值通常会是执行时间，如min是最短执行时间等

​             线程：update的操作需要获取锁，操作之后释放锁。线程安全。

​    timer类metrics

​           作用：meter和histogram的组合体

​           例如：统计某个函数的qps和执行耗时。

​           线程安全：meter和histogram都安全，所以也线程安全

#### 总结：

-  统计某个函数被调用的频率（TPS），使用Meters。
- 统计某个方法的耗时，使用Histograms。--注意时间是以纳秒为单位的
- 既要统计某个方法的TPS又要统计其耗时时，使用Timers。--注意时间是以纳秒为单位的
- counter用于计数
- gauge只用于记录瞬时值

###  counter与gauge：

- 在某些时候，只能用gauge，比如说这个值是在第三方包提供的，例如guava cache的cache size（而恰好我们将该cache集成在spring cache中，通过注解来使用了），无法用哪个counter来测量
- 在某些时候，只能用counter，比如说一个方法的执行成功与失败次数

histogram：

在统计中位数以及95%这样的数据的时候，通常需要把所有的数据拿出来，然后进行运算（在大量的数据下该方法失效，所以采用了水库采集法--reservoir sampling，通过维护一个小的、可管理的水库来代表全部统计数据），具体采集法有以下几种：

- Uniform Reservoirs：随机选择具有线性递减概率的储层的值，仅用于长时间的测量。测量统计数据最近是不是发生了变化，不要使用这个（使用下边的指数衰减水库）。

- Exponentially Decaying Reservoirs（指数衰减水库）：该水库采集的数据可以代表大约最后5分钟的全部数据。该水库也是Times 类metrics使用histogram的默认选择水库。

- Sliding Window Reservoirs：代表过去n次测量的数据

- Sliding Time Window Reservoirs：严格的代表过去n秒内的数据（注意与指数衰减库的区别，该方法严格的记录过去的每一秒的数据（而指数衰减其实还是在最后5min进行抽样），所以在高频下可能需要更多内存，而且也是最慢的水库类型）

  参考文献：

  ​     http://wuchong.me/blog/2015/08/01/getting-started-with-metrics/

  ​    https://metrics.dropwizard.io/3.1.0/getting-started/