## 动态定时任务



以实现动态定时任务为需求，构建任务调度

#### 静态

开启@EnableScheduling、@EnableAsync

``` java

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.transaction.annotation.EnableTransactionManagement;

import javax.annotation.PostConstruct;
import java.util.TimeZone;


@SpringBootApplication(scanBasePackages = "org.xxx")
@EnableTransactionManagement
@MapperScan("org.xxx.**.dao")
@EnableAsync
@EnableScheduling
public class xxxAdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(xxxAdminApplication.class, args);
    }

    /**
     * 设置时区
     */
    @PostConstruct
    void started() {
        TimeZone.setDefault(TimeZone.getTimeZone("GMT+8"));
    }
}
```

MyTask.java

``` java
import lombok.extern.slf4j.Slf4j;
import org.xxx.admin.service.SyncService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * <p>定时任务</p>
 *
 * @ClassName MyTask
 * @Description 定时任务
 */
@Slf4j
@Component
public class MyTask {

    @Autowired
    private SyncService syncService;

    @Async
    @Scheduled(fixedRate = 60 * 1000) // 1分钟 执行一次
    // @Scheduled(cron = "0 0/2 8 22 * * ?") // 8-22点，每2分钟执行一次
    public void testTask() {
        log.info("执行计算落地页pv消耗定时任务：{}", "MyTask");
        syncService.updatePageConsumerPv();
    }


}

```





#### 动态

corn

``` tex
0 * * * * ? 每1分钟触发一次 
0 0 * * * ? 每天每1小时触发一次 
0 0 10 * * ? 每天10点触发一次 
0 * 14 * * ? 在每天下午2点到下午2:59期间的每1分钟触发 
0 30 9 1 * ? 每月1号上午9点半 
0 15 10 15 * ? 每月15日上午10:15触发
/5 * * * ? 每隔5秒执行一次 
0 /1 * * ? 每隔1分钟执行一次 
0 0 5-15 * * ? 每天5-15点整点触发 
0 0/3 * * * ? 每三分钟触发一次 
0 0-5 14 * * ? 在每天下午2点到下午2:05期间的每1分钟触发 
0 0/5 14 * * ? 在每天下午2点到下午2:55期间的每5分钟触发 
0 0/5 14,18 * * ? 在每天下午2点到2:55期间和下午6点到6:55期间的每5分钟触发 
0 0/30 9-17 * * ? 朝九晚五工作时间内每半小时 
0 0 10,14,16 * * ? 每天上午10点，下午2点，4点
0 0 12 ? * WED 表示每个星期三中午12点 
0 0 17 ? * TUES,THUR,SAT 每周二、四、六下午五点 
0 10,44 14 ? 3 WED 每年三月的星期三的下午2:10和2:44触发 
0 15 10 ? * MON-FRI 周一至周五的上午10:15触发
0 0 23 L * ? 每月最后一天23点执行一次 
0 15 10 L * ? 每月最后一日的上午10:15触发 
0 15 10 ? * 6L 每月的最后一个星期五上午10:15触发
0 15 10 * * ? 2005 2005年的每天上午10:15触发 
0 15 10 ? * 6L 2002-2005 2002年至2005年的每月的最后一个星期五上午10:15触发 
0 15 10 ? * 6#3 每月的第三个星期五上午10:15触发
```

网站：

``` tex
http://cron.qqe2.com/
```

mysql

