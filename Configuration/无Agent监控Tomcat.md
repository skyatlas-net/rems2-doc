# REMS2 无代理程序监控Tomcat 用户指南
## 对Tomcat Server做的调整
REMS2需要使用JMX协议，对Tomcat Server进行连接、获取Tomcat配置、运行时数据；为保障Tomcat Server的运行安全，严格禁止关闭Tomcat Server的JMX认证功能，REMS2 会在Tomcat上配置：
1. 指定JMX连接的端口，默认9999，如果多个Tomcat运行在同一个服务器上，考虑使用不同的端口
2. 创建configRole和monitorRole，并指定连接密码

_ 操作步骤如下 ：_
1. 修改<TOMCAT\_SERVER>/bin/下的catalina.sh (如果是windows平台，则修改catalina.bat）脚本:
	在 **# ----- Execute The Requested Command ----------------------------------------- ** 行之前，添加如下代码：
```
JAVA_OPTS="-Djava.awt.headless=true -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.authenticate=true -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.password.file=$CATALINA_BASE/conf/jmx_passwdfile -Dcom.sun.management.jmxremote.access.file=$CATALINA_BASE/conf/jmx_accessfile -Djava.rmi.server.hostname=192.168.0.11"
```
	根据需要，调整port，调整hostname为tomcat服务器的IP地址。
2. 在<TOMCAT\_SERVER>/conf 下创建jmx\_passwdfile和jmx\_accessfile
```
# cat jmx_accessfile
    monitorRole readonly
    controlRole readwrite
# cat jmx_passwdfile
    monitorRole	tomcat
    controlRole	tomcat
```
	jmx\_passwdfile文件中的tomcat，是连接Tomcat所需要的认证密码，可以根据需要修改。
3. 完成修改后，需要重新启动Tomcat
## REMS2 中需要完成的配置
1. 添加监控host
2. 在host上应用template ： Template Tomcat JMX Trapper
3. 在host上添加 MACRO
	- \{$TOMCAT\_PASSWORD} : 连接Tomcat的密码
	- \{$TOMCAT\_USER} : 连接Tomcat的用户（上文中的controlRole）
	- \{$TOMCAT\_SERVER} : Tomcat的监听IP地址（上文中的192.168.0.11）
	- \{$TOMCAT\_PORT}: Tomcat的JMX监听端口（上文中的9999）