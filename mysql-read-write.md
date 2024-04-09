## 读写分离

### 方案一：拦截器实现

ReadWriteInterceptor 实现了 Interceptor ， 是通过注解 @Intercepts({ @Signature() }) 过滤，并在拦截器内通过  MappedStatement 类的getSqlCommandType() 方法区分是否为读或写，然后根据负载均衡策略进行分配读操作。

数据源枚举

``` java
package com.example.canary.common.mybatis;

import lombok.AllArgsConstructor;
import lombok.Getter;

import java.util.ArrayList;
import java.util.List;

/**
 * 数据源
 *
 * @author zhaohongliang 2023-10-17 15:44
 * @since 1.0
 */
@Getter
@AllArgsConstructor
public enum DataSourceEnum {

    /**
     * master
     */
    MASTER("master"),

    /**
     * slave1
     */
    SLAVE1("slave1"),

    /**
     * slave2
     */
    SLAVE2("slave2");

    /**
     * key
     */
    private final String key;

    public static List<DataSourceEnum> getSlaveValues() {
        List<DataSourceEnum> slaveValues = new ArrayList<>();
        slaveValues.add(SLAVE1);
        slaveValues.add(SLAVE2);
        return slaveValues;
    }

}

```

数据源上下文

``` java
package com.example.canary.common.mybatis;

/**
 * 数据源上下文
 *
 * @author zhaohongliang 2023-10-16 20:39
 * @since 1.0
 */
public class DataSourceContextHolder {

    private DataSourceContextHolder() {}

    private static final ThreadLocal<DataSourceEnum> CONTEXT_HOLDER = new ThreadLocal<>();

    public static void setDataSourceKey(DataSourceEnum dataSourceKey) {
        CONTEXT_HOLDER.set(dataSourceKey);
    }

    public static DataSourceEnum getDataSourceKey() {
        return CONTEXT_HOLDER.get();
    }

    public static void clearDataSourceKey() {
        CONTEXT_HOLDER.remove();
    }
}
```

拦截器实现

``` java
package com.example.canary.common.mybatis;

import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.cache.CacheKey;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Signature;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionSynchronizationManager;

import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 拦截器动态切换数据源
 *
 * @author zhaohongliang 2023-10-16 22:39
 * @since 1.0
 */
@Slf4j
@Intercepts({
    @Signature(type = Executor.class, method = "update", args = { MappedStatement.class, Object.class }),
    @Signature(type = Executor.class, method = "query", args = { MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class }),
    @Signature(type = Executor.class, method = "query", args = { MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class })
})
@ConditionalOnProperty(value = "spring.datasource.cluster.enabled")
@Component
public class ReadWriteInterceptor implements Interceptor {

    @Autowired
    private DataSourceHealthIndicator dataSourceHealthIndicator;

    private AtomicInteger index = new AtomicInteger(0);

    @Override
    public Object intercept(Invocation invocation) throws Throwable {

        MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];
        // boolean isMaster = mappedStatement.getId().toLowerCase(Locale.ENGLISH).contains(DataSourceEnum.MASTER.getKey())
        boolean synchronizationActive = TransactionSynchronizationManager.isSynchronizationActive();
        // 获取写入操作数据源 key
        // DataSourceEnum writeDataSourceKey = getWriteDataSourceKey();

        // 当前是否处于事务同步活动状态
        if (!synchronizationActive) {
            if (mappedStatement.getSqlCommandType().equals(SqlCommandType.SELECT)) {
                // 负载均衡策略
                int currentIndex = Math.abs(index.getAndIncrement() % 2);
                DataSourceEnum slaveKey = DataSourceEnum.getSlaveValues().get(currentIndex);
                DataSourceContextHolder.setDataSourceKey(slaveKey);
            } else {
                DataSourceContextHolder.setDataSourceKey(DataSourceEnum.MASTER);
            }
        } else {
            DataSourceContextHolder.setDataSourceKey(DataSourceEnum.MASTER);
        }

        try {
            return invocation.proceed();
        } finally {
            // reset index
            if (index.get() == Integer.MAX_VALUE) {
                index = new AtomicInteger(0);
            }
            // clear
            DataSourceContextHolder.clearDataSourceKey();
        }
    }

    /**
     * 获取写入操作数据源 key
     *
     * @return
     */
    private DataSourceEnum getWriteDataSourceKey() {
        Health health = dataSourceHealthIndicator.health();
        // 获取健康状态的详细信息
        Map<String, Object> details = health.getDetails();
        if ("not healthy".equals(details.get("master"))) {
            // 主库不健康，选择备库
            return DataSourceEnum.SLAVE1;
        } else {
            // 主库健康，选择主库
            return DataSourceEnum.MASTER;
        }
    }

}
```