``` mysql
-- 删除旧表
DROP TABLE IF EXISTS `task_base`;
-- 创建新表
CREATE TABLE `task_base` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `name` varchar(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '名称',
  `name_en` varchar(200) DEFAULT NULL COMMENT '名称（英文）',
  `description` varchar(200) DEFAULT NULL COMMENT '描述',
  `corn_expression` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '表达式',
  `class_name` varchar(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '类名',
  `method_name` varchar(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '方法名',
  `method_params` varchar(200) DEFAULT NULL COMMENT '方法参数',
  `is_disabled` bigint DEFAULT '0' COMMENT '是否禁用, 0:否 1:是',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
  `is_deleted` tinyint DEFAULT '0' COMMENT '是否删除 0:否 非0:是',
  PRIMARY KEY (`id`),
  UNIQUE KEY `udx_multi_1` (`name`,`is_deleted`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='定时任务';

-- 插入数据
INSERT INTO `task_base` (`id`, `name`, `name_en`, `description`, `corn_expression`, `class_name`, `method_name`, `method_params`, `is_disabled`, `create_time`, `update_time`, `is_deleted`) VALUES
(1, '更新数据', '', '每1分钟一次', '0 * * * * ?', 'org.xxx.admin.service.impl.XxxServiceImpl', 'test1', '', 0, '2023-05-04 08:26:08', '2023-05-12 08:10:09', 0),
(2, 'test', NULL, '每10分钟一次', '0 0/10 * * * ? ', 'org.xxx.admin.service.impl.XxxServiceImpl', 'test2', NULL, 0, '2023-05-12 08:10:09', '2023-05-15 03:50:40', 0);
```

ScheduledConfig.java

```java
package org.xxx.admin.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;

/**
 * <p>定时任务线程池</p>
 *
 * @ClassName ScheduledConfig
 * @Description 定时任务线程池
 * @Author zhaohongliang
 * @Date 2022-11-30 21:34
 * @Since 1.0
 */
@Slf4j
@EnableAsync
@Configuration
@EnableScheduling
public class ScheduledConfig implements SchedulingConfigurer {

    @Autowired
    private ThreadPoolTaskScheduler taskScheduler;

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setTaskScheduler(taskScheduler);
    }

    /**
     * 定时任务线程池
     *
     * @return
     */
    @Bean
    public ThreadPoolTaskScheduler taskScheduler() {
        log.info("Creating ThreadPoolTaskScheduler ...");
        ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        taskScheduler.setPoolSize(20);
        taskScheduler.setThreadNamePrefix("TaskExecutor-");
        taskScheduler.setWaitForTasksToCompleteOnShutdown(true);
        taskScheduler.setAwaitTerminationSeconds(60);
        log.info("Successfully started");
        return taskScheduler;
    }

}
```

CronTaskRegistrar.java

