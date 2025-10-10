---
title: "ThreadPoolTaskScheduler+cron表达式可配置化定时任务"
date: "2024-09-24"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

页面配置java生成cron表达式  
spring ThreadPoolTaskScheduler执行cron表达式任务  

参数
```
@Getter
@Setter
public class ConfigureBO {

    private Integer id;
    private Integer dbid;
    private Integer enabled;
    private Integer discoveryType;
    private String cronExpression;

    /**
     * 所选作业类型:
     * 1  -> 每天
     * 2  -> 每周
     * 3  -> 每月
     */
    private Integer jobType;

    /**
     * 一周的哪几天
     */
    private List<Integer> dayOfWeeks;

    /**
     * 一个月的哪几天
     */
    private List<Integer> dayOfMonths;

    /**
     * 秒
     */
    private Integer second;

    /**
     * 分
     */
    private Integer minute;

    /**
     * 时
     */
    private Integer hour;
}
```

根据参数生成cron表达式
```js
/**
     * 方法摘要：构建Cron表达式
     *
     * @param configureBO
     * @return String
     */
    public static String createCronExpression(ConfigureBO configureBO) {
        StringBuffer cronExp = new StringBuffer();

        if (null == configureBO.getJobType()) {
            throw new BusinessException("执行周期未配置");//执行周期未配置
        }

//        if (null != configureBO.getSecond()
//                && null == configureBO.getMinute()
//                && null == configureBO.getHour()) {
//            //每隔几秒
//            if (configureBO.getJobType() == 0) {
//                cronExp.append("0/").append(configureBO.getSecond());
//                cronExp.append(" ");
//                cronExp.append("* ");
//                cronExp.append("* ");
//                cronExp.append("* ");
//                cronExp.append("* ");
//                cronExp.append("?");
//            }
//
//        }
//
//        if (null != configureBO.getSecond()
//                && null != configureBO.getMinute()
//                && null == configureBO.getHour()) {
//            //每隔几分钟
//            if (configureBO.getJobType() == 4) {
//                cronExp.append("* ");
//                cronExp.append("0/").append(configureBO.getMinute());
//                cronExp.append(" ");
//                cronExp.append("* ");
//                cronExp.append("* ");
//                cronExp.append("* ");
//                cronExp.append("?");
//            }
//
//        }

        if (null != configureBO.getSecond()
                && null != configureBO.getMinute()
                && null != configureBO.getHour()) {
            //秒
            cronExp.append(configureBO.getSecond()).append(" ");
            //分
            cronExp.append(configureBO.getMinute()).append(" ");
            //小时
            cronExp.append(configureBO.getHour()).append(" ");

            if (configureBO.getJobType() == 0) {
                LocalDateTime now = LocalDateTime.now();
                int dayOfMonth = now.getDayOfMonth();
                int monthValue = now.getMonthValue();
                int year = now.getYear();
                cronExp.append(dayOfMonth).append(" ");
                cronExp.append(monthValue).append(" ");
                cronExp.append("? ");
//                cronExp.append(year);

            }
            //每天
            else if (configureBO.getJobType() == 1) {
                cronExp.append("* ");//日
                cronExp.append("* ");//月
                cronExp.append("?");//周
            }

            //按每周
            else if (configureBO.getJobType() == 2) {
                //一个月中第几天
                cronExp.append("? ");
                //月份
                cronExp.append("* ");
                //周
                String joinWeeks = Joiner.on(",").join(configureBO.getDayOfWeeks());

                cronExp.append(joinWeeks);

            }

            //按每月
            else if (configureBO.getJobType() == 3) {
                //一个月中的哪几天
                String joinMonths = Joiner.on(",").join(configureBO.getDayOfMonths());

                cronExp.append(joinMonths);
                //月份
                cronExp.append(" * ");
                //周
                cronExp.append("?");
            }

        } else {
            throw new BusinessException("时或分或秒参数未配置");//时或分或秒参数未配置
        }
        return cronExp.toString();
    }
```
校验cron表达式是否合法
```
private static boolean legal(String cronExpression) {
    CronSequenceGenerator sequenceGenerator = new CronSequenceGenerator(cronExpression);
    Date next = sequenceGenerator.next(new Date());
    long initialDelay = next.getTime() - System.currentTimeMillis();
    return initialDelay >= 0;
}
```
借助spring ThreadPoolTaskScheduler 注册定时任务

```
private static final ConcurrentHashMap<String, ScheduledFuture<?>> scheduledFutureConcurrentHashMap = new ConcurrentHashMap<>();

private ThreadPoolTaskScheduler threadPoolTaskScheduler;


public synchronized void reStartOrStopJob(AssetMetadataConfigure configure, DsIns dsIns) {
    String key = ThreadHolder.getTenantId() + "_" + configure.getDbid();
    //todo lock
    if (Objects.equals(configure.getEnabled(), 1)) {
        stopJob(key, configure);
        startJob(key, configure, dsIns);
    } else {
        stopJob(key, configure);
    }
}


private void startJob(String key, AssetMetadataConfigure configure, DsIns dsIns) {

    if (legal(configure.getCronExpression())) {

        ScheduledFuture<?> oldScheduledFuture = scheduledFutureConcurrentHashMap.computeIfAbsent(key, hashKey -> {
        return threadPoolTaskScheduler.schedule(
                new Job(ThreadHolder.getTenantId(), configure, dsDomainService, taskDispatchFacade, assetConfigureService),
                new CronTrigger(configure.getCronExpression()));
        });
        log.info("key:{} 已经关联定时任务, ", key);
    } else {
        log.info("表达式非法:{}", JSON.toJSONString(configure));

    }

}

private void stopJob(String key, AssetMetadataConfigure configure) {
    ScheduledFuture<?> scheduledFuture = scheduledFutureConcurrentHashMap.remove(key);
    if (Objects.nonNull(scheduledFuture)) {
        boolean cancel = scheduledFuture.cancel(true);
        log.info("修改周期任务:{}, cronExpression:{}, cancel:{}", JSON.toJSONString(configure), configure.getCronExpression(), cancel);
        if (!cancel) {
            throw new BusinessException("取消定时任务失败");
        }
    }
}
```