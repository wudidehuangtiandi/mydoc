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

## 8.关于乌班图使用nginx的简单密码和centOS的区别

其它和centos相同，除了命令不同

```
apt-get install httpd-tools

htpasswd -c -b passwd 账号 密码
```

## 9.关于springwebsocket的封装及基于springwebsocket和redis发布订阅实现的分布式websocket的demo

首先是websocket，可在多线程环境下使用

```java
package com.yushang.dsp.a_kj.config.websocket;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.websocket.Session;
import java.io.IOException;
import java.util.Collection;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/**
 * websocket 客户端用户集
 * 
 * @author ruoyi
 */
public class WebSocketUsers
{
    /**
     * WebSocketUsers 日志控制器
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(WebSocketUsers.class);

    /**
     * 用户集
     */
    private static Map<String, Session> USERS = new ConcurrentHashMap<String, Session>();

    /**
     * 存储用户
     *
     * @param key 唯一键
     * @param session 用户信息
     */
    public synchronized static void put(String key, Session session)
    {
        USERS.put(key, session);
    }

    /**
     * 移除用户
     *
     * @param session 用户信息
     *
     * @return 移除结果
     */
    public synchronized static boolean remove(Session session)
    {
        String key = null;
        boolean flag = USERS.containsValue(session);
        if (flag)
        {
            Set<Map.Entry<String, Session>> entries = USERS.entrySet();
            for (Map.Entry<String, Session> entry : entries)
            {
                Session value = entry.getValue();
                if (value.equals(session))
                {
                    key = entry.getKey();
                    break;
                }
            }
        }
        else
        {
            return true;
        }
        return remove(key);
    }

    /**
     * 移出用户
     *
     * @param key 键
     */
    public synchronized static boolean remove(String key)
    {
        LOGGER.info("\n 正在移出用户 - {}", key);
        Session remove = USERS.remove(key);
        if (remove != null)
        {
            boolean containsValue = USERS.containsValue(remove);
            LOGGER.info("\n 移出结果 - {}", containsValue ? "失败" : "成功");
            return containsValue;
        }
        else
        {
            LOGGER.info("未查询到用户");
            return true;
        }
    }

    /**
     * 获取在线用户列表
     *
     * @return 返回用户集合
     */
    public synchronized static Map<String, Session> getUsers()
    {
        return USERS;
    }

    /**
     * 群发消息文本消息
     *
     * @param message 消息内容
     */
    public synchronized static void sendMessageToUsersByText(String message)
    {
        Collection<Session> values = USERS.values();
        for (Session value : values)
        {
            sendMessageToUserByText(value, message);
        }
    }

    /**
     * 发送文本消息,同步确保多线程下的稳定性
     *
     * @param session 自己的用户名
     * @param message 消息内容
     */
    public synchronized static void sendMessageToUserByText(Session session, String message)
    {
        if (session != null)
        {
            try
            {
                session.getBasicRemote().sendText(message);
            }
            catch (IOException e)
            {
                LOGGER.error("\n[发送消息异常]", e);
            }
        }
        else
        {
            LOGGER.info("\n[你已离线]");
        }
    }
}

package com.yushang.dsp.a_kj.config.websocket;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

/**
 * websocket 配置
 * 
 * @author ruoyi
 */
@Configuration
public class WebSocketConfig
{
    @Bean
    public ServerEndpointExporter serverEndpointExporter()
    {
        return new ServerEndpointExporter();
    }
}

package com.yushang.dsp.a_kj.config.websocket;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.websocket.*;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;
import java.util.concurrent.Semaphore;

/**
 * websocket 消息处理
 * 
 * @author ruoyi
 */
@Component
@ServerEndpoint("/websocket/message/{machineCode}")
public class WebSocketServer
{
    /**
     * WebSocketServer 日志控制器
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(WebSocketServer.class);

    /**
     * 默认最多允许同时在线人数100
     */
    public static int socketMaxOnlineCount = 200;

    private static Semaphore socketSemaphore = new Semaphore(socketMaxOnlineCount);

    /**
     * 连接建立成功调用的方法
     */
    @OnOpen
    public void onOpen(Session session,@PathParam("machineCode")String  machineCode) throws Exception
    {
        boolean semaphoreFlag = false;
        // 尝试获取信号量
        semaphoreFlag = SemaphoreUtils.tryAcquire(socketSemaphore);
        if (!semaphoreFlag)
        {
            // 未获取到信号量
            LOGGER.error("\n 当前在线人数超过限制数- {}", socketMaxOnlineCount);
            WebSocketUsers.sendMessageToUserByText(session, "当前在线人数超过限制数：" + socketMaxOnlineCount);
            session.close();
        }
        else
        {
            // 添加用户
            WebSocketUsers.put(machineCode, session);
            LOGGER.info("\n 建立连接 - {}",machineCode+":"+session);
            LOGGER.info("\n 当前人数 - {}", WebSocketUsers.getUsers().size());
            WebSocketUsers.sendMessageToUserByText(session, "连接成功");
        }
    }

    /**
     * 连接关闭时处理
     */
    @OnClose
    public void onClose(Session session,@PathParam("machineCode")String  machineCode)
    {
        LOGGER.info("\n 关闭连接 - {}", session);
        // 移除用户
        WebSocketUsers.remove(machineCode);
        // 获取到信号量则需释放
        SemaphoreUtils.release(socketSemaphore);
    }

    /**
     * 抛出异常时处理
     */
    @OnError
    public void onError(Session session, Throwable exception,@PathParam("machineCode")String  machineCode) throws Exception
    {
        if (session.isOpen())
        {
            // 关闭连接
            session.close();
        }
        String sessionId = machineCode;
        LOGGER.info("\n 连接异常 - {}", sessionId);
        LOGGER.info("\n 异常信息 - {}", exception);
        // 移出用户
        WebSocketUsers.remove(sessionId);
        // 获取到信号量则需释放
        SemaphoreUtils.release(socketSemaphore);
    }

    /**
     * 服务器接收到客户端消息时调用的方法
     */
    @OnMessage
    public void onMessage(String message, Session session)
    {
        //String msg = message.replace("你", "我").replace("吗", "");
        LOGGER.info("收到信息:{} ",message);
        WebSocketUsers.sendMessageToUserByText(session, "收到信息:"+message);
    }
}

package com.yushang.dsp.a_kj.config.websocket;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.websocket.Session;
import java.io.IOException;
import java.util.Collection;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/**
 * websocket 客户端用户集
 * 
 * @author ruoyi
 */
public class WebSocketUsers
{
    /**
     * WebSocketUsers 日志控制器
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(WebSocketUsers.class);

    /**
     * 用户集
     */
    private static Map<String, Session> USERS = new ConcurrentHashMap<String, Session>();

    /**
     * 存储用户
     *
     * @param key 唯一键
     * @param session 用户信息
     */
    public synchronized static void put(String key, Session session)
    {
        USERS.put(key, session);
    }

    /**
     * 移除用户
     *
     * @param session 用户信息
     *
     * @return 移除结果
     */
    public synchronized static boolean remove(Session session)
    {
        String key = null;
        boolean flag = USERS.containsValue(session);
        if (flag)
        {
            Set<Map.Entry<String, Session>> entries = USERS.entrySet();
            for (Map.Entry<String, Session> entry : entries)
            {
                Session value = entry.getValue();
                if (value.equals(session))
                {
                    key = entry.getKey();
                    break;
                }
            }
        }
        else
        {
            return true;
        }
        return remove(key);
    }

    /**
     * 移出用户
     *
     * @param key 键
     */
    public synchronized static boolean remove(String key)
    {
        LOGGER.info("\n 正在移出用户 - {}", key);
        Session remove = USERS.remove(key);
        if (remove != null)
        {
            boolean containsValue = USERS.containsValue(remove);
            LOGGER.info("\n 移出结果 - {}", containsValue ? "失败" : "成功");
            return containsValue;
        }
        else
        {
            LOGGER.info("未查询到用户");
            return true;
        }
    }

    /**
     * 获取在线用户列表
     *
     * @return 返回用户集合
     */
    public synchronized static Map<String, Session> getUsers()
    {
        return USERS;
    }

    /**
     * 群发消息文本消息
     *
     * @param message 消息内容
     */
    public synchronized static void sendMessageToUsersByText(String message)
    {
        Collection<Session> values = USERS.values();
        for (Session value : values)
        {
            sendMessageToUserByText(value, message);
        }
    }

    /**
     * 发送文本消息,同步确保多线程下的稳定性
     *
     * @param session 自己的用户名
     * @param message 消息内容
     */
    public synchronized static void sendMessageToUserByText(Session session, String message)
    {
        if (session != null)
        {
            try
            {
                session.getBasicRemote().sendText(message);
            }
            catch (IOException e)
            {
                LOGGER.error("\n[发送消息异常]", e);
            }
        }
        else
        {
            LOGGER.info("\n[你已离线]");
        }
    }
}

```

