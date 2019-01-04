# Rems2 监控 Apache 

系统环境：centos6.5 x64
apache: httpd-2.4.4

 
首先在本机下载模板：[模板](https://github.com/rdvn/zabbix-templates/archive/master.zip)
该zip包有apache、memcache、redis、varnish模板,我们解压后使用其中的apache模板(模板文件需要编辑修改)
 
## 一.打开apache的server-status：
 
    `# vi /usr/local/apache2/conf/httpd.conf`
末行添加如下内容：
```
---------------------
ExtendedStatus On
<location /server-status>
SetHandler server-status
Order Allow,Deny
Allow from all
</location>
---------------------
```
重启apache使其生效：
    ```
    # /usr/local/apache2/bin/apachectl restart
    ```
## 二.rems2 配置：
 
将下载下来的zip包内apache目录下的apache_status.sh上传到系统/usr/local/bin/下，并赋予
执行权限
    ```
    # chmod +x apache_status.sh
    # ll /usr/local/bin/apache_status.sh
    ---------------
    -rwxr-xr-x 1 root root 248 4月 23 2012 apache_status.sh
    ---------------
    ```
在 rems2 中可以自定义监控变量，通过自己写的bash脚本来抓取相关信息返回给rems2 server，这里我们需要在运行rems2 agent的服务器上编辑/etc/rems2/zabbix_agentd.conf (也可能在/usr/local/etc 内)
修改rems2_agentd.conf配置：
   ```
    # vi /usr/local/etc/rems2_agentd.conf
    ```
末行添加如下内容：
    ```
    -------------
    UserParameter=apache\[\*\],/usr/local/bin/apache_status.sh $1
    -------------
    ```
其中apache\[*\]是定义的zabbix agent变量，/data/shells/apache_status.sh 定义这个变量的动作脚本。
重启rems2 agentd 服务
    ```
    # pkill rems2_agentd
    # /etc/init.d/rems2_agentd start
    ```