# 一些开发技巧

## 1.如何查看mybatis-plus拼接好的sql

ClientPreparedStatement的 ResultSetInternalMethods  debug this参数即可，注意ClientPreparedStatement类为源码类，idea搜索的时候双击shift,右上角includ no pj item打勾即可。

## 2.如何铺平双次循环对比的代码，使得两集合对比不再嵌套循环

下面给一段JS，原理就是将第一个数组中的某项取出，放到对象中（key 为该数组某项的某字段，value为某项），然后第二次循环用另一个数组去对比，代码如下。java同理，只不过对象换成hashmap即可。

```javascript
  let cd = [];
        if (data) {
          this.checkData = [];

          let midData = {};

          for (let i = 0; i < this.tableData.length; i++) {
            midData[this.tableData[i].idcard] = i;
          }

          for (let i = 0; i < data.length; i++) {
            this.$refs.popTable.toggleRowSelection([
              { row: this.tableData[midData[data[i].idCard]] },
            ]);

            cd.push(this.tableData[midData[data[i].idCard]]);
          }
        }

        this.checkData = cd;

        this.$parent.addPerson(cd);
```

## 3.POWERDESIGN逆向生成如何插入备注

```
https://blog.csdn.net/w13511069150/article/details/79621256




    Option   Explicit     
    ValidationMode   =   True     
    InteractiveMode   =   im_Batch  
    Dim blankStr  
    blankStr   =   Space(1)  
    Dim   mdl   '   the   current   model    
        
    '   get   the   current   active   model     
    Set   mdl   =   ActiveModel     
    If   (mdl   Is   Nothing)   Then     
          MsgBox   "There   is   no   current   Model "     
    ElseIf   Not   mdl.IsKindOf(PdPDM.cls_Model)   Then     
          MsgBox   "The   current   model   is   not   an   Physical   Data   model. "     
    Else     
          ProcessFolder   mdl     
    End   If    
        
    Private   sub   ProcessFolder(folder)     
    On Error Resume Next    
          Dim   Tab   'running     table     
          for   each   Tab   in   folder.tables     
                if   not   tab.isShortcut   then     
                      tab.name   =   tab.comment    
                      Dim   col   '   running   column     
                      for   each   col   in   tab.columns     
                      if col.comment = "" or replace(col.comment," ", "")="" Then  
                            col.name = blankStr  
                            blankStr = blankStr & Space(1)  
                      else    
                            col.name = col.comment     
                      end if    
                      next     
                end   if     
          next    
        
          Dim   view   'running   view     
          for   each   view   in   folder.Views     
                if   not   view.isShortcut   then     
                      view.name   =   view.comment     
                end   if     
          next    
        
          '   go   into   the   sub-packages     
          Dim   f   '   running   folder     
          For   Each   f   In   folder.Packages     
                if   not   f.IsShortcut   then     
                      ProcessFolder   f     
                end   if     
          Next     
    end   sub    
```

## 4.关于SQL如何在字段左右补0，使得String对比的时候能够按照位数来

左边补0
update `csgm_population` a set a.house_number =  LPAD(a.house_number,5,0)    

右边 RPAD

## 5.关于mysql批量更新

批量更新需要jdbc链接参数加上&allowMultiQueries=true

## 6.关于mybatis-plus封装批量插入及更新的方式