``` java
package org.xxx.admin.task;

import lombok.extern.slf4j.Slf4j;
import org.xxx.admin.context.SpringContext;
import org.xxx.admin.entity.task.TaskPO;
import org.xxx.admin.repository.TaskRepository;
import org.xxx.admin.utils.StringUtil;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
import org.springframework.scheduling.support.CronTrigger;
import org.springframework.stereotype.Component;
import org.springframework.util.CollectionUtils;

import java.lang.reflect.Method;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ScheduledFuture;

/**
 * <p>动态添加定时任务</p>
 *
 * @ClassName CronTaskRegistrar
 * @Description 动态添加定时任务
 * @Author zhaohongliang
 * @Date 2023-05-04 11:01
 * @Since 1.0
 */
@Slf4j
@Component
public class CronTaskRegistrar implements InitializingBean {

    /**
     * 正在运行的任务
     */
    private final Map<String, ScheduledFuture<?>> scheduledFutureMap = new ConcurrentHashMap<>(16);

    public Map<String, ScheduledFuture<?>> getScheduledFutureMap() {
        return scheduledFutureMap;
    }

    @Autowired
    private ThreadPoolTaskScheduler taskScheduler;

    @Autowired
    private TaskRepository taskRepository;

    /**
     * 执行任务
     *
     * @param runnable
     */
    public void executeCronTask(Runnable runnable) {
        runnable.run();
    }

    /**
     * 添加任务
     *
     * @param taskId
     * @param runnable
     * @param cronExpression
     */
    public void addCronTask(String taskId, Runnable runnable, String cronExpression) {
        ScheduledFuture<?> scheduledFuture = taskScheduler.schedule(runnable, new CronTrigger(cronExpression));
        scheduledFutureMap.put(taskId, scheduledFuture);
    }

    /**
     * 删除任务
     *
     * @param taskId
     */
    public void removeCronTask(String taskId) {
        ScheduledFuture<?> scheduledFuture = scheduledFutureMap.remove(taskId);
        if (scheduledFuture != null) {
          	if (scheduledFuture) {
                return;
            }
        	  scheduledFuture.cancel(true);
        }
    }

    /**
     * 注册任务
     */
    public void registerCronTask() {
        scheduledFutureMap.clear();

        // 监测线程
        MoniterRunable moniterRunable = new MoniterRunable(this);
        ScheduledFuture<?> moniterScheduledFuture = taskScheduler.schedule(moniterRunable, new CronTrigger("0 0 * * * ?"));
        scheduledFutureMap.put("0", moniterScheduledFuture);

        List<TaskPO> tasks = taskRepository.listTask();
        if (CollectionUtils.isEmpty(tasks)) {
            return;
        }
        tasks.forEach(task -> {
            Class clazz = null;
            String beanName = null;
            Object bean = null;
            Method method = null;
            try {
                clazz = Class.forName(task.getClassName());
                beanName = StringUtil.toLowerFirstCase(clazz.getSimpleName());
                bean = SpringContext.getBean(beanName);
                method = clazz.getMethod(task.getMethodName());
            } catch (ClassNotFoundException e) {
                log.error("启动【{}】任务失败，原因：找不到 {} 类，异常信息：{}", task.getName(),  task.getClassName(), e.getMessage());
                e.printStackTrace();
            } catch (NoSuchMethodException e) {
                log.error("启动【{}】任务失败，原因：找不到 {} 方法，异常信息：{}", task.getName(), task.getMethodName(), e.getMessage());
                e.printStackTrace();
            }
            if (bean != null && method != null ) {
                TaskRunnable taskRunnable = new TaskRunnable(task.getName(), task.getCronExpression(), bean, method);
                this.addCronTask(task.getId().toString(), taskRunnable, task.getCronExpression());
            }
        }) ;

    }

    /**
     * 初始化完bean执行
     */
    @Override
    public void afterPropertiesSet() {
        log.info("任务调度器开始执行...");
        this.registerCronTask();
        if (!CollectionUtils.isEmpty(scheduledFutureMap)) {
            scheduledFutureMap.forEach((k, v) -> {
                log.info("register taskId {} complete...", k);
            }) ;
        }

    }

}
```

MoniterRunnable.java

``` java
package org.xxx.admin.task;

import lombok.extern.slf4j.Slf4j;
import org.xxx.admin.context.SpringContext;
import org.xxx.admin.exception.BusinessException;
import org.xxx.admin.utils.StringUtil;
import org.springframework.beans.factory.annotation.Autowired;

import java.lang.reflect.Method;
import java.util.Map;
import java.util.concurrent.ScheduledFuture;

/**
 * <p>监测线程</p>
 *
 * @ClassName MoniterRunable
 * @Description 监测线程
 * @Author zhaohongliang
 * @Date 2023-05-04 20:04
 * @Since 1.0
 */
@Slf4j
public class MoniterRunable implements Runnable {


    private final CronTaskRegistrar cronTaskRegistrar;

    public MoniterRunable(CronTaskRegistrar cronTaskRegistrar) {
        this.cronTaskRegistrar = cronTaskRegistrar;
    }

    @Override
    public void run() {
        Map<String, ScheduledFuture<?>> scheduledFutureMap = cronTaskRegistrar.getScheduledFutureMap();
        scheduledFutureMap.forEach((k, v) -> {
            // TODO 发送预警
            log.info("register taskId {} run...", k);
        });
    }

}

```



TaskRunnable.java

