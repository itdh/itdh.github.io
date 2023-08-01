---
title: PostgreSQL定位问题时常用SQL
toc: true
tags:
  - PostgreSQL
categories:
  - PostgreSQL
abbrlink: f1ecacbb
date: 2023-08-01 14:21:30
cover: /img/cover/PostgreSQL定位问题时常用SQL.jpeg
password:
---



# 一、查看当前正在运行的 SQL

## 1.1 SQL 语句

```sql
SELECT 
procpid, 
start, 
now() - start AS lap, 
current_query 
FROM 
(SELECT 
backendid, 
pg_stat_get_backend_pid(S.backendid) AS procpid, 
pg_stat_get_backend_activity_start(S.backendid) AS start, 
pg_stat_get_backend_activity(S.backendid) AS current_query 
FROM 
(SELECT pg_stat_get_backend_idset() AS backendid) AS S 
) AS S 
WHERE 
current_query <> 'idle' 
ORDER BY 
lap DESC; 
```

## 1.2 字段说明

|    字段名     |     说明     |
| :-----------: | :----------: |
|    procpid    |   进程 id    |
|     start     | 进程开始时间 |
|      lap      |   经过时间   |
| current_query | 执行中的 SQL |

## 1.3 停止正在执行的 SQL

### 1.3.1 方法一

```sql
SELECT pg_cancel_backend(进程id);
```

### 1.3.2 方法二

使用系统命令 `kill -9 进程id`

# 二、查看当前事务锁等待、持锁信息的 SQL

## 2.1 SQL 语句

```sql
with    
t_wait as    
(    
  select a.mode,a.locktype,a.database,a.relation,a.page,a.tuple,a.classid,a.granted,   
  a.objid,a.objsubid,a.pid,a.virtualtransaction,a.virtualxid,a.transactionid,a.fastpath,    
  b.state,b.query,b.xact_start,b.query_start,b.usename,b.datname,b.client_addr,b.client_port,b.application_name   
    from pg_locks a,pg_stat_activity b where a.pid=b.pid and not a.granted   
),   
t_run as   
(   
  select a.mode,a.locktype,a.database,a.relation,a.page,a.tuple,a.classid,a.granted,   
  a.objid,a.objsubid,a.pid,a.virtualtransaction,a.virtualxid,a.transactionid,a.fastpath,   
  b.state,b.query,b.xact_start,b.query_start,b.usename,b.datname,b.client_addr,b.client_port,b.application_name   
    from pg_locks a,pg_stat_activity b where a.pid=b.pid and a.granted   
),   
t_overlap as   
(   
  select r.* from t_wait w join t_run r on   
  (   
    r.locktype is not distinct from w.locktype and   
    r.database is not distinct from w.database and   
    r.relation is not distinct from w.relation and   
    r.page is not distinct from w.page and   
    r.tuple is not distinct from w.tuple and   
    r.virtualxid is not distinct from w.virtualxid and   
    r.transactionid is not distinct from w.transactionid and   
    r.classid is not distinct from w.classid and   
    r.objid is not distinct from w.objid and   
    r.objsubid is not distinct from w.objsubid and   
    r.pid <> w.pid   
  )    
),    
t_unionall as    
(    
  select r.* from t_overlap r    
  union all    
  select w.* from t_wait w    
)    
select locktype,datname,relation::regclass,page,tuple,virtualxid,transactionid::text,classid::regclass,objid,objsubid,   
string_agg(   
'Pid: '||case when pid is null then 'NULL' else pid::text end||chr(10)||   
'Lock_Granted: '||case when granted is null then 'NULL' else granted::text end||' , Mode: '||case when mode is null then 'NULL' else mode::text end||' , FastPath: '||case when fastpath is null then 'NULL' else fastpath::text end||' , VirtualTransaction: '||case when virtualtransaction is null then 'NULL' else virtualtransaction::text end||' , Session_State: '||case when state is null then 'NULL' else state::text end||chr(10)||   
'Username: '||case when usename is null then 'NULL' else usename::text end||' , Database: '||case when datname is null then 'NULL' else datname::text end||' , Client_Addr: '||case when client_addr is null then 'NULL' else client_addr::text end||' , Client_Port: '||case when client_port is null then 'NULL' else client_port::text end||' , Application_Name: '||case when application_name is null then 'NULL' else application_name::text end||chr(10)||    
'Xact_Start: '||case when xact_start is null then 'NULL' else xact_start::text end||' , Query_Start: '||case when query_start is null then 'NULL' else query_start::text end||' , Xact_Elapse: '||case when (now()-xact_start) is null then 'NULL' else (now()-xact_start)::text end||' , Query_Elapse: '||case when (now()-query_start) is null then 'NULL' else (now()-query_start)::text end||chr(10)||    
'SQL (Current SQL in Transaction): '||chr(10)||  
case when query is null then 'NULL' else query::text end,    
chr(10)||'--------'||chr(10)    
order by    
  (  case mode    
    when 'INVALID' then 0   
    when 'AccessShareLock' then 1   
    when 'RowShareLock' then 2   
    when 'RowExclusiveLock' then 3   
    when 'ShareUpdateExclusiveLock' then 4   
    when 'ShareLock' then 5   
    when 'ShareRowExclusiveLock' then 6   
    when 'ExclusiveLock' then 7   
    when 'AccessExclusiveLock' then 8   
    else 0   
  end  ) desc,   
  (case when granted then 0 else 1 end)  
) as lock_conflict  
from t_unionall   
group by   
locktype,datname,relation,page,tuple,virtualxid,transactionid::text,classid,objid,objsubid ;  
```

## 2.2 字段说明

|    字段名     |                             说明                             |
| :-----------: | :----------------------------------------------------------: |
|   locktype    | 被锁定的对象类型：relation、extend、page、tuple、transactionid、virtualxid、object、userlock、advisory |
|    datname    |                           数据库名                           |
|   relation    | 如果对象不是表或只是表的一部分，则此值为“NULL”，否则此值是表的OID |
|     page      | 表中的页号，如果对象不是表行（tuple）或表页（relation page），则此值为“NULL” |
|     tuple     |                     页内的行号（tuple）                      |
|  virtualxid   |                          虚拟事务id                          |
| transactionid |                            事务id                            |
|    classid    |                    包含该对象系统目录的id                    |
|     objid     |                     对象在系统目录的oid                      |
|   objsubid    | 如果对象是表列（table column），此列的值为列号，这时classid和objid指向表 |

