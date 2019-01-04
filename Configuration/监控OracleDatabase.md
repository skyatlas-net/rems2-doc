# REMS2 监控Oracle 需要配置的macro
```
{$ORA\_SERVER}
{$ORA\_PORT}
{$ORA\_USER}
{$ORA\_PASSWORD}
{$ORA\_SERVICE}
```
# Oracle 数据库监控实现机制说明
完成rems2 server安装后，ansible 任务会自动在 /home/oracle/rmsora 中安装python程序，负责对Oracle数据库的监控（需要的Oracle Client等，都会在ansible任务中自动安装完成）
## rmsora python程序配置说明
rmsora python会根据配置，自动完成对Oracle数据库的监控，其具备以下特点：
* 自动运行（通过crontab调度）
* 配置修改后不需要重启，自动生效
* 配置通过SQL语句实现
### 每一个要监控的Oracle数据库（RAC集群不管多少节点，都当作一个监控目标），需要一个如下配置文件
文件存储在 /home/oracle/rmsora/etc/ ， 文件名称 rmsora_\<hostname\>.cfg
``` 
[rmsora]
db_url: //localhost:15214/fsdb02
username: cistats
password: knowoneknows
role: normal
# for ASM instance role should be SYSDBA
out_dir: $HOME/rmsora_out
hostname: testhost          
visible_name: visible name
checks_dir: etc/rmsora_checks
# site_checks: sap,ebs
site_checks: extra,fileio,sga_sess
rems_server: 127.0.0.1 
```
说明：
1. db_url: //\<database_ip\>:\<listener_port\>/\<service_name\>
2. username: 连接的用户名
3. password: 
4. role: normal|sysdba
5. out_dir:
6. hostname:  这个数据库监控目标，在REMS2中的名称（全局唯一）
7. visible_name: 显示名称
8. checks_dir: 检查、监控内容存储位置（SQL语句）
9. site_checks: 额外的检查内容
10. rems_server: rems2 服务器IP地址
在 etc/rmsora_checks 文件夹内，存储着多个 .cfg 文件，其中存有要执行的SQL检查，以及检查的时间间隔等，rmsora 会自动根据连接的数据库角色（primary， standby，asm）、版本选择执行的主检查 cfg 文件，比如 primary数据库，版本11，会执行primary.11.cfg
### primary.11.cfg 文件说明
```
# vim: syntax=sql
[auto_discovery_1000]
minutes: 1000
expu.lld: select '' "{#PDB}", username "{#USERNAME}"
            from dba_users s
            where account_status IN ( 'OPEN', 'EXPIRED(GRACE)' )
            and expiry_date > sysdate
            and expiry_date < (sysdate + 30)
ustat.lld: select '' "{#PDB}", account_status "{#STATUS}"
            from dba_users
            group by account_status
[auto_discovery_60]
minutes: 60
inst.lld: select distinct inst_name "{#INST_NAME}"
            from (select inst_name from v$active_instances 
                  union
                  select instance_name from gv$instance)
db.lld: select name "{#PDB}" from v$database
parm.lld: select i.instance_name "{#INST_NAME}", p.name "{#PARAMETER}"
            from gv$instance i, gv$parameter p
            where i.instance_number = p.inst_id
            and   p.type in (3,6) and p.isdefault = 'FALSE'
p_ts.lld: select tablespace_name "{#TS_NAME}", '' "{#PDB}"
            from dba_tablespaces where contents = 'PERMANENT'
t_ts.lld: select tablespace_name "{#TS_NAME}", '' "{#PDB}"
            from dba_tablespaces where contents = 'TEMPORARY'
u_ts.lld: select tablespace_name "{#TS_NAME}", '' "{#PDB}"
            from dba_tablespaces where contents = 'UNDO'
service.lld: select '' "{#PDB}", i.instance_name "{#INST_NAME}", s.name "{#SERVICE_NAME}"
               from gv$services s join gv$instance i
                 on (   s.inst_id = i.inst_id)
rman.lld: select distinct(object_type) "{#OBJ_TYPE}" from v$rman_status where operation like 'BACKUP%'
arl_dest.lld: select i.instance_name "{#INST_NAME}",d.dest_name "{#ARL_DEST}"
            from gv$archive_dest d
            , gv$instance i
            , v$database db
            where d.status != 'INACTIVE'
            and   d.inst_id = i.inst_id
            and   db.log_mode = 'ARCHIVELOG'
[startup]
minutes: 0
version: select 'inst['||instance_name||',version]', version from gv$instance
lastpatch: select  'db[last_patch_hist]', ACTION||':'||NAMESPACE||':'||VERSION||':'||ID||':'||COMMENTS
        from sys.registry$history
        where action_time = (select max(action_time) from sys.registry$history)

[checks_01m]
minutes: 1
inst.uptime: select 'inst['||instance_name||',uptime]' key,(sysdate -startup_time)*60*60*24 val from gv$instance
db.openmode: select 'db['||name||',openstatus]', decode(open_mode,'MOUNTED',1,'READ ONLY',2,'READ WRITE',3,0) from v$database
scn: select 'db[current_scn]', current_scn from v$database
     union all
     select 'db[delta_scn]', current_scn from v$database
blocked: select 'blocked[topsid]', topsid||'('||blocked||')'
          from (
          select final_blocking_instance||'/'||final_blocking_session topsid, count(*) blocked
          from gv$session
          where  FINAL_BLOCKING_SESSION_STATUS='VALID'
          group by final_blocking_instance||'/'||final_blocking_session
          order by 2 desc, 1
          )
          where rownum < 2
          union all
          select 'blocked[count]', ''||count(*)
           from gv$session 
           where  FINAL_BLOCKING_SESSION_STATUS='VALID'
[checks_05m]
minutes: 5
parm.val:  select 'parm['||i.instance_name||','||p.name||',value]' key, p.value
            from gv$instance i, gv$parameter p
            where i.instance_number = p.inst_id
            and   p.type in (3,6) and p.isdefault = 'FALSE'
            and   upper(p.description) not like '%SIZE%'
            union all
            select 'parm['||i.instance_name||','||p.name||',size]' key, p.value
            from gv$instance i, gv$parameter p
            where i.instance_number = p.inst_id
            and   p.type in (3,6) and p.isdefault = 'FALSE'
            and   upper(p.description) like '%SIZE%'
service.cnt: select 'service[,'||i.instance_name||','|| s.service_name||',sess]' ,count(*)
               from gv$session s join gv$instance i
                 on (   s.inst_id = i.inst_id)
                 group by i.instance_name, s.service_name

u_ts: SELECT   'u_ts[,'||tablespace_name||','||
           CASE
             WHEN k = 1 THEN 'filesize]'
             WHEN k = 2 THEN 'maxsize]'
             WHEN k = 3 THEN 'usedbytes]'
             WHEN k = 4 THEN 'pctfree]'
           END key
  ,        CASE
           WHEN k = 1 THEN file_size
           WHEN k = 2 THEN file_max_size
           WHEN k = 3 THEN file_size - file_free_space
           WHEN k = 4 THEN ROUND(file_free_space / file_size * 100)
           END value
  FROM   ( --
         SELECT   t.tablespace_name
         ,        SUM(f.bytes) file_size
         ,        SUM(CASE
                        WHEN f.autoextensible = 'NO'
                        THEN f.bytes
                        ELSE GREATEST(f.bytes, f.maxbytes)
                      END) file_max_size
         ,        SUM(NVL(( SELECT   SUM(a.bytes)
                            FROM     dba_free_space a
                            WHERE    a.tablespace_name = t.tablespace_name
                            AND      a.file_id         = f.file_id
                            AND      a.relative_fno    = f.relative_fno
                          ), 0)) file_free_space
         FROM     dba_tablespaces t
         JOIN     dba_data_files f
         ON     ( f.tablespace_name = t.tablespace_name )
         WHERE    t.CONTENTS = 'UNDO'
         GROUP BY t.tablespace_name
       )
  cross JOIN   ( SELECT LEVEL k FROM dual CONNECT BY LEVEL <= 4 ) k

t_ts: select   't_ts[,'||t.TABLESPACE||',filesize]', t.totalspace
    from (select   round (sum (d.bytes))  AS totalspace,
                   round (sum ( case when maxbytes < bytes then bytes else maxbytes end)) max_bytes,
									 d.tablespace_name tablespace
              from dba_temp_files d
          group by d.tablespace_name) t
   union all
   select   't_ts[,'||t.TABLESPACE_name||',maxsize]', sum(maxbytes)
        from (select case when autoextensible = 'NO'
                               then bytes
                     else
                      case when bytes > maxbytes
                               then bytes
                      else          maxbytes
                      end
                     end maxbytes, tablespace_name
                from dba_temp_files) f
            , dba_tablespaces t
       where t.contents = 'TEMPORARY'
         and  t.tablespace_name = f.tablespace_name
       group by t.tablespace_name
  union all
  select 't_ts[,'||t.tablespace_name||',usedbytes]', nvl(sum(u.blocks*t.block_size),0) bytes
    from gv$sort_usage u right join
       dba_tablespaces t
           on ( u.tablespace = t.tablespace_name)
             where   t.contents = 'TEMPORARY'
               group by t.tablespace_name
  union all
  select   't_ts[,'||t.TABLESPACE_name||',pctfree]', round(((t.totalspace - nvl(u.usedbytes,0))/t.totalspace)*100) "PCTFREE"
    from (select   round (sum (d.bytes))  AS totalspace,
                   round (sum ( case when maxbytes < bytes then bytes else maxbytes end)) max_bytes,
                   d.tablespace_name
              from dba_temp_files d
          group by d.tablespace_name) t
      left join (
                        select u.tablespace tablespace_name, round(sum(u.blocks*t.block_size)) usedbytes
                        from gv$sort_usage u
                        , dba_tablespaces t
                        where u.tablespace = t.tablespace_name
                        and   t.contents = 'TEMPORARY'
                        group by tablespace
                 )u
           on t.tablespace_name = u.tablespace_name
arl_dest: select 'arl_dest['|| i.instance_name||','||d.dest_name||',status]', ''||decode (d.status,'VALID',0,'DEFERRED',1,'ERROR',2,3)
            from gv$archive_dest d
            , gv$instance i
            , v$database db
            where d.status != 'INACTIVE'
            and   d.inst_id = i.inst_id
            and db.log_mode = 'ARCHIVELOG'
          union all
          select 'arl_dest['|| i.instance_name||','||d.dest_name||',sequence]', ''||d.log_sequence
            from gv$archive_dest d
            , gv$instance i
            , v$database db
            where d.status != 'INACTIVE'
            and   d.inst_id = i.inst_id
            and db.log_mode = 'ARCHIVELOG'
          union all
          select 'arl_dest['|| i.instance_name||','||d.dest_name||',error]', '"'||d.error||'"'
            from gv$archive_dest d
            , gv$instance i
                , v$database db
            where d.status != 'INACTIVE'
            and   d.inst_id = i.inst_id
            and db.log_mode = 'ARCHIVELOG'
fra: select 'fra[limit]', space_limit from v$recovery_file_dest def
      union all
     select 'fra[used]', space_used from v$recovery_file_dest def
      union all
     select 'fra[reclaimable]', space_reclaimable from v$recovery_file_dest def
      union all
     select 'fra[files]', number_of_files from v$recovery_file_dest def
[checks_20m]
minutes: 20
rman: with stats as (
        select r.object_type, r.operation, r.start_time, r.end_time, r.status
               ,max(start_time) over (partition by  r.object_type, r.operation) max_start
               , input_bytes, output_bytes
        from v$rman_status r
        where r.row_type = 'COMMAND'
        and   not r.object_type is null
        and   r.operation like 'BACKUP%'
        )
        , types as (
        select 'ARCHIVELOG' object_type from dual
        union all
        select 'CONTROLFILE' from dual
        union all
        select 'SPFILE' from dual
        union all
        select 'DB INCR' from dual
        union all
        select 'DB FULL' from dual
        union all
        select 'RECVR AREA' from dual
        )
        , data as (
        select t.object_type, s.start_time, nvl(s.status,'noinfo') status, round(nvl((s.end_time - s.start_time),0)*24*60*60) elapsed
        , nvl(input_bytes,0) input_bytes, nvl(output_bytes,0) output_bytes
        from types t left outer join
             stats s on (s.object_type = t.object_type)
        where nvl(s.start_time,sysdate) = nvl(s.max_start,sysdate)
        )
        select '"rman['||object_type||',status]"', ''||decode(status,'COMPLETED',0,
                                               'FAILED',   1,
                                               'COMPLETED WITH WARNINGS',2,
                                               'COMPLETED WITH ERRORS',  3,
                                               'noinfo',                 4,
                                               'RUNNING',                5,
                                               9) status
        from data
        union all
        select '"rman['||object_type||',ela]"',''||elapsed
        from data
        union all
        select '"rman['||object_type||',input]"',''||input_bytes
        from data
        union all
        select '"rman['||object_type||',output]"',''||output_bytes
        from data
        union all
        select '"rman['||object_type||',age]"',''||round((sysdate - nvl(start_time,sysdate))*24*3600) age
        from data
        union all select 'rman[bct,status]', ''||decode(status,'ENABLED',0,'DISABLED',1,2) from v$block_change_tracking
        union all select 'rman[bct,file]', filename from v$block_change_tracking
        union all select 'rman[bct,bytes]', ''||nvl(bytes,0) from v$block_change_tracking
[checks_60m]
minutes: 60
p_ts: SELECT   'p_ts[,'||tablespace_name||','||
           CASE
             WHEN k = 1 THEN 'filesize]'
             WHEN k = 2 THEN 'maxsize]'
             WHEN k = 3 THEN 'usedbytes]'
             WHEN k = 4 THEN 'pctfree]'
           END key
  ,        CASE
           WHEN k = 1 THEN file_size
           WHEN k = 2 THEN file_max_size
           WHEN k = 3 THEN file_size - file_free_space
           WHEN k = 4 THEN ROUND(file_free_space / file_size * 100)
           END value
  FROM   ( --
         SELECT   t.tablespace_name
         ,        SUM(f.bytes) file_size
         ,        SUM(CASE
                        WHEN f.autoextensible = 'NO'
                        THEN f.bytes
                        ELSE GREATEST(f.bytes, f.maxbytes)
                      END) file_max_size
         ,        SUM(NVL(( SELECT   SUM(a.bytes)
                            FROM     dba_free_space a
                            WHERE    a.tablespace_name = t.tablespace_name
                            AND      a.file_id         = f.file_id
                            AND      a.relative_fno    = f.relative_fno
                          ), 0)) file_free_space
         FROM     dba_tablespaces t
         JOIN     dba_data_files f
         ON     ( f.tablespace_name = t.tablespace_name )
         WHERE    t.CONTENTS = 'PERMANENT'
         GROUP BY t.tablespace_name
       )
  cross JOIN   ( SELECT LEVEL k FROM dual CONNECT BY LEVEL <= 4 ) k
expu: select 'expu[,'|| username||',expiring]' key, (expiry_date - sysdate)*24*3600 value
	from dba_users s
	where account_status IN ( 'OPEN', 'EXPIRED(GRACE)' )
	and expiry_date > sysdate
	and expiry_date < (sysdate + 30)
  union all
  select '"ustat[,'||account_status||',count]"' key, count(*) value
  from dba_users
  group by account_status
alertlog: select 'inst['||i.instance_name||',log]', d.value||'/alert_'||i.instance_name||'.log' from gv$instance i, gv$diag_info d
           where i.inst_id = d.inst_id and d.name = 'Diag Trace'
[checks_720m]
minutes: 720
version: select 'inst['||instance_name||',version]', version from gv$instance
lastpatch: select  'db[last_patch_hist]', ACTION||':'||NAMESPACE||':'||VERSION||':'||ID||':'||COMMENTS
        from sys.registry$history
        where action_time = (select max(action_time) from sys.registry$history)
```
其格式为python options文件格式，了解格式后，可以自行编写SQL，添加

## REMS2 中的配置工作
在rems2中，需要按照常规完成添加节点、链接模板工作。
* 如果通过ansible 安装了 ansible-RTM 任务，在REMS2 中完成节点添加、模板链接后，会自动完成rmsora的相关工作
* 否则，需要在rmsora中添加 rmsora_\[hostname\].cfg 文件
* 如希望自动完成rmsora配置，需要在 rems2 中的节点上配置 MACRO ，如本文开头所述
