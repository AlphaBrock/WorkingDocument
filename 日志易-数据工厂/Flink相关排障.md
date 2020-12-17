## 划重点

1. Flink作为流式计算，其对于数据处理的核心在于`时间`，所以出问题80%可能是跟数据时间有关
2. 使flink计算结果更接近于spl同于，建议配置任务时:时间(`Time Characteristic`)选择 Event Time

## 故障分类

### 低级问题

1. fornaxee数据流: kafkaSource-->Filter，kafkaSource消费有数据，但是经过Filter后没数据了

   Filter的算子作用就是过滤数据，经过Filter算子后没数据了，只能有俩种情况

   - Filter算子表达式未匹配到相关数据，即kakfa消费出来的数据没有需要的数据
   - Filter算子表达式写的有问题

2. kafkaSource消费不出数据

   要查这个问题，首先要理解，对于kafkaSource来说，其数据源是从哪里来的？

   kafkaSource的作用就是从某个kafka环境中的某个Topic数据消费出来，在实施过程，一般消费的是我们平台自身的Topic，数据源明确了，那就需要知道数据流向了

   ```mermaid
   graph LR
       日志源 --> agent --> collector --> kafka --> logriver --> kafka
   ```

   从上面我们就可以知道整个数据流了，在这个过程需要切记的一件事，kafkaSource消费的数据是logriver解析后再回写到kafka的，所以出问题也就大概俩种问题

   - 路由回写规则问题，导致数据未正确回写入kafka

   - 采集端到进入kafka这个流程除了问题，这个时候请按照该流程图排查，具体说明文档参考 [期望的日志未能搜到 排障](https://tower.im/projects/2a4652353fc84996ad72d0556c7464c8/docs/7c059b499b024f3a83cb38b72e104952/)

     ![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201216145024.png)

### 中级问题

1. 数据源没问题，配置也没问题，窗口计算中也并未出现丢数据，但是输出结果和SPL统计结果就是不一致

   ![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201214213056.png)

   Flink中的时间有三类

   - Event Time 

     事件事件，在我们平台中，一种是指日志产生的时间，即kafka中的timestamp字段

   -  Ingestion Time

     摄入时间，事件进入Flink的时间

   - Processing Time

     处理时间，根据处理机器的系统时钟决定数据流当前的时间

   我们默认的Fornaxee创建的任务使用的是`Processing Time`，但是Flink中的算子计算常常是SPL转换过来的，在日志易平台中一个SPL的统计所使用的时间窗口是依赖于`timestamp`的，所以如果在Flink计算中，Watermark不是依赖于Event Time，则会导致出现该问题，所以解决方案就是修改Flink任务配置

   解决方案:

   - Time Characteristic配置改成EventTime

   ![image-20201216211745019](https://cdn.nlark.com/yuque/0/2020/png/8380787/1608124750033-8d73f00b-83d2-4ce7-9ef6-9efe9c026177.png)

2. Flink输出结果和SPL统计结果相隔一个时间点，即: SPL是12点10分的值，但是Flink计算结果却是12点11分

   Flink窗口计算的核心是: 窗口本身有个[starttime entime]，当水位线 >= entime则触发计算，输出结果中有个`end`字段存放时间戳，这个时间戳为窗口中的`endtime`

   **窗口的本质其实和spl中的bucket分桶一致**，但是SPL中bucket切出时间为`开始`时间

   换而言之，同一个值，Flink得出来的结果比SPL统计的结果相差一个窗口时间

   比如: 12点10分 -- 12点11分的数据统计，其分别为Flink和SPL中 的 【starttime endimte】，时间范围为1分钟，但是Flink计算值标记的时间戳是starttime，即12点10分，SPL计算值标记的时间戳是endimte，即12点11分

   解决方案:

   - 在Evaluator算子中，对end字段值减去一个窗口时间

### 高级问题

1. 预览有数据，但实际任务开启无数据

   这种情况一般就是watermark(水位线)问题，Flink窗口计算的核心是: 窗口本身有个[starttime entime]，当水位线 >= entime才会触发计算，如果watermark(水位线)的值是未来的，这个时候窗口时间也是未来的(**在这里任务时间的EvenTime**)，但是kafka中大量为正常事件的数据，这个时候消费出来的数据时间戳一直 < starttime，此时没有数据计入到窗口，则窗口会一直认为数据未达到，实际上很大部分数据都已经来了，但是不在窗口中，被当成迟到事件丢弃了，才导致任务一直没有数据

   解决方案:

   - 排查是否存在某些主机时钟是否有差距，即未来时间
   - 排查日志入库是否有很大的延迟，比对 agent_send_timestamp, collector_recv_timestamp，因为需要分析的数据延迟了，但是部分数据又没延迟，混杂在一起，导致水位线错乱了

2. Aggregator算子丢弃数据造成统计结果和SPL统计相差甚远

   假如我们设置10s的时间窗口（window），那么0-10s，11-20s都是一个窗口，以0~10s为例，0为start-time，10为end-time。假如有4个数据的event-time分别是8(A),12.5(B),9(C),13.5(D)，我们设置Watermarks为当前所有到达数据event-time的最大值减去延迟值3.5秒

   1. 当A到达的时候，Watermarks为`max{8}-3.5=8-3.5 = 4.5 < 10`,不会触发计算
   2. 当B到达的时候，Watermarks为`max(12.5,8)-3.5=12.5-3.5 = 9 < 10`,不会触发计算
   3. 当C到达的时候，Watermarks为`max(12.5,8,9)-3.5=12.5-3.5 = 9 < 10`,不会触发计算
   4. 当D到达的时候，Watermarks为`max(13.5,12.5,8,9)-3.5=13.5-3.5 = 10 = 10`,触发计算
   5. 触发计算的时候，会将A，C（因为他们都小于10）都计算进去，其中C是乱序迟到的。

   max这个很关键，就是当前窗口内，所有事件的最大事件。

   这里的延迟3.5s是我们假设一个数据到达的时候，比他早3.5s的数据肯定也都到达了，这个是需要根据经验推算。假设加入D到达以后又到达了一个E，event-time=6，但是由于0~10的时间窗口已经开始计算了，所以E就丢了。

   > 到了这步，大致也就俩种问题： 要么乱序程度大，要么迟到事件太多

   解决方案：

   Max-Out-Of-Orderness 稍微调大

   当出现数据丢的太多时，Aggregator算子的Lateness值调大

   **注意:  Lateness不能调的太大，否则会导致性能开销过大**

3. 在一段时间内，Flink计算结果和SPL统计结果几乎一致，但是偶尔会有几个时间段，俩者差距颇大，现象如下: 忽略Flink的峰谷，明显有俩个时间段，5点30-6点40 9点-9点30

   ![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201217094535.png)

   看到大部分时间俩条线是重合的，维度部分时间段出现了异常，那大致可以说明，算子是没问题的，**拿5点30-6点40时间段来看，这期间Flink计算值明显少了很多，但6点40后Flink计算值突然激增，结合Flink窗口计算方式(只有到了窗口内的数据才能被计算)**，我们大致可以猜测，这期间，kakfa中的数据其实是少了很多，为什么少: 

   - 出现了积压
   - Agent采集端积压
   - 发送过程中带宽不足(Agent本身可以限制发送速率)

   以上三种可能情况我们可以通过一个方式来验证，比对 agent_send_timestamp(agent扫描到日志时间), collector_recv_timestamp(集群接收到日志时间), timestamp(日志产生时间, 做了时间戳解析) ，这三个时间的差值

   ```sql
   index=yyrz appname:jzyypt tag: jzyypt_esb_service_records
   |eval source_cost=(collector_recv_timestamp-timestamp)/1000
   |eval recv_cost=(collector_recv_timestamp-agent_send_timestamp)/1000
   |eval scan_cost=(agent_send_timestamp-timestamp)/1000
   |bucket timestamp span=1m as ts
   |stats pct(source_cost,95) as a, pct(recv_cost,95) as b, pct(scan_cost,95) as c by ts
   |rename a.95 as "集群接收时间减去timestamp",b.95 as "集群接收时间减去agent发送时间",c.95 as "agent发送时间减去timestamp"
   ```

   ![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201217100138.png)

   从图中可以看出 5点30-6点40期间， collector_recv_timestamp-timestamp和collector_recv_timestamp-agent_send_timestamp的值激增，而agent_send_timestamp-timestamp的值很稳定，也就是说agent扫描读取日志后到集群接收到日志期间的延迟很大，着了刚好验证了刚刚的猜想，找到原因了，那就好对症下药了

   解决方案:

   - 检查Agent 发送带宽是不是受到了限制(Agent配置查看)
   - 在这期间是否是Agent缓存满了，即这个时间段日志激增，导致发送过程延迟
   - 检查Agent是否为性能不足
   - **其他可能性待补充。。。**