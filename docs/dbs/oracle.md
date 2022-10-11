# ORACLE

## 一.简单命令

```
本机登陆 sqlplus  / as sysdba

.创建一个新用户：create user wudidehuangtiandi identified by 123456;

.授予DBA权限： grant connect,resource,dba to wudidehuangtiandi;

创建永久表空间：（如不创建则使用默认永久表空间）

-- 创建表空间

create tablespace PIVAS datafile 'PIVAS.dbf' size 10m;

-- 设置表空间自动增长
alter database datafile 'PIVAS.dbf' autoextend on;


-- 创建用户,并默认表空间
create user wudidehuangtiandi identified by interface account unlock default tablespace PIVAS

-- 给新用户授权
grant connect,resource,dba to wudidehuangtiandi ; 

alter user wudidehuangtiandi default tablespace PIVAS ;
解释：以上语句就是说给username用户重新指定表空间为userspace；
扩展：创建用户的时候指定表空间。
sql：create user wudidehuangtiandi identified by 123456 default tablespace PIVAS ;

以上为oracle新建表空间的方法，搞完了换个用户登录即可

```