```Java
import com.baomidou.mybatisplus.core.injector.AbstractMethod;
import com.baomidou.mybatisplus.core.injector.DefaultSqlInjector;
import com.baomidou.mybatisplus.core.metadata.TableInfo;

import java.util.List;

public class CustomizedSqlInjector extends DefaultSqlInjector {

    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass,TableInfo tableInfo) {
        List<AbstractMethod> methodList = super.getMethodList(mapperClass,tableInfo);
        methodList.add(new InsertBatchMethod());
        methodList.add(new UpdateBatchMethod());
        return methodList;
    }
}
 



// insert 这样会返回ID

import cn.hutool.core.util.ObjectUtil;
import cn.hutool.core.util.StrUtil;
import com.baomidou.mybatisplus.annotation.FieldStrategy;
import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.core.enums.SqlMethod;
import com.baomidou.mybatisplus.core.injector.AbstractMethod;
import com.baomidou.mybatisplus.core.metadata.TableInfo;
import com.baomidou.mybatisplus.core.toolkit.StringUtils;
import org.apache.ibatis.executor.keygen.Jdbc3KeyGenerator;
import org.apache.ibatis.executor.keygen.KeyGenerator;
import org.apache.ibatis.executor.keygen.NoKeyGenerator;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 批量插入 方法体
 *
 * @author aoyang
 */
public class InsertBatchMethod extends AbstractMethod {

    private final Logger logger = LoggerFactory.getLogger(InsertBatchMethod.class);

    //keyProperty="id"

    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {

        KeyGenerator keyGenerator = NoKeyGenerator.INSTANCE;
        String keyProperty = null;
        String keyColumn = null;
        if (StringUtils.isNotBlank(tableInfo.getKeyProperty())) {
            if (tableInfo.getIdType() == IdType.AUTO) {
                /* 自增主键 */
                keyGenerator = Jdbc3KeyGenerator.INSTANCE;
                keyProperty = tableInfo.getKeyProperty();
                keyColumn = tableInfo.getKeyColumn();
            }
        }
        final String sql = "<script>insert into %s %s values %s</script>";
        final String fieldSql = prepareFieldSql(tableInfo);
        final String valueSql = prepareValuesSql(tableInfo);
        final String sqlResult = String.format(sql, tableInfo.getTableName(), fieldSql, valueSql);

        //log.debug("sqlResult----->{}", sqlResult);
        SqlSource sqlSource = languageDriver.createSqlSource(configuration, sqlResult, modelClass);
        return this.addInsertMappedStatement(mapperClass, modelClass, "insertBatch", sqlSource, keyGenerator, keyProperty, keyColumn);
    }

    private String prepareFieldSql(TableInfo tableInfo) {
        StringBuilder fieldSql = new StringBuilder();
        if (StrUtil.isNotEmpty(tableInfo.getKeyColumn()))
            fieldSql.append(tableInfo.getKeyColumn()).append(",");
        tableInfo.getFieldList().forEach(x -> {
            if (!ObjectUtil.equals(FieldStrategy.NEVER, x.getInsertStrategy())) {
                fieldSql.append(x.getColumn()).append(",");
            }
        });
        fieldSql.delete(fieldSql.length() - 1, fieldSql.length());
        fieldSql.insert(0, "(");
        fieldSql.append(")");
        return fieldSql.toString();
    }

    private String prepareValuesSql(TableInfo tableInfo) {
        final StringBuilder valueSql = new StringBuilder();
        valueSql.append("<foreach collection=\"collection\" item=\"item\" index=\"index\" open=\"(\" separator=\"),(\" close=\")\">");
        if (StrUtil.isNotEmpty(tableInfo.getKeyColumn()))
            valueSql.append("#{item.").append(tableInfo.getKeyProperty()).append("},");
        tableInfo.getFieldList().forEach(x -> {
            if (!ObjectUtil.equals(FieldStrategy.NEVER, x.getInsertStrategy()))
                valueSql.append("#{item.").append(x.getProperty()).append("},");
        });
        valueSql.delete(valueSql.length() - 1, valueSql.length());
        valueSql.append("</foreach>");
        return valueSql.toString();
    }
}






import org.mybatis.spring.annotation.MapperScan;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@MapperScan("yushang.web.**.mapper")
public class MyBatisPlusConfig {
    /**
     * 批量新增|修改
     */
    @Bean
    public CustomizedSqlInjector customizedSqlInjector() {
        return new CustomizedSqlInjector();
    }
}
 




import com.baomidou.mybatisplus.core.injector.AbstractMethod;
import com.baomidou.mybatisplus.core.metadata.TableInfo;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 批量更新 方法体
 * 条件为主键，选择性更新
 *
 * @author aoyang
 */
public class UpdateBatchMethod extends AbstractMethod {

    private final Logger logger = LoggerFactory.getLogger(UpdateBatchMethod.class);

    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {

        String sql = "<script>\n<foreach collection=\"collection\" item=\"item\" separator=\";\">\nupdate %s %s where %s=#{%s} %s\n</foreach>\n</script>";

        String additional = tableInfo.isWithVersion() ? tableInfo.getVersionFieldInfo().getVersionOli("item", "item.") : "" + tableInfo.getLogicDeleteSql(true, true);

        String setSql = sqlSet(tableInfo.isWithLogicDelete(), false, tableInfo, false, "item", "item.");
        String sqlResult = String.format(sql, tableInfo.getTableName(), setSql, tableInfo.getKeyColumn(), "item." + tableInfo.getKeyProperty(), additional);
        //log.debug("sqlResult----->{}", sqlResult);
        SqlSource sqlSource = languageDriver.createSqlSource(configuration, sqlResult, modelClass);
        // 第三个参数必须和RootMapper的自定义方法名一致
        return this.addUpdateMappedStatement(mapperClass, modelClass, "updateBatch", sqlSource);
    }
}
```

