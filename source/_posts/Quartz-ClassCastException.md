---
title: Quartz-ClassCastException
date: 2021-07-02 10:04:07
tags:
    - Java
    - quartz
categories:
    - Exception
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
## 问题
项目中使用Quartz的时候，使用 jobDataMap中放入一个对象

{% codeblock lang:java %}
ScheduleJobEntity scheduleJob;
jobDetail.getJobDataMap().put(ScheduleJobEntity.ENTITY_PARAM_KEY, entity);
{% endcodeblock %}

如果使用反射的方式去加载类文件创建对象
```
ScheduleJobEntity entity = (ScheduleJobEntity) jobExecutionContext.getJobDetail()
                .getJobDataMap().get(ScheduleJobEntity.ENTITY_PARAM_KEY);
```
当进行强转的时候，就会抛出`ClassCastException`异常，当debug观察结果的时候，发现构建的对象类型也是`ScheduleJobEntity`类型，属性也相同，并没有发现
有其他的问题。
