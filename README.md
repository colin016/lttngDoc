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
`TRACEPOINT_CREATE_PROBES`：在Tracepoint Provider的C文件里面定义，在H文件里面定义的TRACEPOINT_EVENT才生效   
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
Again, before including a given tracepoint provider header file, `TRACEPOINT_CREATE_PROBES` and `TRACEPOINT_DEFINE` must be defined in one, **and only one**, translation unit. Other C source files of the same application may include tp.h to use tracepoints with the tracepoint() macro, but must not define `TRACEPOINT_CREATE_PROBES`/`TRACEPOINT_DEFINE` again.
This translation unit may be built as an object file by making sure to add . to the include path:   
`gcc -c -I. file.c` 
The second approach is to isolate the tracepoint provider code into a separate object file by using a dedicated C source file to create probes:   
```
#define TRACEPOINT_CREATE_PROBES

#include "tp.h"
```
`TRACEPOINT_DEFINE` must be defined by a translation unit of the application. Since we're talking about static linking here, it could as well be defined directly in the file above, before `#include "tp.h"`:   
```
#define TRACEPOINT_CREATE_PROBES
#define TRACEPOINT_DEFINE

#include "tp.h"
```
This is actually what lttng-gen-tp does, and is the recommended practice.   
Build the tracepoint provider:  
`gcc -c -I. tp.c`   
Finally, the resulting object file may be archived to create a more portable tracepoint provider static library:    
`ar rc tp.a tp.o`   
Using a static library does have the advantage of centralising the tracepoint providers objects so they can be shared between multiple applications. This way, when the tracepoint provider is modified, the source code changes don't have to be patched into each application's source code tree. The applications need to be relinked after each change, but need not to be otherwise recompiled (unless the tracepoint provider's API changes).  
Regardless of which method you choose, you end up with an object file (potentially archived) containing the trace providers assembled code. To link this code with the rest of your application, you must also link with liblttng-ust and libdl:    
`gcc -o app tp.o other.o files.o of.o your.o app.o -llttng-ust -ldl`    
or  
`gcc -o app tp.a other.o files.o of.o your.o app.o -llttng-ust -ldl`    
If you're using a BSD system, replace -ldl with -lc:    
`gcc -o app tp.a other.o files.o of.o your.o app.o -llttng-ust -lc` 
The application can be started as usual, for example:   
`./app` 

###动态链接
The second approach to package the tracepoint providers is to use dynamic linking: the library and its member functions are explicitly sought, loaded and unloaded at runtime using `libdl`.    
It has to be noted that, for a variety of reasons, the created shared library is be dynamically loaded, as opposed to dynamically linked. The tracepoint provider shared object is, however, linked with `liblttng-ust`, so that `liblttng-ust` is guaranteed to be loaded as soon as the tracepoint provider is. If the tracepoint provider is not loaded, since the application itself is not linked with liblttng-ust, the latter is not loaded at all and the tracepoint calls become inert.
The process to create the tracepoint provider shared object is pretty much the same as the static library method, except that:     
    
-since the tracepoint provider is not part of the application anymore, `TRACEPOINT_DEFINE` must be defined, for each tracepoint provider, in exactly one translation unit (C source file) of the application;      
-`TRACEPOINT_PROBE_DYNAMIC_LINKAGE` must be defined next to `TRACEPOINT_DEFINE`.    

Regarding `TRACEPOINT_DEFINE` and `TRACEPOINT_PROBE_DYNAMIC_LINKAGE`, the recommended practice is to use a separate C source file in your application to define them, then include the tracepoint provider header files afterwards. For example: 
```
#define TRACEPOINT_DEFINE
#define TRACEPOINT_PROBE_DYNAMIC_LINKAGE

/* include the header files of one or more tracepoint providers below */
#include "tp1.h"
#include "tp2.h"
#include "tp3.h"
```
`TRACEPOINT_PROBE_DYNAMIC_LINKAGE` makes the macros included afterwards (by including the tracepoint provider header, which itself includes LTTng-UST headers) aware that the tracepoint provider is to be loaded dynamically and not part of the application's executable.   
The tracepoint provider object file used to create the shared library is built like it is using the static library method, only with the -fpic option added:   
`gcc -c -fpic -I. tp.c`
It is then linked as a shared library like this:    
`gcc -shared -Wl,--no-as-needed -o tp.so -llttng-ust tp.o`
As previously stated, this tracepoint provider shared object isn't linked with the user application: it's loaded manually. This is why the application is built with no mention of this tracepoint provider, but still needs `libdl`:    
`gcc -o app other.o files.o of.o your.o app.o -ldl`
Now, to make LTTng-UST tracing available to the application, the `LD_PRELOAD` environment variable is used to preload the tracepoint provider shared library before the application actually starts:  
`LD_PRELOAD=/path/to/tp.so ./app`