``` java
package org.xxx.admin.task;

import lombok.extern.slf4j.Slf4j;
import org.xxx.admin.utils.DateUtil;
import org.springframework.scheduling.support.CronTrigger;
import org.springframework.scheduling.support.SimpleTriggerContext;

import java.lang.reflect.Method;
import java.time.Duration;
import java.time.LocalDateTime;
import java.util.Date;

/**
 * <p>线程</p>
 *
 * @ClassName TaskRunnable
 * @Description 线程
 * @Author zhaohongliang
 * @Date 2023-05-04 11:30
 * @Since 1.0
 */
@Slf4j
public class TaskRunnable implements Runnable {

    private final String taskName;

    private final String cronExpression;

    private final Object bean;

    private final Method method;

    public TaskRunnable(String taskName, String cronExpression, Object bean, Method method) {
        this.taskName = taskName;
        this.cronExpression = cronExpression;
        this.bean = bean;
        this.method = method;
    }

    // @SneakyThrows
    @Override
    public void run() {
        LocalDateTime startTime = LocalDateTime.now();
        try {
            method.invoke(bean);
        } catch (Exception e) {
            log.error("任务【{}】执行失败,异常信息：{}", taskName, e.getMessage());
            e.printStackTrace();
        }
        LocalDateTime endTime = LocalDateTime.now();
        Duration duration = Duration.between(startTime, endTime);
        Date nextTime = new CronTrigger(this.cronExpression).nextExecutionTime(new SimpleTriggerContext());
        log.info("【{}】执行完毕，开始时间：{}，结束时间：{}，执行耗时：{} ms，下次执行时间：{}", taskName, startTime, endTime, duration.toMillis(), DateUtil.getLocalDateTime(nextTime));

    }

}
```

TaskRepositoryImpl.java

``` java
package org.xxx.admin.repository.impl;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import org.xxx.admin.dao.TaskDao;
import org.xxx.admin.entity.task.TaskPO;
import org.xxx.admin.repository.TaskRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;

import java.util.List;

/**
 * <p>定时任务</p>
 *
 * @ClassName TaskRepositoryImpl
 * @Description 定时任务
 * @Author zhaohongliang
 * @Date 2023-05-04 13:56
 * @Since 1.0
 */
@Service
public class TaskRepositoryImpl implements TaskRepository {

    @Autowired
    private TaskDao taskDao;

    /**
     * 新增
     *
     * @param taskPO
     * @return
     */
    @Override
    public int insert(TaskPO taskPO) {
        return taskDao.insert(taskPO);
    }

    /**
     * 根据id查询
     *
     * @param id
     * @return
     */
    @Override
    public TaskPO getById(Long id) {
        return taskDao.selectById(id);
    }

    /**
     * 列表
     *
     * @return
     */
    @Override
    public List<TaskPO> listTask() {
        LambdaQueryWrapper<TaskPO> queryWrapper = new LambdaQueryWrapper<TaskPO>();
        queryWrapper.eq(TaskPO::getDisabled, 0);
        return taskDao.selectList(queryWrapper);
    }

    /**
     * 是否唯一
     *
     * @param id
     * @param name
     * @return
     */
    @Override
    public boolean checkTaskIsUnique(Long id, String name) {
        LambdaQueryWrapper<TaskPO> queryWrapper = new LambdaQueryWrapper<>();
        queryWrapper.ne(id != null, TaskPO::getId, id);
        queryWrapper.eq(StringUtils.hasText(name), TaskPO::getName, name);
        return taskDao.selectOne(queryWrapper) == null;
    }

    /**
     * 更新
     *
     * @param taskPO
     * @return
     */
    @Override
    public int update(TaskPO taskPO) {
        return taskDao.updateById(taskPO);
    }

    /**
     * 删除
     *
     * @param id
     * @return
     */
    @Override
    public int delete(Long id) {
        return taskDao.deleteById(id);
    }
}

```



TaskServiceImpl.java

