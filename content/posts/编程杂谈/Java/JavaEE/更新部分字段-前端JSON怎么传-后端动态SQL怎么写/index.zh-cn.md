---
date: 2026-02-21T10:27:04+08:00
lastmod: 2026-02-21T11:22:16+08:00
categories:
  - 编程杂谈
  - Java
  - JavaEE
title: 更新部分字段-前端JSON怎么传-后端动态SQL怎么写
draft: "false"
tags:
  - Mybatis
series: []
---
## 更新部分字段时，服务端如何处理前端传过来的null字段？

核心思想：只要你传key，服务端必更新。
- 方法一：用Map接收，判断是否有Key。但是DTO对象就没用了，判断Key的方式也要思考，如果以后字段名变更了呢？显然这种方式可行但不合理。
- 方法二：在DTO对象中，添加一个集合记录哪些字段被设置了。
```java
public class UserPatchDTO {
    private String name;
    private Integer age;
    private Map<String, Object> rawUpdates = new HashMap<>();

    // 所有 JSON 字段都会经过此方法，记录原始值
    @JsonAnySetter
    public void setRawField(String key, Object value) {
        rawUpdates.put(key, value);
    }

    // 显式 Setter 也可以同时调用，但需要确保 Jackson 优先使用 @JsonAnySetter
    public void setName(String name) {
        this.name = name;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    // 判断字段是否被传递
    public boolean containsField(String field) {
        return rawUpdates.containsKey(field);
    }

    // 获取字段原始值（可能为 null）
    public Object getRawField(String field) {
        return rawUpdates.get(field);
    }

    // getters...
}
```
- 虽然的确知道了哪些字段要更新，但是这样怎么用在动态SQL语句？只能在service层逐个判断？看似合理，实际不可行。

- 所以衍生出了新的问题：有什么场景是前端必须传null的吗？对于一个表格，如果要设置某个字段为空，设置空字符串不就好了。这样后端DTO接收对象的时候，序列化就是空串而不是null，就能清楚的知道到底想干什么。


## 前端传空串而不是null
假如前端传过来的是 `{"phone": ""}`  在使用动态SQL时，以下哪个SQL语句能够设置空号码？

### 一，同时判断null和空串''
也就是说，当DTO中的phone字段为null或者为空串时，不更新字段
```xml
<if test="phone != null and phone != ''"> phone = #{phone}, </if> 
```

### 二，仅判断null
此语句的含义是，当DTO中的phone字段为null时，不更新字段，其他情况，包括空串，也必须更新字段。
```xml
<if test="phone != null"> phone = #{phone}, </if> 
```

分析：如果前端传过来的是空串，那么DTO序列化后，phone字段就是""而不是null，意味只有第二种动态SQL写法能够更新字段。


## 前端传null

假如前端传过来的是 `{"phone": null}`  在使用动态SQL时，以下哪个SQL语句能够设置空号码？

### 一，同时判断null和空串''
也就是说，当DTO中的phone字段为null或者为空串时，不更新字段
```xml
<if test="phone != null and phone != ''"> phone = #{phone}, </if> 
```

### 二，仅判断null
此语句的含义是，当DTO中的phone字段为null时，不更新字段，其他情况，包括空串，也必须更新字段。
```xml
<if test="phone != null"> phone = #{phone}, </if> 
```

分析：如果前端传过来的是null，那么DTO序列化后，phone字段就是null，无论是第一种动态SQL写法还是第二种动态SQL写法，都不会更新数据库。



## MyBatis动态SQL中，何时判断null，何时判断空串'' ?

是否判断 ''，取决于字段类型是否是 String。

- String 类型：通常要判断 != null and != ''
- 非 String 类型（Integer / Long / LocalDateTime）：只能判断 != null，不能判断 ''


例如有如下实体类
```java

public class Employee implements Serializable {  
  
    private static final long serialVersionUID = 1L;  
  
    private Long id;  
  
    private String username;  
  
    private String name;  
  
    private String password;  
  
    private String phone;  
  
    private String sex;  
  
    private String idNumber;  
  
    private Integer status;  
  
    //@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")  
    private LocalDateTime createTime;  
  
    //@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")  
    private LocalDateTime updateTime;  
  
    private Long createUser;  
  
    private Long updateUser;  
  
}
```


那么动态SQL可以写成
```xml
<update id="updateById">  
    update employee  
    <set>  
        <!-- String：判 null + '' -->  
        <if test="username != null and username != ''"> username = #{username}, </if>  
        <if test="name != null and name != ''"> name = #{name}, </if>  
        <if test="password != null and password != ''"> password = #{password}, </if>  
        <if test="phone != null and phone != ''"> phone = #{phone}, </if>  
        <if test="sex != null and sex != ''"> sex = #{sex}, </if>  
        <if test="idNumber != null and idNumber != ''"> id_number = #{idNumber}, </if>  
  
        <!-- 非 String：只判 null -->  
        <if test="status != null"> status = #{status}, </if>  
        <if test="createTime != null and createTime != ''"> create_time = #{createTime}, </if>  
        <if test="updateTime != null"> update_time = #{updateTime}, </if>  
        <if test="createUser != null"> create_user = #{createUser}, </if>  
        <if test="updateUser != null"> update_user = #{updateUser}, </if>  
    </set>  
    where id = #{id}  
</update>
```
如果非String类型判断空串，mybatis会报错

```
 Error updating database.  Cause: java.lang.IllegalArgumentException: invalid comparison: java.time.LocalDateTime and java.lang.String
```

- 对于非String类型，应该约定只要设置了值，就不要清空了，而是仅允许修改为另一个有效值。



## 总结
1. 前端如果要设置某个字段为空，不要传null，而是空串，但是这只能对字符串有效。
2. 对于非字符串类型，应该约定只要设置了就仅允许修改为另一个有效值，前端传null表示不做任何操作而不是清空字段。