下面是分布式环境,共六个类

```java
package com.yushang.dsp.a_kj.config.websocket;

import com.yushang.dsp.a_kj.config.redis.RedisConfig;
import com.yushang.dsp.a_kj.domain.constant.Constants;
import com.yushang.dsp.a_kj.utils.SpringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

@Component
public class BisPublisher {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Value("${spring.redis.channel.msgToAll}")
    private String msgToAll;

    /**
     * 分布式下的操作,带上机器码,广播消息
     */
    public synchronized void sendMessageToUserByTextPublish(String machineCode, String message) {

        String msg = message.replace("\"", "");
        stringRedisTemplate.convertAndSend(msgToAll, machineCode + Constants.LZF + msg);
    }
}


package com.yushang.dsp.a_kj.config.websocket;

import com.yushang.dsp.a_kj.domain.constant.Constants;
import com.yushang.dsp.a_kj.utils.StringUtils;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.connection.Message;
import org.springframework.data.redis.connection.MessageListener;
import org.springframework.stereotype.Component;

import javax.websocket.Session;
import java.io.IOException;
import java.util.Map;

/**
 * redis订阅消息消费者
 */
@Component
@Slf4j
public class BisSubscriber implements MessageListener {

    @Autowired
    WebSocketServer webSocketServer;

    /**
     * 统一消息管理
     * @param message
     * @param pattern
     */
    @Override
    public void onMessage(Message message, byte[] pattern) {

        Map<String, Session> users = WebSocketUsers.getUsers();

        String body = new String(message.getBody());
        if(StringUtils.isNotEmpty(body)&&body.contains(Constants.LZF)){
            String[] value = body.split(Constants.LZF);
            String mactionId = value[0];
            String msg = value[1];

            //判断是否是当前服务的对象
            Session session = users.get(mactionId);
            if(session!=null&&session.isOpen()){
                //先处理特殊事件

                //关闭
                if(msg.contains(Constants.QT)){
                    log.info("\n 关闭连接 - {}", session);
                    // 移除用户
                    WebSocketUsers.remove(mactionId);
                    // 获取到信号量则需释放
                    webSocketServer.release();
                }
                //异常
                if(msg.contains(Constants.EX)){
                    if (session != null) {
                        if (session.isOpen()) {
                            // 关闭连接
                            try {
                                session.close();
                            } catch (IOException e) {
                                log.error("异常关闭失败");
                                e.printStackTrace();
                            }
                        }
                        log.info("\n 连接异常 - {}", mactionId);

                        // 移出用户
                        WebSocketUsers.remove(mactionId);
                        // 获取到信号量则需释放
                        webSocketServer.release();
                    }
                }
                //正常的消息处理
                WebSocketUsers.sendMessageToUserByText(session,msg);
            }
        }
    }
 
}

package com.yushang.dsp.a_kj.config.websocket;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.Semaphore;

/**
 * 信号量相关处理
 * 
 * @author ruoyi
 */
public class SemaphoreUtils
{
    /**
     * SemaphoreUtils 日志控制器
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(SemaphoreUtils.class);

    /**
     * 获取信号量
     * 
     * @param semaphore
     * @return
     */
    public static boolean tryAcquire(Semaphore semaphore)
    {
        boolean flag = false;

        try
        {
            flag = semaphore.tryAcquire();
        }
        catch (Exception e)
        {
            LOGGER.error("获取信号量异常", e);
        }

        return flag;
    }

    /**
     * 释放信号量
     * 
     * @param semaphore
     */
    public static void release(Semaphore semaphore)
    {

        try
        {
            semaphore.release();
        }
        catch (Exception e)
        {
            LOGGER.error("释放信号量异常", e);
        }
    }
}

package com.yushang.dsp.a_kj.config.websocket;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

/**
 * websocket 配置
 * 
 * @author ruoyi
 */
@Configuration
public class WebSocketConfig
{
    @Bean
    public ServerEndpointExporter serverEndpointExporter()
    {
        return new ServerEndpointExporter();
    }
}


package com.yushang.dsp.a_kj.config.websocket;

import com.yushang.dsp.a_kj.config.redis.RedisConfig;
import com.yushang.dsp.a_kj.domain.constant.Constants;
import com.yushang.dsp.a_kj.utils.SpringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.connection.Message;
import org.springframework.data.redis.connection.MessageListener;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

import javax.websocket.*;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;
import java.util.Map;
import java.util.concurrent.Semaphore;

/**
 * websocket 消息处理
 *
 * @author ruoyi
 */
@Component
@ServerEndpoint("/websocket/message/{machineCode}")
public class WebSocketServer {


    /**
     * WebSocketServer 日志控制器
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(WebSocketServer.class);

    /**
     * 默认最多允许同时在线人数2000
     */
    public static int socketMaxOnlineCount = 2000;

    private static Semaphore socketSemaphore = new Semaphore(socketMaxOnlineCount);

    /**
     * 连接建立成功调用的方法
     */
    @OnOpen
    public void onOpen(Session session, @PathParam("machineCode") String machineCode) throws Exception {
        boolean semaphoreFlag = false;
        // 尝试获取信号量
        semaphoreFlag = SemaphoreUtils.tryAcquire(socketSemaphore);
        if (!semaphoreFlag) {
            // 未获取到信号量
            LOGGER.error("\n 当前在线人数超过限制数- {}", socketMaxOnlineCount);
            WebSocketUsers.sendMessageToUserByText(session, "当前在线人数超过限制数：" + socketMaxOnlineCount);
            session.close();
        } else {
            // 添加用户
            WebSocketUsers.put(machineCode, session);
            getPublisher().sendMessageToUserByTextPublish(machineCode, "用户已链接");
            LOGGER.info("\n 建立连接 - {}", machineCode + ":" + session);
            LOGGER.info("\n 当前人数 - {}", WebSocketUsers.getUsers().size());
            WebSocketUsers.sendMessageToUserByText(session, "连接成功");
        }
    }

    /**
     * 连接关闭时处理
     */
    @OnClose
    public void onClose(@PathParam("machineCode") String machineCode) {
        //查询本地是否有这个用户
        Map<String, Session> users = WebSocketUsers.getUsers();
        Session session = users.get(machineCode);
        if (session != null) {
            LOGGER.info("\n 关闭连接 - {}", session);
            // 移除用户
            WebSocketUsers.remove(machineCode);
            // 获取到信号量则需释放
            SemaphoreUtils.release(socketSemaphore);
        }else {
            //推送广播让对应的机器处理
            getPublisher().sendMessageToUserByTextPublish(machineCode, Constants.QT);
        }
    }

    /**
     * 抛出异常时处理
     */
    @OnError
    public void onError( Throwable exception, @PathParam("machineCode") String machineCode) throws Exception {
        Map<String, Session> users = WebSocketUsers.getUsers();
        Session session = users.get(machineCode);
        if (session != null) {
            if (session.isOpen()) {
                // 关闭连接
                session.close();
            }
            String sessionId = machineCode;
            LOGGER.info("\n 连接异常 - {}", sessionId);
            LOGGER.info("\n 异常信息 - {}", exception);
            // 移出用户
            WebSocketUsers.remove(sessionId);
            // 获取到信号量则需释放
            SemaphoreUtils.release(socketSemaphore);
        }else {
            //推送广播让对应的机器处理
            getPublisher().sendMessageToUserByTextPublish(machineCode, Constants.EX);
        }
    }

    /**
     * 服务器接收到客户端消息时调用的方法
     */
    @OnMessage
    public void onMessage(String message, @PathParam("machineCode") String machineCode) {
        LOGGER.info("收到消息:"+message);
        //广播消息
        getPublisher().sendMessageToUserByTextPublish(machineCode,message);
    }

    /**
     * @return
     */
    public BisPublisher getPublisher(){
       return SpringUtils.getBean(BisPublisher.class);
    }

    /**
     * 释放
     */
    public void release(){
        SemaphoreUtils.release(socketSemaphore);
    }

}

package com.yushang.dsp.a_kj.config.websocket;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import javax.websocket.Session;
import java.io.IOException;
import java.util.Collection;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/**
 * websocket 客户端用户集
 * 
 * @author ruoyi
 */

public class WebSocketUsers
{

    /**
     * WebSocketUsers 日志控制器
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(WebSocketUsers.class);

    /**
     * 用户集
     */
    private static Map<String, Session> USERS = new ConcurrentHashMap<String, Session>();

    /**
     * 存储用户
     *
     * @param key 唯一键
     * @param session 用户信息
     */
    public synchronized static void put(String key, Session session)
    {
        USERS.put(key, session);
    }

    /**
     * 移除用户
     *
     * @param session 用户信息
     *
     * @return 移除结果
     */
    public synchronized static boolean remove(Session session)
    {
        String key = null;
        boolean flag = USERS.containsValue(session);
        if (flag)
        {
            Set<Map.Entry<String, Session>> entries = USERS.entrySet();
            for (Map.Entry<String, Session> entry : entries)
            {
                Session value = entry.getValue();
                if (value.equals(session))
                {
                    key = entry.getKey();
                    break;
                }
            }
        }
        else
        {
            return true;
        }
        return remove(key);
    }

    /**
     * 移出用户
     *
     * @param key 键
     */
    public synchronized static boolean remove(String key)
    {
        LOGGER.info("\n 正在移出用户 - {}", key);
        Session remove = USERS.remove(key);
        if (remove != null)
        {
            boolean containsValue = USERS.containsValue(remove);
            LOGGER.info("\n 移出结果 - {}", containsValue ? "失败" : "成功");
            return containsValue;
        }
        else
        {
            LOGGER.info("未查询到用户");
            return true;
        }
    }

    /**
     * 获取在线用户列表
     *
     * @return 返回用户集合
     */
    public synchronized static Map<String, Session> getUsers()
    {
        return USERS;
    }

    /**
     * 群发消息文本消息
     *
     * @param message 消息内容
     */
    public synchronized static void sendMessageToUsersByText(String message)
    {
        Collection<Session> values = USERS.values();
        for (Session value : values)
        {
            sendMessageToUserByText(value, message);
        }
    }

    /**
     * 发送文本消息,同步确保多线程下的稳定性
     *
     * @param session 自己的用户名
     * @param message 消息内容
     */
    public synchronized static void sendMessageToUserByText(Session session, String message)
    {
        if (session != null)
        {
            try
            {
                session.getBasicRemote().sendText(message);
            }
            catch (IOException e)
            {
                LOGGER.error("\n[发送消息异常]", e);
            }
        }
        else
        {
            LOGGER.info("\n[你已离线]");
        }
    }
}

```

