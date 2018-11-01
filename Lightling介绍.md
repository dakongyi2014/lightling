Lightling

​       lightling是我构想的一个实时计算平台，目的是简化实时计算的开发难度，为实时计算job的运行提供必要的监控和预警。为实时job是7*24小时能力保驾护航。

​       目前进展，

​          一、开发了基于spark-streaming的工具包，主要包含对

​           1、offset的管理

​           2、各种db的sink

​           3、业务metric的封装

​      该工具包的后续将引入flink。

​        二、调优和监控与预警功能

​           调优诊断：

​               现状：目前针对spark-streaming的调优主要依赖sparkUI提供的界面。但是SparkUI只会展示当前batch的job运行状态。无历史数据，缺乏对比。

​               后续计划：我们将开发sdk将spark产生的Metic数据采集。并集成都我们的linghtling平台中，提供更加全面的调优诊断大盘。

​            预警功能：

​                  spark在并未实现预警功能，该功能需要用户自定义。目前我们的的预警方案，是通过shell对spark-streaming job的运行状态进行监控并提供相关的预警。

​                 后续计划：根据采集到的job运行数据监控（如timeline,stage运行时间，job运行时间，每轮batch数据输入的速度，数据量的大小，每轮batchSink的时间，task失败次数，executor失败次数等），以及资源分配数据（如每个application的container在各个机器分布的情况，每个job申请的executor的数据，申请的内存，cup core等资源的监控）再配合运维提供的机器层面的监控metic。制定符合我们平台的各类预警指标。并进行预警。真正实现实时计算的稳，准，快的要求。

​        相关架构：

​                ![](C:\Users\admin\Desktop\lightling架构图.png)

​               ![](C:\Users\admin\Desktop\monitor.png)

​                           

​             