# 函数

## 1. 函数的基本介绍

### 1.1 函数分类

- Aggregate函数

> Aggregate 函数的操作面向一系列的值，并返回一个单一的值。例如max(字段)

- Scalar 函数

> Scalar 函数的操作面向某个单一的值，并返回基于输入值的一个单一的值。例如ucase(字段)

### 1.2 函数的语法

```sql
select fun(列) from 表
```

举例：

```sql
select max(sale_count) from sales
```

### 1.3 内建函数

>内建函数即指由语法规定存在的函数，包含在编译器的运行时库当中，程序员不必单独书写代码实现它，只需要调用既可，他们的实现，由编译器厂商完成

#### 1.3.1 常用的内建函数

| 类型 | 作用 | oracle | mysql | postgre |
| ------ | ------ | ------ | ------ | ------ |
| 聚合函数 | 平均数 | avg() | avg() | avg() |
| 聚合函数 | 计算行数 | count() | count() | count() |
| 聚合函数 | 最大值 | max() | max() | max() |
| 聚合函数 | 最小值 | min() | min() | min() |
| 聚合函数 | 求和 | sum() | sum() | sum() |
| 聚合函数 | 值拼接 | wm_concat() | group_concat() | string_agg() |
| 数学函数 | 绝对值 | abs() | abs() | abs() |
| 数学函数 | 不小于参数的最小整数 | ceil() | ceil() | ceil() |
| 数学函数 | 不大于参数的最大整数 | exp() | exp() | exp() |
| 数学函数 | 自然指数 | floor() | floor() | floor() |
| 数学函数 | 圆整为最接近的整数 | round() | round() | round() |
| 数学函数 | 平方根 | sqrt() | sqrt() | sqrt() |
| 数学函数 | 截断 | trunc() | truncate() | trunc() |
| 数学函数 | 反余弦 | acos() | acos() | acos() |
| 数学函数 | 反正弦 | asin() | asin() | asin() |
| 数学函数 | 反正切 | atan() | atan() | atan() |
| 数学函数 | 余弦 | cos() | cos() | cos() |
| 数学函数 | 余切 | cot() | cot() | cot() |
| 数学函数 | 正弦 | sin() | sin() | sin() |
| 数学函数 | 正切 | tan() | tan() | tan() |
| 字符串函数 | 首尾去空 | trim() | trim() | trim() |
| 字符串函数 | 大写 | upper() | upper() | upper() |
| 字符串函数 | 小写 | lower() | lower() | lower() |
| 字符串函数 | 截取字符串 | substring() | substring() | substring() |
| 字符串函数 | 拼接符 | \| | 无 | \||
| 类型转换函数 | 将时间戳转换为字符串 | to_char() | date_format() | to_char() |
| 类型转换函数 | 字符串转换为日期 | to_date() | str_to_date() | to_date() |

##### 1.3.2 分析函数和聚合函数的差别
>
>1. 分组方式：以oracle为例聚合函数使用 GROUP BY 分组，分析函数使用 PARTITION BY 进行分组。
>2. 返回值：分析函数会根据表中的行数，每行返回一个计算结果，而聚合函数只返回一个结果或者每一个组返回一个结果。
>
>3. 聚合函数通常是能直接使用的 如max()、min()、avg()、count()等。
>
>4. 聚合函数若是要和其余的字段一块儿查询，那么其余的字段必须是分组的字段，根据分组的字段，每一个组返回一个计算结果。
>5. ORDER BY 在聚合函数中，只是单纯的排序。在分析函数中除了能够排序，还能够累计求值。
>6. 分析函数几乎没有任何限制，而聚合函数使用限制比较多的。

###### 1.3.2.1 oracle分析函数

普通形态：over()
进阶形态：聚合函数 over()
如 avg()over()

```sql
select e.* ,avg(e.sal) over(), (select avg(sal) from emp ) from emp e
```

高级形态：聚合函数 over(partition by 分组字段1,分组字段2)
如：avg()over(partition by)

```sql
select e.* ,avg(e.sal) over(partition by e.deptno) 各部门平均工资 from emp e
```

终极形态：聚合函数 over(partition by 分组字段1,分组字段2 order by 排序字段1,排序字段2..)
如：查询每一年各月，各部门，各类岗位的人数函数

```sql
select DEPTNO 部门,avg(DEPTNO)over() 部门平均值,max(DEPTNO)over(order by DEPTNO asc),min(DEPTNO)over(order by DEPTNO asc) from DEPT
```

###### 1.3.2.2 mysql分析函数

```sql
 substring_index( group_concat(xxx order by vvv desc) ,',',1) t 
```
>
>- row_number() over(order by ): 变量会最后再计算，所以是先排序好之后，才会开始计算
>- row_number() over(partition by order by ): 实现分组排名效果
>- dense_rank() over(order by): 变量会最后再计算，所以是先排序好之后，才会开始计算
>- rank() over()

###### 1.3.2.3 postgre分析函数
>
>- row_number()：在一个结果集中，返回当前的行的号码。
>- rank()、dense_rank()：在一个结果集中，用来排名，前者是完全差异后者是不完全差异，简言之前者是按阿拉伯数字顺序来，后者则会跳跃。
>- lag(value any)、lead(value any)：用来对当前行对于指定的字段与下一行或者前一行的值进行比较。
>- first_value(value any)、last_value(value any)：在一个窗口中，返回指定排序的第一个值和最后一个值。
>- 其他类似与sum(),agv()，max(),min()也都是能与窗口函数配合使用，当做分析函数。
>
### 1.4 自定义函数

#### 1.4.1 oracle

参考链接：https://blog.csdn.net/qq_31652795/article/details/116381604

