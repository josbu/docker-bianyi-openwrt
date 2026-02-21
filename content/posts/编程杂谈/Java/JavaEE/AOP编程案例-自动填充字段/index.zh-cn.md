---
date: 2026-02-22T00:28:45+08:00
lastmod: 2026-02-22T00:28:45+08:00
categories:
  - 编程杂谈
  - Java
  - JavaEE
title: AOP编程案例-自动填充字段
draft: "false"
tags:
  - AOP
  - SpringBoot
series: []
---
## 案例：service层修改与新增操作自动填充字段

已知Service层中，每次新增和修改操作都要调用部分以下方法
- 新增：`createTime,updateTime,createUser,updateUser`，
- 修改：`updateTime, updateUser`

现在我们打算把这些方法统一抽取到AOP中，利用反射方法去调用它们。

## 明确切入点：mapper的所有的insert和update方法
仅使用`execution(* com.sky.mapper.*.*(..))` 会拦截到到一些不必要的方法，例如查询和删除，因此可以配合自定义注解`@Autofill`，能够更准确的拦截连接点，即在需要增强的方法手动添加该注解。


### 自定义注解Autofill
```java
package com.sky.annotation;  
  
import com.sky.enumeration.OperationType;  
  
import java.lang.annotation.ElementType;  
import java.lang.annotation.Retention;  
import java.lang.annotation.RetentionPolicy;  
import java.lang.annotation.Target;  
  
// 此注解用于给AOP指定需要自动填充“更新日期“、”创建日期“的字段的方法上  
@Target(ElementType.METHOD) // 生效位置：方法  
@Retention(RetentionPolicy.RUNTIME) // 生效时机:运行时  
public @interface AutoFill {  
    // 数据库操作类型，区分UPDATE和INSERT  
    OperationType value();  
}
```

注意到我们在注解中使用了一个OperationType作为成员，这是为了区分UPDATE和INSERT的枚举类型
```java
package com.sky.enumeration;  
  
/**  
 * 数据库操作类型  
 */  
public enum OperationType {  
  
    /**  
     * 更新操作  
     */  
    UPDATE,  
  
    /**  
     * 插入操作  
     */  
    INSERT  
  
}
```

### 拦截Mapper
现在我们直接在Mapper中拦截所有的INSERT和UPDATE方法，往这些方法上加自定义注解。
- CategoryMapper
```java
package com.sky.mapper;

import com.github.pagehelper.Page;
import com.sky.annotation.AutoFill;
import com.sky.dto.CategoryPageQueryDTO;
import com.sky.entity.Category;
import com.sky.enumeration.OperationType;
import org.apache.ibatis.annotations.Delete;
import org.apache.ibatis.annotations.Mapper;

@Mapper
public interface CategoryMapper {

    /**
     * 分类-分页查询
     * @param categoryPageQueryDTO
     * @return
     */
    Page<Category> page(CategoryPageQueryDTO categoryPageQueryDTO);

    /**
     * 分类-新增
     * @param category
     */
    @AutoFill(OperationType.INSERT)
    void insert(Category category);

    /**
     * 分类-启用或者禁用
     * @param category
     */
    @AutoFill(OperationType.UPDATE)
    void update(Category category);

    @Delete("delete from category where id = #{id}")
    void deleteById(Long id);
}
```


### `@PointCut` 写法
使用表达式逻辑运算，保证是mapper下的@Autofill注解
```java
@Pointcut("execution(* com.sky.mapper.*.*(..)) && @annotation(com.sky.annotation.AutoFill)")  
public void autoFillPointCut() {}
```


## 明确通知：增强方法-书写自动填充逻辑

由于我们要对方法的参数进行赋值，因此需要用反射获取到传入的对象，并调用它的set方法。