## 10.关于restTemplate实现统一的拦截器配置及消息转换

共四个类,包括如何设置代理及如何改变请求头等。

```java
package com.yushang.dsp.a_kj.config.resttemplate;

import org.springframework.http.MediaType;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;

import java.util.ArrayList;
import java.util.List;

/**
 * @ClassName : MsgConverter
 * @Description :
 * @Author : GC
 * @Date: 2020-10-24 14:14
 */
public class MsgConverter extends MappingJackson2HttpMessageConverter {
    public MsgConverter() {
        List<MediaType> mediaTypes = new ArrayList<>();
        mediaTypes.add(MediaType.APPLICATION_JSON);
        setSupportedMediaTypes(mediaTypes);
    }

}

package com.yushang.dsp.a_kj.config.resttemplate;

import cn.hutool.http.HttpUtil;
import com.yushang.dsp.a_kj.domain.constant.CacheConstants;
import com.yushang.dsp.a_kj.domain.constant.Constants;
import com.yushang.dsp.a_kj.utils.RedisCache;
import com.yushang.dsp.a_kj.utils.RedisSpinLock;
import com.yushang.dsp.a_kj.utils.SpringUtils;
import com.yushang.dsp.a_kj.utils.StringUtils;
import lombok.extern.log4j.Log4j;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpRequest;
import org.springframework.http.client.*;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.Proxy;
import java.sql.Time;
import java.util.Arrays;
import java.util.List;
import java.util.Random;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Slf4j
public class MyRestAuthInterceptor implements ClientHttpRequestInterceptor {

    private List<String> AgentLists = Arrays.asList(new String[]{
            "Opera/9.80 (X11; Linux i686; U; hu) Presto/2.9.168 Version/11.50",
            "Opera/9.80 (X11; Linux i686; U; ru) Presto/2.8.131 Version/11.11",
            "Opera/9.80 (X11; Linux i686; U; es-ES) Presto/2.8.131 Version/11.11",
            "Mozilla/5.0 (Windows NT 5.1; U; en; rv:1.8.1) Gecko/20061208 Firefox/5.0 Opera 11.11",
            "Opera/9.80 (X11; Linux x86_64; U; bg) Presto/2.8.131 Version/11.10",
            "Opera/9.80 (Windows NT 6.0; U; en) Presto/2.8.99 Version/11.10",
            "Opera/9.80 (Windows NT 5.1; U; zh-tw) Presto/2.8.131 Version/11.10",
            "Opera/9.80 (Windows NT 6.1; Opera Tablet/15165; U; en) Presto/2.8.149 Version/11.1",
            "Opera/9.80 (X11; Linux x86_64; U; Ubuntu/10.10 (maverick); pl) Presto/2.7.62 Version/11.01",
            "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36",
            "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.97 Safari/537.36",
            "Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:70.0) Gecko/20100101 Firefox/70.0",
            "Opera/9.80 (X11; Linux i686; Ubuntu/14.10) Presto/2.12.388 Version/12.16",
            "Opera/9.80 (Windows NT 6.0) Presto/2.12.388 Version/12.14",
            "Mozilla/5.0 (Windows NT 6.0; rv:2.0) Gecko/20100101 Firefox/4.0 Opera 12.14",
            "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.0) Opera 12.14",
            "Opera/12.80 (Windows NT 5.1; U; en) Presto/2.10.289 Version/12.02",
            "Opera/9.80 (Windows NT 6.1; U; es-ES) Presto/2.9.181 Version/12.00",
            "Opera/9.80 (Windows NT 5.1; U; zh-sg) Presto/2.9.181 Version/12.00",
            "Opera/12.0(Windows NT 5.2;U;en)Presto/22.9.168 Version/12.00",
            "Opera/12.0(Windows NT 5.1;U;en)Presto/22.9.168 Version/12.00",
            "Mozilla/5.0 (Windows NT 5.1) Gecko/20100101 Firefox/14.0 Opera/12.0",
            "Opera/9.80 (Windows NT 6.1; WOW64; U; pt) Presto/2.10.229 Version/11.62",
            "Opera/9.80 (Windows NT 6.0; U; pl) Presto/2.10.229 Version/11.62",
            "Opera/9.80 (Macintosh; Intel Mac OS X 10.6.8; U; fr) Presto/2.9.168 Version/11.52",
            "Opera/9.80 (Macintosh; Intel Mac OS X 10.6.8; U; de) Presto/2.9.168 Version/11.52",
            "Opera/9.80 (Windows NT 5.1; U; en) Presto/2.9.168 Version/11.51",
            "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; de) Opera 11.51",
            "Opera/9.80 (X11; Linux x86_64; U; fr) Presto/2.9.168 Version/11.50"
    });

    private String url = "http://proxy.siyetian.com/apis_get.html?token=AesJWLNR1Z41keJdXTq10dORUQ61keZhXT31STqFUeNpXQ61keFpXT6F1dOpXTx4EVjhXTq10d.QOxIDM4UDM4YTM&limit=1&type=0&time=&split=1&split_text=&area=0&repeat=0&isp=0";

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {

        int index = new Random().nextInt(28);

        String agent = AgentLists.get(index);

        SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();

//        String ipAddress = getIpAddress();
//        if (StringUtils.isNotEmpty(ipAddress)) {
//            //log.info("ip代理地址为:"+ipAddress);
//            String[] ip = ipAddress.split(":");
//            //获取高匿代理IP和端口
//            Proxy proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress(ip[0], Integer.valueOf(ip[1])));
//            requestFactory.setProxy(proxy);
//        }

        ClientHttpRequest clientHttpRequest = requestFactory.createRequest(request.getURI(), request.getMethod());

        HttpHeaders headers = request.getHeaders();
        clientHttpRequest.getHeaders().addAll(headers);

        return execution.execute(clientHttpRequest, body);

    }


    private String getIpAddress() {
        RedisSpinLock redisSpinLock = SpringUtils.getBean(RedisSpinLock.class);
        RedisCache redisCache = SpringUtils.getBean(RedisCache.class);

        redisSpinLock.lock(CacheConstants.IPLOCK, UUID.randomUUID().toString());
        try {
            Object cacheObject = redisCache.getCacheObject(CacheConstants.IPLOCK);
            if (cacheObject != null) {
                return (String) cacheObject;
            } else {
                String result = HttpUtil.get(url);
                if (result.contains("code")) {
                    log.error("获取IP失败", result);
                    return "";
                } else {
                    redisCache.setCacheObject(CacheConstants.IPLOCK, result, 2, TimeUnit.MINUTES);
                    return result;
                }
            }
        } finally {
            redisSpinLock.unlock(CacheConstants.IPLOCK, UUID.randomUUID().toString());
        }

    }
}

package com.yushang.dsp.a_kj.config.resttemplate;

import org.apache.http.conn.params.ConnManagerParams;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.params.BasicHttpParams;
import org.apache.http.params.HttpParams;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

/**
 * @ClassName : RestTemplateConfig
 * @Description :
 * @Author : GC
 * @Date: 2020-09-09 14:58
 */
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        HttpComponentsClientHttpRequestFactory httpRequestFactory = new HttpComponentsClientHttpRequestFactory();

        //设置最大链接
        httpRequestFactory.setHttpClient(
                HttpClientBuilder.create().setMaxConnTotal(800).setMaxConnPerRoute(400)
                        .build()
        );

        httpRequestFactory.setConnectionRequestTimeout(30 * 1000);
        httpRequestFactory.setConnectTimeout(30 * 3000);
        httpRequestFactory.setReadTimeout(30 * 3000);
        RestTemplate restTemplate = new RestTemplate(httpRequestFactory);
        //消息转换器
        restTemplate.getMessageConverters().add(new WxMappingJackson2HttpMessageConverter());
        restTemplate.getMessageConverters().add(new MsgConverter());
        //设置拦截器
        restTemplate.getInterceptors().add(new MyRestAuthInterceptor());
        return restTemplate;
    }
}

package com.yushang.dsp.a_kj.config.resttemplate;

import org.springframework.http.MediaType;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;

import java.util.ArrayList;
import java.util.List;

/**
 * @ClassName : WxMappingJackson2HttpMessageConverter
 * @Description :
 * @Author : GC
 * @Date: 2020-09-10 16:09
 */
public class WxMappingJackson2HttpMessageConverter extends MappingJackson2HttpMessageConverter {
    public WxMappingJackson2HttpMessageConverter() {
        List<MediaType> mediaTypes = new ArrayList<>();
        mediaTypes.add(MediaType.TEXT_PLAIN);
        setSupportedMediaTypes(mediaTypes);
    }
}


```

