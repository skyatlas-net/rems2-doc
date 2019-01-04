REMS2 LLD (Low-level discovery)
# 概述
LLD允许我们自动创建监控项目、触发器、数据图表等。比如，Rems2可以自动创建一个机器上的文件系统活着网卡，而不需要手工为每一个文件系统或者网卡进行配置。再举个例子，监控Oracle表空间，不通系统的表空间名称、数量完全不通，我们可能希望能够自动根据表空间的具体名称，自动为每一个表空间创建诸如“剩余”空间的监控项目。
Rems2中，有六种自动发现的监控项目是开箱即用的：
1. 文件系统
2. 网卡
3. CPU或者CPU核
4. SNMP OID
5. ODBC的SQL查询
6. Windows系统的服务
用户可以使用一个特别的JSON数据协议，定义自由的“自动发现”策略。
首先用户需要在模板配置页的“自动发现”模块下创建discovery。“自动发现”规则包含：
1. 一个监控项目，用于自动发现必要的实体（如：文件系统、网卡、Oracle表空间）
2. 监控项目的属性（Prototype）、触发器、数据图表
自动发现的流程没有什么特别：服务器要求Rems agent（或者其他模块）提供监控项目，agent返回文本值。差别在于，这个文本值是JSON格式，其包含被“发现”的被监控实体列表。返回的数据中，会包含一些列macro－value对，比如：网络监控项”net.if.discovery”会返回“\{#IFNAME}”->”lo” 和”\{#IFNAME}”->”eth0”。这些返回的“macro”在后续的自动创建监控项、触发器、数据图表设置“监控节点”时，会作为“name”、“key”之类的属性使用。
接下来的部分解释一下——how－to－使用、配置上述过程：
## Discovery of file systems
```
Configuration -> Templates -> Discovery -> Create discovery rule …
name —>  discovery 规则的名称
Type —>  执行discovery的类型，Rems agent、rems agent ，如果自定义的是不是就得是Rems2 trapper？？
Key —> 对应type的key值，需要返回json
update interval —> 执行discovery操作的间隔
custom intervals —> …
keep lost resources period(in days) —> 自动发现的监控项，即使“not discovered anymore”仍然保留天数
description，enabled …
_Filters_ tab 设置“自动发现”规则的过滤条件定义
可以定义逻辑表达式 —> A or (B and C) …
每一个逻辑但愿都是 诸如 返回的 “MACRO” 匹配 某 regex ？
创建完这个“filesystem” 自动发现规则后，在该规则的items（对应这个规则，自动创建的监控项）部分点击“Create prototype“ ：
Name —> 监控项名称，可以带占位符（$1 $2 …) Free disk space on $1(percentage)
Type —> 监控项执行类型（Rems2 agent、Rems2 trapper…),此处监控文件系统，使用Rems2 agent
key —> 监控项key（可使用MACRO，此处监控文件系统，使用vfs.fs.size\[\{#FSNAME}],pfree）
Type of information —> Numeric(float) 返回数据类型
Units —> %  百分比
。。。
同样的，我们创建trigger prototypes…
Name —> Free disk space is less than 20% on volume \{#FSNAME}
Expression —> \{Template OS Linux:vfs.fs.size\[\{#FSNAME},pfree].last(0)}<20
…
当前版本的Rems2，trigger prototypes之间可以定义dependencies关系，一个trigger prototype可以依赖于同一个LLD 规则定义的trigger prototype或者一个常规trigger。一个trigger prototype不要去依赖于其他LLD规则定义的trigger prototype。监控节点的trigger prototype不能依赖于模板的trigger prototype。
同样的，我们可以创建数据图表prototype：
    定义方式跟常规数据图表没有什么区别，但是选择的items是基于发现规则创建的item prototype。
这样，“自动发现”规则的整个过程就这些了。
需要注意，自动发现规则创建“实体”名称不能已经在系统对应“namespace”中存在。
另外，LLD创建的监控项、触发器、数据图表不能够被手工删除。如果被发现的entities消失，它们会自动被删除（Keep lost resources period）。
Entities如果包含其他被标志为“待删除”的entities，将不会再更新数据，如：一个LLD触发器，如果包含“待删除”监控项，将不再会更新。
```
## Discovery of network interfaces
		net.if.discovery, net.if.in\[\{#IFNAME},bytes],net.if.out\[\{#IFNAME},bytes]
## Discovery of CPUs and CPU cores
		system.cpu.discovery —> \{#CPU.NUMBER} \{#CPU.STATUS}
## Discovery using ODBC SQL queries
		可以给予SQL 查询来实现LLD，SQL查询的结果会自动转换为JSON对象（数据列名为macro名称，数据值为macro值）。比较麻烦的是这里执行SQL需要通过ODBC，既然大家监控Oracle之类的都不愿意通过ODBC这种方式，那么我想必然有其原因。
## Discovery of Windows services
与文件系统“自动发现”类似，但该自动发现规则key是service.discovery,能够返回如下macros：
* \{#SERVICE.NAME}
* \{#SERVICE.DISPLAYNAME}
* \{#SERVICE.DESCRIPTION}
* \{#SERVICE.STATE}
* \{#SERVICE.STATENAME}
* \{#SERVICE.PATH}
* \{#SERVICE.USER}
* \{#SERVICE.STARTUP}
* \{#SERVICE.STARTUPNAME}
## CREATING CUSTOM LLD 规则
		我们可以使用UserParameter，配置一个特定的key，用于调用客户端脚本（sh，perl。。。）脚本返回JSON对象，Rems2根据JSON来定义LLD规则。	