# lttngDoc

##lttng基础概念
```c
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
如上所列，Session是用来保存一些信息的，跟浏览器Session类似  
1.用lttng之前首先要创建一个Session，例如：lttng -n create session		
2.创建了Session后，需要做一些基础配置，例如：lttng enable-event -u -a		
3.然后，用lttng list <sessionName> 就能看到上面的一些信息了。每个属性解析如下:		 
归属关系：Session <- Domain <- Channel <- Event		

<image src="http://lttng.org/images/docs26/concepts.png"></image>
```
Tracing session session1: [inactive] //Session状态 active/inactive
    Trace path: /home/colin/lttng-traces/session1-20160626-101443  //Log内容保存位置

=== Domain: UST global === //Domain:Kernel/UST/Java/Log4J/Python

Buffer type: per UID //Buffer类型：per UID (同一用户下共享一个buffer), per PID（每个进程独立一个buffer）

Channels: //Channel信息，允许有多个Channel
-------------
- channel0: [enabled] //每个Channel可以单独enable/disable

    Attributes:
      overwrite mode: 0 //当Buffer满的时候，对新的Log是采用Overwrite(檫除最旧的sub-subffer)/Discard(丢弃最新的)
      subbufers size: 131072 //Buffer被等分成若干个sub-buffer
      number of subbufers: 4 //sub-buffer数量
      switch timer interval: 0 // 定时自动切换sub-buf:时间到后，将下一个sub-buf来接收新的log（即使旧sub-buf未满）
      read timer interval: 0
      trace file count: 0
      trace file size (bytes): 0
      discarded events: 0
      lost packets: 0
      output: mmap()

    Events: //在代码中体现为tracepoint
      * (type: tracepoint) [enabled]
```
    
<font color=red>Log内容保存位置可以是远程路径</font>   
    
##lttng相关库和应用
###LTTng-tools:
session daemon (lttng-sessiond) //按照Session的配置，对Session管理，创建Socket去接收来自liblttng-ctl的控制信息  
consumer daemon (lttng-consumerd) //真正负责保存log到lttng目录  
relay daemon (lttng-relayd) 
tracing control library (liblttng-ctl) //通过Socket给lttng-sessiond对Session进行配置    
tracing control command line tool (lttng) //lttng-ctl的命令行实现

<image src="http://lttng.org/images/docs27/plumbing-27.png"></image>

###LTTng-UST:
user space tracing library (liblttng-ust) and its headers      
preloadable user space tracing helpers: 
(liblttng-ust-libc-wrapper,liblttng-ust-pthread-wrapper,liblttng-ust-cyg-profile,   
liblttng-ust-cyg-profile-fast and liblttng-ust-dl)    
user space tracepoint code generator command line tool (lttng-gen-tp)      
java.util.logging/log4j :
tracepoint providers (liblttng-ust-jul-jni and liblttng-ust-log4j-jni) and JAR file(liblttng-ust-agent.jar) 
    
//lttng-gen-tp：定义一个tracepoint provider的.tp文件后，用该工具自动产生响应的C和H文件    
//每个TRACEPOINT_PROVIDER都需要写一个tracepoint provider的C和H文件，在H文件里面的定义是TRACEPOINT_EVENT 

###LTTng-modules:
LTTng Linux kernel tracer module    
tracing ring buffer kernel modules  
many LTTng probe kernel modules 

##在代码里面使用lttng
###定义TRACEPOINT PROVIDER
定义Provider：需要定义一个c和一个h文件    
`tp.h  `
```c
#undef TRACEPOINT_PROVIDER
#define TRACEPOINT_PROVIDER my_provider

#undef TRACEPOINT_INCLUDE
#define TRACEPOINT_INCLUDE "./tp.h"

#if !defined(_TP_H) || defined(TRACEPOINT_HEADER_MULTI_READ)
#define _TP_H

#include <lttng/tracepoint.h>

TRACEPOINT_EVENT(
    my_provider, //provider名字
    my_first_tracepoint, //tracepoint
    TP_ARGS(      //在调用tracepoint函数时需要传入的参数列表
        int, my_integer_arg,
        char*, my_string_arg
    ),
    TP_FIELDS(  //后面详述
        ctf_string(my_string_field, my_string_arg)
        ctf_integer(int, my_integer_field, my_integer_arg)
    )
)

TRACEPOINT_EVENT(
    my_provider,
    my_other_tracepoint,
    TP_ARGS(
        int, my_int
    ),
    TP_FIELDS(
        ctf_integer(int, some_field, my_int)
    )
)

