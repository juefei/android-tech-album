# 进程调度cgroups介绍

&emsp;&emsp;cgroups，其名称源自控制组群（control groups）的简写，是Linux内核的一个功能，用来限制，
控制与分离一个进程组群的资源（如CPU、内存、磁盘输入输出等）。几个重要概念说明：

* cpu.share：cpu.share文件中保存了整数值，用来设置cgroup分组任务获得CPU时间的相对值。
举例来说，cgroup A和cgroup B的cpu.share值都是1024，那么cgroup A 与cgroup B中的任务分配到的CPU时间相同，
如果cgroup C的cpu.share为512，那么cgroup C中的任务获得的CPU时间是A或B的一半。。
* cpu.rt_period_us：设置cgroup获得CPU资源分配的周期，单位为微秒。 
* cpu.rt_runtime_us：设置cgroup组内的进程在一次CPU分配周期（即cpu.cfs_period_us指定的值）内可以执行的时间，单位为微秒。
设定这个值可以访问某个cgroup独占CPU资源。最长的获取CPU资源时间取决于逻辑CPU的数量。比如cpu.rt_runtime_us设置为200000（0.2秒），cpu.rt_period_us设置为1000000（1秒）。
在单个逻辑CPU上的获得时间为每秒为0.2秒。2个逻辑CPU，获得的时间则是0.4秒。


# 进程调度cgroups在Android上应用
   
&emsp;&emsp;在Android中也存在cgroups，涉及到CPU的目前只有两个，一个是apps，路径为/dev/cpuctl/apps。
   另一个是bg_non_interactive，路径为/dev/cpuctl/apps/bg_non_interactive。
    
* app的cpu.share

    1.apps下的cpu.share值为1024：
    
        shell@trltechn:/dev/cpuctl/apps $ cat cpu.shares                               
        1024
        
    2.bg_non_interactive下的cpu_share值为52：
    
        shell@trltechn:/dev/cpuctl/apps/bg_non_interactive $ cat cpu.shares            
        52    
        
&emsp;&emsp;也就是说apps分组与bg_non_interactive分组cpu.share值相比接近于20:1。由于Android中只有这两个cgroup，
也就是说apps分组中的应用可以利用95%的CPU，而处于bg_non_interactive分组中的应用则只能获得5%的CPU利用率。 
 
* app的cpu.rt_period_us和cpu.rt_runtime_us：    
    1.apps分组下的两个配置的值：
    
        shell@trltechn:/dev/cpuctl/apps # cat cpu.rt_period_us
        1000000
        shell@trltechn:/dev/cpuctl/apps # cat cpu.rt_runtime_us
        800000
        
        即单个逻辑CPU下每一秒内可以获得0.8秒的执行时间。
        
    2.bg_non_interactive分组下的两个配置的值：
    
        shell@trltechn:/dev/cpuctl/apps/bg_non_interactive # cat cpu.rt_period_us 
        1000000
        shell@trltechn:/dev/cpuctl/apps/bg_non_interactive # cat cpu.rt_runtime_us
        700000
        即单个逻辑CPU下每一秒可以获得0.7秒的执行时间。
        
* app状态与cgroups关系：
    
    1.Activity：
         
        当一个Activity处于可见的状态下，那么这个应用进程就属于apps分组。   
        当一个Activity处于后台不可见的状态下，那么这个应用进程就属于bg_non_interactive分组。     
        
    2.Service： 
            
        当Service调用startForeground方法后，那么这个应用进程则是归类于apps分组。
        
* 如何确定app进程的cgroups：

    1.进入已经root的Android设备终端
    
    2.获取当前进程id.
    
        shell@trltechn:/dev/cpuctl/apps/bg_non_interactive $ ps | grep com.tmall       
        u0_a471   8881  373   1980676 60136 ffffffff 00000000 S com.tmall.wireless
        u0_a471   14803 373   1911244 40268 ffffffff 00000000 S com.tmall.wireless:channel
        
    3.利用进程id查看其所在的cgroups
        
        app在前台时：
        shell@trltechn:/dev/cpuctl/apps/bg_non_interactive $ cat /proc/8881/cgroup     
        2:cpu:/apps
        1:cpuacct:/uid_10471/pid_8881
    
        app在后台时：
        shell@trltechn:/dev/cpuctl/apps/bg_non_interactive $ cat /proc/8881/cgroup
        2:cpu:/apps/bg_non_interactive
        1:cpuacct:/uid_10471/pid_8881
        
* 利用cgroups可以做什么：

    1.通过查看app的cgroups判断app当前状态：前台，后台，关闭。
    
    2.可以参照cgroups的分组的思想来设计有类似场景的方案解决实际问题。
    