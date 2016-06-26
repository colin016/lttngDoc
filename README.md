# lttngDoc

##lttng基础概念
```
colin@colin-virtual-machine:~/lttng/lttng-ust-master/doc/examples/demo$ lttng list session1
Tracing session session1: [inactive]
    Trace path: /home/colin/lttng-traces/session1-20160626-101443

    Domain: UST global ===

Buffer type: per UID

Channels:
-------------
- channel0: [enabled]

    Attributes:
      overwrite mode: 0
      subbufers size: 131072
      number of subbufers: 4
      switch timer interval: 0
      read timer interval: 0
      trace file count: 0
      trace file size (bytes): 0
      discarded events: 0
      lost packets: 0
      output: mmap()

    Events:
      * (type: tracepoint) [enabled]
```
如上所列，Session和浏览器的Session类似，用来保存一些信息的，用lttng之前首先要创建一个Session，例如：lttng -n create session		
创建了Session后，需要做一些基础配置，例如：lttng enable-event -u -a		
然后，用lttng list <sessionName> 就能看到上面的一些信息了。每个属性解析如下:		 
归属关系：Session <- Domain <- Channel <- Event		

```
Tracing session session1: [inactive] //Session状态 active/inactive
    Trace path: /home/colin/lttng-traces/session1-20160626-101443  //Log内容保存位置

=== Domain: UST global === //Domain:Kernel/UST/Java/Log4J/Python

Buffer type: per UID //Buffer类型：per UID (同一用户下共享一个buffer), per PID（每个进程独立一个buffer）

Channels: //Channel信息，允许有多个Channel
-------------
- channel0: [enabled] //每个Channel可以单独enable/disable

    Attributes:
      overwrite mode: 0 //当Buffer满的时候，对新的Log是采用Overwrite/Discard
      subbufers size: 131072 //一个Buffer会被等分成若干个sub-buffer，当Overwrite的时候，只会檫除最旧的一个sub-buffer
      number of subbufers: 4 //sub-buffer数量
      switch timer interval: 0 // 定时自动切换sub-buffuer: 时间到后，自动将下一个sub-buffer来接收新的log（即使旧sub-buffer未满）
      read timer interval: 0
      trace file count: 0
      trace file size (bytes): 0
      discarded events: 0
      lost packets: 0
      output: mmap()

    Events: //在代码中体现为tracepoint
      * (type: tracepoint) [enabled]
```
====================lttng相关库和应用=======================
LTTng-tools:
session daemon (lttng-sessiond) //按照Session的配置，对Session管理，创建Socket去接收来自liblttng-ctl的控制信息
consumer daemon (lttng-consumerd) //真正负责保存log到lttng目录
relay daemon (lttng-relayd)
tracing control library (liblttng-ctl) //通过Socket给lttng-sessiond对Session进行配置
tracing control command line tool (lttng) //lttng-ctl的命令行实现

LTTng-UST:
user space tracing library (liblttng-ust) and its headers
preloadable user space tracing helpers (liblttng-ust-libc-wrapper, liblttng-ust-pthread-wrapper, liblttng-ust-cyg-profile, liblttng-ust-cyg-profile-fast and liblttng-ust-dl)
user space tracepoint code generator command line tool (lttng-gen-tp)
java.util.logging/log4j tracepoint providers (liblttng-ust-jul-jni and liblttng-ust-log4j-jni) and JAR file (liblttng-ust-agent.jar)
//lttng-gen-tp：定义一个tracepoint provider的.tp文件后，用该工具自动产生响应的C和H文件
//每个TRACEPOINT_PROVIDER都需要写一个tracepoint provider的C和H文件，在H文件里面的定义是TRACEPOINT_EVENT

LTTng-modules:
LTTng Linux kernel tracer module
tracing ring buffer kernel modules
many LTTng probe kernel modules