## 7.mybatis-plus自动注入，注意不可照抄，注意字段名

```java
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.binding.MapperMethod.ParamMap;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.plugin.*;
import org.springframework.stereotype.Component;
import yushang.common.utils.SecurityUtils;
import java.lang.reflect.Field;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Properties;

/**
 * mybatis拦截器，自动注入创建人、创建时间、删除标记、修改人、修改时间
 * 1、update
 * 	//该方法会在所有的INSERT、UPDATE、DELETE执行时被调用，因此如果想要拦截这3类操作，可以拦截该方法。接口方法对应的签名如下：
 *  @Signature(
 * type= Executor.class,method = "update",args = {MappedStatement.class, Object.class})
 *
 * 2、query
 * 	// 该方法会在所有SELECT查询方法执行时被调用。通过这个接口参数可以获取很多有用的信息，因此这是最常被拦截的一个方法。使用该方法需要注意的是，虽然接口中还有一个参数更多的同名接口，但由于MyBatis的设计原因，这个参数多的接口不能被拦截。接口方法对应的签名如下：
 * @Signature(
 * type= Executor.class,method = "query",args = {MappedStatement.class, Object.class,RowBounds.class, ResultHandler.class})
 *
 * 3、flushStatements
 * 	//该方法只在通过SqlSession方法调用flushStatements方法或执行的接口方法中带有@Flush注解时才被调用，接口方法对应的签名如下：
 *        @Signature(
 * type= Executor.class,method = "flushStatements",args = {})
 *
 * 4、commit
 * 	//该方法只在通过SqlSession方法调用commit方法时才被调用，接口方法对应的签名如下：
 *    @Signature(
 * type= Executor.class,method = "commit",args = {boolean.class})
 *
 * 5、rollback
 * 	//该方法只在通过SqlSession方法调用rollback方法时才被调用，接口方法对应的签名如下：
 *    @Signature(
 * type= Executor.class,method = "rollback",args = {boolean.class})
 *
 * 6、getTransaction
 * 	//该方法只在通过SqlSession方法获取数据库连接时才被调用，接口方法对应的签名如下：
 *    @Signature(
 * type= Executor.class,method = "getTransaction",args = {})
 *
 * 7、close
 * 	//该方法只在延迟加载获取新的Executor后才会被执行，接口方法对应的签名如下：
 *    @Signature(
 * type= Executor.class,method = "close",args = {boolean.class})
 *
 * 8、isClosed
 * 	//方法只在延迟加载执行查询方法前被执行，接口方法对应的签名如下：
 *    @Signature(
 * type= Executor.class,method = "isClosed",args = {})
 *
 *
 * 注意，由于参数关系，本类对批量操作无效
 */
@Slf4j
@Component
@Intercepts({
        @Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class}),
}
)
public class MybatisInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {

        MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];

        SqlCommandType sqlCommandType = mappedStatement.getSqlCommandType();

        Object parameter = invocation.getArgs()[1];

        if (parameter == null) {
            return invocation.proceed();
        }
        if (SqlCommandType.INSERT == sqlCommandType) {

            Field[] fields = getAllFields(parameter);

            for (Field field : fields) {
                try {
                    if ("createBy".equals(field.getName())) {
                        field.setAccessible(true);
                        Object local_createBy = field.get(parameter);
                        field.setAccessible(false);
                        if (local_createBy == null || local_createBy.equals("")) {
                            field.setAccessible(true);
                            String username = "";
                            try {
                                username = SecurityUtils.getUsername();
                            } catch (Exception e) {
                                log.warn("没有获取到用户,创建者将插入空值");
                            }
                            field.set(parameter, username);
                            field.setAccessible(false);
                        }
                    }
                    // 注入创建时间
                    if ("createTime".equals(field.getName())) {
                        field.setAccessible(true);
                        Object local_createDate = field.get(parameter);
                        field.setAccessible(false);
                        if (local_createDate == null || local_createDate.equals("")) {
                            if(field.get(parameter)==null){
                                field.setAccessible(true);
                                field.set(parameter, LocalDateTime.now());
                                field.setAccessible(false);
                            }
                        }
                    }

                    // 注入删除标记
//                    if ("deleted".equals(field.getName())) {
//                        field.setAccessible(true);
//                        Object local_createDate = field.get(parameter);
//                        field.setAccessible(false);
//                        if (local_createDate == null || local_createDate.equals("")) {
//                            field.setAccessible(true);
//                            field.set(parameter, 0);
//                            field.setAccessible(false);
//                        }
//                    }

                } catch (Exception e) {
                    log.warn("SQL自动注入异常");
                }
            }
        }
        if (SqlCommandType.UPDATE == sqlCommandType) {

            Field[] fields = null;
            if (parameter instanceof ParamMap) {
                ParamMap<?> p = (ParamMap<?>) parameter;
                //update-begin-author:scott date:20190729 for:批量更新报错issues/IZA3Q--
                if (p.containsKey("et")) {
                    parameter = p.get("et");
                } else if(p.containsKey("param1")) {
                    parameter = p.get("param1");
                }
                //update-end-author:scott date:20190729 for:批量更新报错issues/IZA3Q-

                //update-begin-author:scott date:20190729 for:更新指定字段时报错 issues/#516-
                if (parameter == null) {
                    return invocation.proceed();
                }
                //update-end-author:scott date:20190729 for:更新指定字段时报错 issues/#516-
                fields = getAllFields(parameter);
            } else {
                fields = getAllFields(parameter);
            }
            for (Field field : fields) {

                try {
                    if ("updateBy".equals(field.getName())) {
                        field.setAccessible(true);
                        String username = "";
                        try {
                            username = SecurityUtils.getUsername();
                        } catch (Exception e) {
                            log.warn("没有获取到用户,修改者将插入空值");
                        }
                        field.set(parameter, username);

                        field.setAccessible(false);
                    }
                    if ("updateTime".equals(field.getName())) {
                        if(field.get(parameter)==null){
                            field.setAccessible(true);
                            field.set(parameter, LocalDateTime.now());
                            field.setAccessible(false);
                        }

                    }

                } catch (Exception e) {
                    log.warn("SQL自动注入异常");
                }
            }
        }

        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }


    /**
     * 获取类的所有属性，包括父类
     *
     * @param object
     * @return
     */
    private Field[] getAllFields(Object object) {
        Class<?> clazz = object.getClass();
        List<Field> fieldList = new ArrayList<>();
        while (clazz != null) {
            fieldList.addAll(new ArrayList<>(Arrays.asList(clazz.getDeclaredFields())));
            clazz = clazz.getSuperclass();
        }
        Field[] fields = new Field[fieldList.size()];
        fieldList.toArray(fields);
        return fields;
    }

}
```