##### 1.4.1.1 基本语法

```sql
create [or replace] function 函数名 
([p1,p2...pn])
return datatype
is|as
--声明部分 
begin
--程序块
end
```

- 1、`function` 是创建函数的关键字。
- 2、p1,p2...pn是函数的入参，Oracle创建的函数也可以不需要入参。
- 3、return datatype:是函数的返回值的类型
  
##### 1.4.1.2 无参函数定义

```sql
create or replace function 函数名
return 类型
is
begin
  return xxx;
end;
-- 调用方式1
select 函数名() from dual;
-- 调用方式2
begin
  dbms_output.put_line(函数名)
end;  
```

==dbms_output.put_line：在serveroutput on的情况下，用来使dbms_output生效(默认即打开),输出字符，并刷新buffer。==

```sql
set serveroutput on --将output 服务打开
set serveroutput off --将output 服务关闭

==类型[^1]：数据库字段类型，例如 varchar2/number/char==

```

##### 1.4.1.3 有参函数定义

```sql
create or replace function 函数名(字段 类型)
return 类型
is
  xxx 表.字段%type;
begin
  select aaa into xxx from 表 where 查询字段=字段
  return xxx;
end;  
-- 调用
select 函数名(字段值) from dual;  
```

##### 1.4.1.4 输出参数函数定义

```sql
create or replace function 函数名(字段1 类型,字段2 OUT 类型)
return 类型
is 
  xxx 表.字段%type;
begin 
  select aaa,bbb into xxx,ddd from 表 where 查询字段=字段1;
  return xxx;
end;
```

##### 1.4.1.5 输入输出参数函数定义

```sql
create or replace function 函数名(字段1 类型,字段2 IN OUT 类型)
return 类型
is 
  xxx 表.字段%type;
begin 
  update 表 set 修改字段=修改字段+字段2 where 查询字段=字段1
    returning aaa,bbb into xxx,ddd;
  return xxx;
end;  
```

##### 1.4.1.6 结果缓存函数定义

```sql
create or replace function 函数名(字段 类型)
return 类型
result_cache relies_on(表)
is 
  xxx 表.字段%type;
begin 
  select aaa into xxx from 表 where 查询字段=字段
  return xxx;
end;  
```

##### 1.4.1.7 异常处理函数定义

```sql
create or replace function 函数名(字段 类型)
return 类型
is 
  xxx 表.字段%type;
begin 
  select aaa into xxx from 表 where 查询字段=字段
  return xxx;
exception
  when no_data_found then raise_application_error(-20000,字段||'不存在');  
end;  
```

##### 1.4.1.8 返回行数据类型函数定义

```sql
create or replace function 函数名(字段 类型)
return 表%rowType
is 
  xxx 表%rowType;
begin 
  select * into xxx from 表 where 查询字段=字段
  return xxx;
exception
  when no_data_found then raise_application_error(-20000,字段||'不存在');  
end;  
```

##### 1.4.1.9 返回集合类型函数定义

```sql
create or replace function 函数名(字段 类型)
return 表
is 
  xxx 表;
begin 
  select aaa,bbb collect into xxx from 表 where 查询字段=字段
  return xxx;
exception
  when no_data_found then raise_application_error(-20000,字段||'不存在');  
end;  
```

##### 1.4.1.10 返回结果集函数定义

###### 1.4.1.10.1 游标结果集

```sql
create or replace function 函数名(字段 类型)
return 表
is 
  xxx 表;
begin 
  open 表 for
  select aaa,bbb from 表 where 查询字段=字段
  return xxx;
end 函数名;  
```

###### 1.4.1.10.2 table结果集

```sql
-- 定义一个行类型
create or replace type xxxRowType as object(字段1 类型, 字段2 类型)

-- 定义一个表类型
create or replace type vvvTabType as table of xxxRowType

-- 创建函数
create or replace function 函数名(字段 类型)
  return vvvTabType
  is aaaRow xxxRowType; -- 定义单行
     bbbTab vvvTabType = :vvvTabType(); --定义返回结果，并初始化
  begin 
    for temp in (select ccc,ddd,eee from 表 where 查询字段=字段)
    loop
      aaaRow := vvvTabType(temp.ccc,temp.ddd);
      bbbTab.extend;
      bbbTab(bbbTab.count) := aaaRow;
    end loop;
  return (bbbTab) ;
end 函数名;
-- 调用
select * from table(bbbTab('ggg'));
```

###### 1.4.1.10.3 管道结果集

```sql
-- 定义一个行类型
create or replace type xxxRowType as object(字段1 类型, 字段2 类型)

-- 定义一个表类型
create or replace type vvvTabType ad table of xxxRowType
-- 创建函数
create or replace function 函数名(字段 类型)
  return vvvTabType pipelined
  is aaaRow xxxRowType; -- 定义单行
  begin 
    for temp in (select ccc,ddd,eee from 表 where 查询字段=字段)
    loop
      aaaRow := vvvTabType(temp.ccc,temp.ddd);
      pipe row(aaaRow);
    end loop;
  return;
end 函数名;
-- 调用
select * from table(bbbTab('ggg'));
```

#### 1.4.2 mysql

##### 1.4.2.1 基本语法

参考链接：https://zhuanlan.zhihu.com/p/128744140

```sql
create 
  function 函数名
  ([p1,p2...pn])
returns datatype
is|as
--声明部分 
begin
--程序块
end
```

- 1、`function` 是创建函数的关键字,同MySQL内置函数一样，大小写不敏感。
- 2、p1,p2...pn是函数的入参，Oracle创建的函数也可以不需要入参。
- 3、returns datatype:是函数的返回值的类型,注意是**<font color="red">returns</font>** 不是return
  