```java
package org.xxx.admin.service.impl;

import lombok.extern.slf4j.Slf4j;
import org.xxx.admin.context.ResultEntity;
import org.xxx.admin.context.SpringContext;
import org.xxx.admin.entity.task.TaskAO;
import org.xxx.admin.entity.task.TaskPO;
import org.xxx.admin.exception.BusinessException;
import org.xxx.admin.repository.TaskRepository;
import org.xxx.admin.service.TaskService;
import org.xxx.admin.task.CronTaskRegistrar;
import org.xxx.admin.task.TaskRunnable;
import org.xxx.admin.utils.StringUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.lang.reflect.Method;
import java.util.Optional;

/**
 * <p>定时任务</p>
 *
 * @ClassName TaskServiceImpl
 * @Description 定时任务
 * @Author zhaohongliang
 * @Date 2023-05-04 15:24
 * @Since 1.0
 */
@Slf4j
@Service
public class TaskServiceImpl implements TaskService {

    @Autowired
    private TaskRepository taskRepository;

    @Autowired
    private CronTaskRegistrar cronTaskRegistrar;


    /**
     * 新增定时任务
     *
     * @param taskAO
     * @return
     */
    @Override
    @SuppressWarnings("rawtypes")
    public ResultEntity saveTask(TaskAO taskAO) {
        boolean flag = taskRepository.checkTaskIsUnique(null, taskAO.getName());
        if (!flag) {
            throw new BusinessException("任务名称重复");
        }
        TaskPO taskPO = taskAO.convertToPo();
        taskRepository.insert(taskPO);
        return ResultEntity.success(taskPO);
    }

    /**
     * 更新定时任务
     *
     * @param taskAO
     * @return
     */
    @Override
    @SuppressWarnings("rawtypes")
    public ResultEntity updateTask(TaskAO taskAO) {
        TaskPO oldTaskPO = taskRepository.getById(taskAO.getId());
        if (oldTaskPO == null) {
            throw new BusinessException("任务不存在");
        }
        boolean flag = taskRepository.checkTaskIsUnique(taskAO.getId(), taskAO.getName());
        if (!flag) {
            throw new BusinessException("任务名称重复");
        }
        TaskPO taskPO = taskAO.convertToPo();
        taskRepository.update(taskPO);
        return ResultEntity.success();
    }

    /**
     * 删除定时任务
     *
     * @param id
     * @return
     */
    @Override
    @SuppressWarnings("rawtypes")
    public ResultEntity deleteTask(Long id) {
        TaskPO oldTaskPO = taskRepository.getById(id);
        if (oldTaskPO == null) {
            throw new BusinessException("任务不存在");
        }
        taskRepository.delete(id);
        return ResultEntity.success();
    }

    /**
     * 执行定时任务
     *
     * @param id
     * @return
     */
    @Override
    @SuppressWarnings("rawtypes")
    public ResultEntity executeTask(Long id) {
        TaskPO taskPO = taskRepository.getById(id);
        Optional.ofNullable(taskPO).orElseThrow(() -> new BusinessException("此定时任务不存在"));
        Class<?> clazz = null;
        Object bean = null;
        Method method = null;
        try {
            clazz = Class.forName(taskPO.getClassName());
            String beanName = StringUtil.toLowerFirstCase(clazz.getSimpleName());
            bean = SpringContext.getBean(beanName);
            method = clazz.getMethod(taskPO.getMethodName());
        } catch (ClassNotFoundException e) {
            log.error("启动 {} 任务失败，原因：找不到 {} 类，异常信息：{}", taskPO.getName(),  taskPO.getClassName(), e.getMessage());
            throw new BusinessException("启动 " + taskPO.getName() + " 任务失败，原因：找不到" + taskPO.getClassName() + "类");
        } catch (NoSuchMethodException e) {
            log.error("启动 {} 任务失败，原因：找不到 {} 方法，异常信息：{}", taskPO.getName(), taskPO.getMethodName(), e.getMessage());
            throw new BusinessException("启动 " + taskPO.getName() + " 任务失败，原因：找不到" + taskPO.getMethodName() + "方法");
        }
        TaskRunnable taskRunnable = new TaskRunnable(taskPO.getName(), taskPO.getCronExpression(), bean, method);
        cronTaskRegistrar.executeCronTask(taskRunnable);

        return ResultEntity.success();
    }

    /**
     * 启动定时任务
     *
     * @param id
     * @return
     */
    @Override
    @SuppressWarnings("rawtypes")
    public ResultEntity startTask(Long id) {
        TaskPO taskPO = taskRepository.getById(id);
        Optional.ofNullable(taskPO).orElseThrow(() -> new BusinessException("此定时任务不存在"));
        Class<?> clazz = null;
        Object bean = null;
        Method method = null;
        try {
            clazz = Class.forName(taskPO.getClassName());
            String beanName = StringUtil.toLowerFirstCase(clazz.getSimpleName());
            bean = SpringContext.getBean(beanName);
            method = clazz.getMethod(taskPO.getMethodName());
        } catch (ClassNotFoundException e) {
            log.error("启动 {} 任务失败，原因：找不到 {} 类，异常信息：{}", taskPO.getName(),  taskPO.getClassName(), e.getMessage());
            throw new BusinessException("启动 " + taskPO.getName() + " 任务失败，原因：找不到" + taskPO.getClassName() + "类");
        } catch (NoSuchMethodException e) {
            log.error("启动 {} 任务失败，原因：找不到 {} 方法，异常信息：{}", taskPO.getName(), taskPO.getMethodName(), e.getMessage());
            throw new BusinessException("启动 " + taskPO.getName() + " 任务失败，原因：找不到" + taskPO.getMethodName() + "方法");
        }
        TaskRunnable taskRunnable = new TaskRunnable(taskPO.getName(), taskPO.getCronExpression(), bean, method);
        cronTaskRegistrar.addCronTask(taskPO.getId().toString(), taskRunnable, taskPO.getCronExpression());

        return ResultEntity.success();
    }

    /**
     * 停止定时任务
     *
     * @param id
     * @return
     */
    @Override
    @SuppressWarnings("rawtypes")
    public ResultEntity stopTask(Long id) {
        TaskPO taskPO = taskRepository.getById(id);
        Optional.ofNullable(taskPO).orElseThrow(() -> new BusinessException("此定时任务不存在"));
        cronTaskRegistrar.removeCronTask(taskPO.getId().toString());
        return ResultEntity.success();
    }
}

```

