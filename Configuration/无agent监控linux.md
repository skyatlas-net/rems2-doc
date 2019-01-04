# Agentless Linux 监控
* REMS2 支持对Linux 服务器进行Agentless监控，其实现基于SSH，所以应该保证SSH可联通
* 需要执行ansible-RTM 任务

## REMS2 监控Linux 需要配置的macro

1. SERVER\_ADDR   服务器IP活着dns名称
2. SSH\_PORT   ssh port (default 22)
3. LOGIN\_USER   Login Username
4. LOGIN\_PASSWORD 

## 说明
ansible-RTM 会安装 java 程序，读取REMS2 中的配置（前文所述macro），执行检查