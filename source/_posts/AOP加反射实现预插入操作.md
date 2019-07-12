---
title: AOP加反射实现预插入操作
date: 2019-01-12 15:45:29
tags:
	- Java
	- Spring
	- aop
categories:
	- 项目笔记
description: 最近对老项目的基础上扩展新的移动端，因此相当于建立一个全新工程。在每次进行插入操作时要求先将保存当前的时间和当前插入者的角色，在老项目中进行保存操作时要先调用preInsert()对保存对象进行赋值后再持久化到数据库中。但是我觉得这种方式并不简洁，而且扩展性并不好，所以我就想到了用aop加反射的方法对插入进行预处理。
---

# 前言

最近对老项目的基础上扩展新的移动端，因此相当于建立一个全新工程。在每次进行插入操作时要求先将保存当前的时间和当前插入者的角色，在老项目中进行保存操作时要先调用`preInsert()`对保存对象进行赋值后再持久化到数据库中。但是我觉得这种方式并不简洁，而且扩展性并不好，所以我就想到了用aop加反射的方法对插入进行预处理。

<!-- more -->

# 运行环境

| 框架           | 版本                   |
| -------------- | ---------------------- |
| JDK            | 1.8.0_161              |
| Springboot     | 2.2.0 SNAPSHOT         |
| Springdata-jpa | springboot-starter-jpa |



# 代码实现

我的思路是这样的：创建一个自定义注解，通过注解的方式标识哪个方法需要进行插入的预处理工作。在切面中，利用前置通知在进入方法前通过反射获取参数对象进行赋值。下面详细看代码：

首先是定义的注解，value并没有用到，这个注解其实只是作为标识存在。

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface preInsert {
    String value() default "";
}

```

项目中的实体都是继承自父类`BaseEntity<T,ID extends Serizliable>`因此，插入时需要的`createTime`和`createBy`这些字段都放在父类中。方法中由`joinPoint`获取当前参数的对象，这里作了一个参数个数的判断，规定了参数只有一个对象值，去掉判断的话，就需要规定即将进行赋值的实体对象参数一定要放在第一位。

在获取到对象的引用之后，通过反射获得类型信息，再循环获取父类信息直到所有父类都不为BaseEntity时才捕获错误，之后就是常规的反射API调用了。

切面类

```java
@Aspect
@Component
public class InsertAop {
    Logger log = LoggerFactory.getLogger(this.getClass());
    /**
     * 切点：preInset注解
     */
    @Pointcut("@annotation(com.sinolvc.pgfp.app.commons.annotations.preInsert)")
    public void insertPointCut() {
    }
    /**
     * 通知类型：before
     * 进行保存操作前，将当前时间放到要保存的实体中
     * 适用于save(T extends BaseEntity),只有一个实体参数的情况
     * 调用方法：setcreateDate,setUpdateDate
     * @param joinPoint
     */
    @Before("insertPointCut()")
    public void beforeInsert(JoinPoint joinPoint) {
        /**
         * 获取参数对象
         */
        Object[] args = joinPoint.getArgs();
        if (args.length == 1) {
            Object arg = args[0];
            /**
             * 得到参数的父类信息
             */
            Class<?> clazz = arg.getClass();
            Date date = new Date();
            /**
             * 假如获取的父类不是BaseEntity，就继续取父类，直到没有取到BaseEntity为止就抛出异常
             */
            try {
               Class<?> superClazz = clazz.getSuperclass();
                while (!superClazz.getName().equals("com.sinolvc.pgfp.app.commons.base.BaseEntity")) {
                    superClazz = superClazz.getSuperclass();
                }
            } catch (NullPointerException e) {
                log.error("save：类" + clazz + "没有继承 BaseEntity");
            }
            try {
                Method setCreateBy = superClazz.getDeclaredMethod("setCreateBy",String.class);
                if(null != setCreateBy ) setCreateBy.invoke(arg,"1");
                Method setCreateDate = superClazz.getDeclaredMethod("setCreateDate", Date.class);
                if (null != setCreateDate) setCreateDate.invoke(arg, date);
                Method setUpdateDate = superClazz.getDeclaredMethod("setUpdateDate", Date.class);
                if (null != setUpdateDate) setUpdateDate.invoke(arg, date);
                Method setDelFlag = superClazz.getDeclaredMethod("setDelFlag",String.class);
                if(null != setDelFlag) setDelFlag.invoke(arg, Constant.DEL_FLAG_NORMAL);
            } catch (NoSuchMethodException e) {
                log.error("增强save方法：类" + clazz + "中没有 set 方法");
            } catch (IllegalArgumentException e) {
                log.error("增强save方法：类" + clazz + "中参数错误");
            } catch (IllegalAccessException | InvocationTargetException e) {
                log.error("增强save方法：类" + clazz + "中方法使用错误");
            }
        }
    }
}

```

注解在逻辑层的方法上

```java
    @Override
    @preInsert
    public Job save(Job job){
        return jobRepository.save(job);
    }
```

下面是BaseEntity的代码

```java
@MappedSuperclass
public class BaseEntity implements Serializable {
    /**
     *  记录标识
     */
    @Id
    @GeneratedValue( generator = "system-uuid" )
    @GenericGenerator(name = "system-uuid", strategy = "uuid")
    protected String id;
    /**
     *  创建日期
     */
    @Column( name = "create_date")
    protected Date createDate;
    /**
     *  更新日期
     */
    @Column( name = "update_date")
    protected Date updateDate;
    /**
     *  删除标记（0：正常；1：删除；2：审核）
     */
    @Column( name = "del_flag")
    protected String delFlag;

    @Column(name="create_by")
    protected String createBy;
   
/***********getter and setter**********//*getset方法省略*/
```

Job实体的方法就不用给出了，注意要继承BaseEntity类。

# 总结

最后发现通过这种方法来进行插入时性能会比直接调用赋值方法差一点点，但是还能接受，最重要的是这样写代码更加整洁，维护起来更加容易。