拓展：此拦截器还可以通过不同的 MappedStatement 的命名规范区分读写，例如：UserMasterMapper（写）UserSlaveMapper（读）通过 mappedStatement.getId().toLowerCase(Locale.ENGLISH).contains(DataSourceEnum.MASTER.getKey()) 判断是否为写入。此方式不推荐，代码侵入性较高。

### 方案二：多数据源

DataSource 自定义注解

``` java
package com.example.canary.common.mybatis;

import org.springframework.core.annotation.AliasFor;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 数据源
 *
 * @author zhaohongliang 2023-10-18 01:27
 * @since 1.0
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface DataSource {

    /**
     * value
     *
     * @return
     */
    @AliasFor("name")
    DataSourceEnum value() default DataSourceEnum.MASTER;

    /**
     * name
     *
     * @return
     */
    @AliasFor("value")
    DataSourceEnum name() default  DataSourceEnum.MASTER;

}

```

ReadOnly 自定义注解

``` java
package com.example.canary.common.mybatis;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 只读
 *
 * @author zhaohongliang 2023-10-20 09:52
 * @since 1.0
 */
@Target({ ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadOnly {

    /**
     * value
     *
     * @return
     */
    boolean value() default true;
}

```

AOP

``` java
package com.example.canary.common.mybatis;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.stereotype.Component;

/**
 * 数据源AOP切面
 *
 * @author zhaohongliang 2023-10-17 21:27
 * @since 1.0
 */
@Slf4j
@Aspect
@Component
public class DataSourceAspect {

    /**
     * 在 @ReadOnly 方法执行执行前后操作数据源
     *
     * @param proceedingJoinPoint
     * @param readOnly
     * @return
     * @throws Throwable
     */
    @Around("@annotation(readOnly)")
    public Object proceed(ProceedingJoinPoint proceedingJoinPoint, ReadOnly readOnly) throws Throwable {
        if (readOnly.value()) {
            DataSourceContextHolder.setDataSourceKey(DataSourceEnum.SLAVE1);
        } else {
            DataSourceContextHolder.setDataSourceKey(DataSourceEnum.MASTER);
        }

        try {
            // 执行目标方法
            return proceedingJoinPoint.proceed();
        } finally {
            DataSourceContextHolder.clearDataSourceKey();
        }
    }


    /**
     * 切点
     */
    @Pointcut("@annotation(com.example.canary.common.mybatis.DataSource) || @within(com.example.canary.common.mybatis.DataSource)")
    public void pointcut() {
    }


    @Around("pointcut()")
    public Object proceed(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {

        MethodSignature methodSignature = (MethodSignature) proceedingJoinPoint.getSignature();

        // 查找方法上面的注解
        DataSource dataSource = AnnotationUtils.findAnnotation(methodSignature.getMethod(), DataSource.class);
        if (dataSource == null) {
            // 查找类上面的注解
            dataSource = AnnotationUtils.findAnnotation(methodSignature.getDeclaringType(), DataSource.class);
        }

        if (dataSource != null) {
            // 数据源名称
            DataSourceEnum dataSourceEnum = dataSource.value();
            DataSourceContextHolder.setDataSourceKey(dataSourceEnum);
        }
        // 执行目标方法
        return proceedingJoinPoint.proceed();
    }

    @Before("pointcut()")
    public void before() {
        // TODO document why this method is empty
    }

    @After("pointcut()")
    public void after() {
        DataSourceContextHolder.clearDataSourceKey();
    }


}

```

说明：虽然能够实现读写分离，但是驴唇不对马嘴。