XxxServiceImpl.java

``` java
@Sl4j
@Service
public class XxxServiceImpl implements XxxService {
  
   /**
     * test
     *
     * @param id
     * @return
     */
    @Override
    public void test() {
        System.out.println("任务执行体...");
    }
}
```



TaskController.java

``` java
package org.xxx.admin.controller;

import org.xxx.admin.annotation.ApiVersion;
import org.xxx.admin.context.ResultEntity;
import org.xxx.admin.context.ValidGroup;
import org.xxx.admin.entity.task.TaskAO;
import org.xxx.admin.service.TaskService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import javax.validation.constraints.NotNull;

/**
 * <p>定时任务</p>
 *
 * @ClassName TaskController
 * @Description 定时任务
 * @Author zhaohongliang
 * @Date 2023-05-04 13:58
 * @Since 1.0
 */
@Validated
@ApiVersion
@RestController
@RequestMapping("/task")
public class TaskController {

    @Autowired
    private TaskService taskService;

    /**
     * 新增定时任务
     *
     * @param taskAO
     * @return
     */
    @PostMapping("/save")
    @SuppressWarnings("rawtypes")
    public ResultEntity saveTask(@Validated( { ValidGroup.Add.class }) @RequestBody TaskAO taskAO) {
        return taskService.saveTask(taskAO);
    }

    /**
     * 更新定时任务
     *
     * @param taskAO
     * @return
     */
    @PutMapping("/update")
    @SuppressWarnings("rawtypes")
    public ResultEntity updateTask(@Validated( { ValidGroup.Edit.class }) @RequestBody TaskAO taskAO) {
        return taskService.updateTask(taskAO);
    }

    /**
     * 删除定时任务
     *
     * @param id
     * @return
     */
    @DeleteMapping("/delete")
    @SuppressWarnings("rawtypes")
    public ResultEntity deleteTask(@NotNull @RequestParam Long id) {
        return taskService.deleteTask(id);
    }

    /**
     * 执行定时任务
     *
     * @param id
     * @return
     */
    @GetMapping("/execute")
    @SuppressWarnings("rawtypes")
    public ResultEntity executeTask(@NotNull @RequestParam Long id) {
        return taskService.executeTask(id);
    }

    /**
     * 启动定时任务
     *
     * @param id
     * @return
     */
    @GetMapping("/start")
    @SuppressWarnings("rawtypes")
    public ResultEntity startTask(@NotNull @RequestParam Long id) {
        return taskService.startTask(id);
    }

    /**
     * 停止定时任务
     *
     * @param id
     * @return
     */
    @GetMapping("/stop")
    public ResultEntity stopTask(@NotNull @RequestParam Long id) {
        return taskService.stopTask(id);
    }



}

```