#endif /* _TP_H */

#include <lttng/tracepoint-event.h>
```
`tp.c`
```c
#define TRACEPOINT_CREATE_PROBES

#include "tp.h"
```
>对于`tp.h `文件，可以只写`TRACEPOINT_EVENT`正文部分，保存为.tp文件，然后用`lttng-gen-tp`工具产生这里描述的C和H文件。
>为了方便在其他项目可以使用，一般一个Provider独立在一个C和H里面，不跟具体用户代码交叉放一起
>`tp.c`里面可以包含多个h文件    
>`TP_ARGS`：具体根据用户在调tracepoint函数打印的时候传入的参数，在这里做相应的定义  
>`TP_FIELDS`：在这里定义的各个变量，在运行的时候会被操作，例如这个代码：   

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

TRACEPOINT_EVENT(
    my_provider,
    my_tracepoint,
    TP_ARGS(
        int, my_int_arg,
        char*, my_str_arg,
        struct stat*, st
    ),
    TP_FIELDS(
        ctf_integer(int, my_constant_field, 23 + 17)
        ctf_integer(int, my_int_arg_field, my_int_arg)
        ctf_integer(int, my_int_arg_field2, my_int_arg * my_int_arg)
        ctf_integer(int, sum4_field, my_str_arg[0] + my_str_arg[1] +
                                     my_str_arg[2] + my_str_arg[3])
        ctf_string(my_str_arg_field, my_str_arg)
        ctf_integer_hex(off_t, size_field, st->st_size)
        ctf_float(double, size_dbl_field, (double) st->st_size)
        ctf_sequence_text(char, half_my_str_arg_field, my_str_arg,
                          size_t, strlen(my_str_arg) / 2)
    )
)
```
用户代码：
```c
#define TRACEPOINT_DEFINE
#include "tp.h"

int main(void)
{
    struct stat s;

    stat("/etc/fstab", &s);

    tracepoint(my_provider, my_tracepoint, 23, "Hello, World!", &s);

    return 0;
}
```
>When viewing the trace, assuming the file size of /etc/fstab is 301 bytes, the event generated by the execution of this tracepoint should have the following fields, in this order:
```
my_constant_field           40
my_int_arg_field            23
my_int_arg_field2           529
sum4_field                  389
my_str_arg_field            "Hello, World!"
size_field                  0x12d
size_dbl_field              301.0
half_my_str_arg_field       "Hello,"
```

##编译链接lttng和用户代码
###定义宏：
两个必须定义的宏：`TRACEPOINT_CREATE_PROBES`和`TRACEPOINT_DEFINE`   
`TRACEPOINT_CREATE_PROBES`：在Tracepoint Provider的C文件里面定义，令我们在H文件里面定义的TRACEPOINT_EVENT生效   
`TRACEPOINT_DEFINE`：调用`tracepoint()`的c文件中定义该宏  
    
As discussed above, the macros used by the user-written tracepoint provider header file are useless until actually used to create probes code (global data structures and functions) in a translation unit (C source file). This is accomplished by defining `TRACEPOINT_CREATE_PROBES` in a translation unit and then including the tracepoint provider header file. When `TRACEPOINT_CREATE_PROBES` is defined, macros used and included by the tracepoint provider header produce actual source code needed by any application using the defined tracepoints. Defining `TRACEPOINT_CREATE_PROBES` produces code used when registering tracepoint providers when the tracepoint provider package loads.
The other important definition is `TRACEPOINT_DEFINE`. This one creates global, per-tracepoint structures referencing the tracepoint providers data. Those structures are required by the actual functions inserted where `tracepoint()` macros are placed and need to be defined by the instrumented application.
    
###静态链接：
With the static linking method, compiled tracepoint providers are copied into the target application. There are three ways to do this:    
1.Use one of your existing C source files to create probes.
2.Create probes in a separate C source file and build it as an object file to be linked with the application (more decoupled).
3.Create probes in a separate C source file, build it as an object file and archive it to create a static library (more decoupled, more portable).  
The first approach is to define `TRACEPOINT_CREATE_PROBES` and include your tracepoint provider(s) header file(s) directly into an existing C source file. Here's an example:
```c
#include <stdlib.h>
#include <stdio.h>
/* ... */

#define TRACEPOINT_CREATE_PROBES
#define TRACEPOINT_DEFINE
#include "tp.h"

/* ... */

int my_func(int a, const char* b)
{
    /* ... */

    tracepoint(my_provider, my_tracepoint, buf, sz, limit, &tt)

    /* ... */
}

/* ... */
```
