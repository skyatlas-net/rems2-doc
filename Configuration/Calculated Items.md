# Calculated 监控项 #
## 概述
有些时候，直接展现监控获得的数据，会不够直观，对已有的监控数据，进行组合、计算后，所获得的新数据，会更直观、负荷用户的需求。
所以，_Calculated 监控项_ 创建了一个虚拟的数据。定期使用我们定义的计算表达式，获得计算值。计算操作有REMS2 Server执行，计算操作不会在Agent程序或者Proxy节点上执行。
_Calculated 监控项_的数值，会保存在REMS2 Repository数据库中，所以_Calculate 监控项_有_History、Trend_数据，可以利用REMS2的“图表”功能，进行快速的展现。_Calculated 监控项_和普通监控项一样，可以被_Trigger_、_Macro_所引用。
## 配置Calculated 监控项
首先，我们要为_Calculate 监控项_制定一个key，这个key不能与其他监控项冲突。
计算表达式，需要在_Formula_中输入，key 和 计算表达式无法直接关联——即使Key中指定了参数，在计算表达式中也无法使用此参数。
最基础的计算表达式语法如下：
'' func(<key>|<hostname:key>,<parameter1>,<parameter2>...)

func 可以是trigger 表达式中使用的function：last,min,max,avg,count, etc
key 计算时所需要使用的别的监控项，强烈建议将key值用双引号（“）包围，否则如果key中如果有空格、逗号会导致Parse错误，如果key中需要出现双引号，需要使用\\转意。
parameter func所需要的参数。
%% 表达式中涉及的所有key，必须已经定义且能正常获取数据，这些key如果配置被修改，相应的在Calculated 监控项中的定义可能也要作相应修改 
%% 表达式中，使用macro有限制，macro可以在参数、常量中使用，不能用于函数名称、监控节点名称、key、key参数

_Calculate 监控项_ unsupported 的常见原因:
1. 引用的监控项
	- 不存在
	- disabled
	- 所在监控节点 disabled
	- not supported
2. no data
3. 除0
4. 语法错误

## 使用举例
** Example 1: 计算 / 文件系统空闲百分比**
'' 100*last("vfs.fs.size[/,free]")/last("vfs.fs.size[/,total]")
'' 

我们可以看到,两次调用last函数,last函数只制定了一个参数: 我们需要的key
** Exapmple 2: 计算过去10分钟,某监控项的平均直**
'' avg("Zabbix Server:zabbix[wcache,values]",600)

avg 函数,第一个参数是key,用双引号包围,第二个参数是时间, 过去600秒内,该key对应监控项的平均直.
** Example 3: 计算网卡带宽**
'' last("net.if.in[eth0,bytes]")+last("net.if.out[eth0,bytes]")
'' 

** Example 4: 下载流量的百分比**
'' 100*last("net.if.in[eth0,bytes]")/(last("net.if.in[eth0,bytes]")+last("net.if.out[eth0,bytes]"))
'' 

** Example 5: 在表达式中引用聚组监控项**
'' last("grpsum[\"video\",\"net.if.out[eth0,bytes]\",\"last\"]") / last("grpsum[\"video\",\"nginx_stat.sh[active]\",\"last\"]")