```java
package com.sky.aop;

import com.sky.annotation.AutoFill;
import com.sky.constant.AutoFillConstant;
import com.sky.context.BaseContext;
import com.sky.enumeration.OperationType;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.time.LocalDateTime;
import java.util.Objects;

@Aspect
@Slf4j
@Component
public class AutoFillAspect {

    @Pointcut("execution(* com.sky.mapper.*.*(..)) && @annotation(com.sky.annotation.AutoFill)")
    public void autoFillPointCut() {}

    @Before("autoFillPointCut()")
    public void autoFill(JoinPoint joinPoint) {
        // 获取方法与注解
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        AutoFill annotation = methodSignature.getMethod().getAnnotation(AutoFill.class);
        OperationType operationType = annotation.value();

        // 准备需要赋值的数据
        Long currentId = BaseContext.getCurrentId();
        LocalDateTime now = LocalDateTime.now();

        // 获取需要被赋值的对象。约定连接点的第一个参数必须是被赋值对象。
        Object[] args = joinPoint.getArgs();
        if (Objects.isNull(args) ||  args.length == 0) { return; }
        Object entity = args[0];

            // 使用反射获取方法
            try {
                Method createTime = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_CREATE_TIME, LocalDateTime.class);
                Method updateTime = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_UPDATE_TIME, LocalDateTime.class);
                Method createUser = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_CREATE_USER, Long.class);
                Method updateUser = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_UPDATE_USER, Long.class);

                // 根据操作方法赋值
                if (operationType == OperationType.INSERT) {
                    log.info("插入方法自动填充");
                    createUser.invoke(entity, currentId);
                    createTime.invoke(entity, now);
                    updateUser.invoke(entity, currentId);
                    updateTime.invoke(entity, now);
                } else if (operationType == OperationType.UPDATE) {
                    log.info("更新方法自动填充");
                    updateUser.invoke(entity, currentId);
                    updateTime.invoke(entity, now);
                }
            } catch (Exception e) {
                log.error("AOP出错{}",e.getMessage());
            }
    }
}

```

注意到我们上述的代码获取反射方法的时候用了自定义常量，实际上这就是setXX方法的字符串名称。
```java
package com.sky.constant;

/**
 * 公共字段自动填充相关常量
 */
public class AutoFillConstant {
    /**
     * 实体类中的方法名称
     */
    public static final String SET_CREATE_TIME = "setCreateTime";
    public static final String SET_UPDATE_TIME = "setUpdateTime";
    public static final String SET_CREATE_USER = "setCreateUser";
    public static final String SET_UPDATE_USER = "setUpdateUser";
}

```


此时，Service层所有的手动赋值都可以注释掉
```java
package com.sky.service.impl;  
  
import com.github.pagehelper.Page;  
import com.github.pagehelper.PageHelper;  
import com.sky.constant.MessageConstant;  
import com.sky.constant.StatusConstant;  
import com.sky.context.BaseContext;  
import com.sky.dto.CategoryDTO;  
import com.sky.dto.CategoryPageQueryDTO;  
import com.sky.entity.Category;  
import com.sky.exception.DeletionNotAllowedException;  
import com.sky.mapper.CategoryMapper;  
import com.sky.mapper.DishMapper;  
import com.sky.mapper.SetmealMapper;  
import com.sky.result.PageResult;  
import com.sky.service.CategoryService;  
import org.springframework.beans.BeanUtils;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.stereotype.Service;  
  
import java.time.LocalDateTime;  
  
@Service  
public class CategoryServiceImpl implements CategoryService {  
    @Autowired  
    private CategoryMapper categoryMapper;  
    @Autowired  
    private SetmealMapper setmealMapper;  
    @Autowired  
    private DishMapper dishMapper;  

  
    @Override  
    public void add(CategoryDTO categoryDTO) {  
        Category category = new Category();  
        BeanUtils.copyProperties(categoryDTO, category);  
        //category.setCreateTime(LocalDateTime.now());  
        //category.setUpdateTime(LocalDateTime.now());        //category.setCreateUser(BaseContext.getCurrentId());        //category.setUpdateUser(BaseContext.getCurrentId());        // 设置默认禁用分类  
        category.setStatus(StatusConstant.DISABLE);  
        categoryMapper.insert(category);  
    }  
  
    @Override  
    public void disableOrEnable(Long id, Integer status) {  
        Category build = Category.builder()  
                .id(id)  
                //.updateTime(LocalDateTime.now())  
                //.updateUser(BaseContext.getCurrentId())                .status(status)  
                .build();  
        categoryMapper.update(build);  
    }  
  
    @Override  
    public void update(CategoryDTO categoryDTO) {  
        Category category = new Category();  
        BeanUtils.copyProperties(categoryDTO, category);  
        //category.setUpdateTime(LocalDateTime.now());  
        //category.setUpdateUser(BaseContext.getCurrentId());        categoryMapper.update(category);  
    }  
  // ... 
}
```
