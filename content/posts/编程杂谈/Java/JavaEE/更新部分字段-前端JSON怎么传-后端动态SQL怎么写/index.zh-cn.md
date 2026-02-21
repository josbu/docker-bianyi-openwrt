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
虽然的确知道了哪些字段要更新，但是这样怎么用在动态SQL语句？只能在service层逐个判断？看似合理，实际不可行。


所以衍生出了新的问题：有什么场景是前端必须传null的吗？对于一个表格，如果要设置某个字段为空，设置空字符串不就好了。这样后端DTO接收对象的时候，序列化就是空串而不是null，就能清楚的知道到底想干什么。


## 前端串空串而不是null
假如前端传过来的是 `{"phone": ""}`  在使用动态SQL时，以下哪个SQL语句能够设置空号码？

一，同时判断null和phone
也就是说，当DTO中的phone字段为null或者为空串时，不更新字段
```xml
<if test="phone != null and phone != ''"> phone = #{phone}, </if> 
```

二，仅判断null
此语句的含义是，当DTO中的phone字段为null时，不更新字段，其他情况，包括空串，也必须更新字段。
```xml
<if test="phone != null"> phone = #{phone}, </if> 
```

分析：如果前端传过来的是空串，那么DTO序列化后，phone字段就是""而不是null，意味只有第二种动态SQL写法能够更新字段。


## 前端传null

假如前端传过来的是 `{"phone": null}`  在使用动态SQL时，以下哪个SQL语句能够设置空号码？

一，同时判断null和phone
也就是说，当DTO中的phone字段为null或者为空串时，不更新字段
```xml
<if test="phone != null and phone != ''"> phone = #{phone}, </if> 
```

二，仅判断null
此语句的含义是，当DTO中的phone字段为null时，不更新字段，其他情况，包括空串，也必须更新字段。
```xml
<if test="phone != null"> phone = #{phone}, </if> 
```

分析：如果前端传过来的是null，那么DTO序列化后，phone字段就是null，无论是第一种动态SQL写法还是第二种动态SQL写法，都不会更新数据库。



## MyBatis动态SQL中，何时判断null，何时判断空串'' ?

具体来说，何时使用`phone != null` ，何时使用 `phone != null and phone != ''` ，以下动态SQL合理吗？

```xml
<update id="updateById">  
    update employee  
        <set>  
            <if test="username != null and username != ''"> username = #{username}, </if>  
            <if test="name != null and name != ''"> name = #{name}, </if>  
            <if test="password != null and password != ''"> password = #{password}, </if>  
            <if test="phone != null and phone != ''"> phone = #{phone}, </if>  
            <if test="sex != null and sex != ''"> sex = #{sex}, </if>  
            <if test="idNumber != null and idNumber != ''"> idNumber = #{idNumber}, </if>  
            <if test="status != null and status != ''"> status = #{status}, </if>  
            <if test="createTime != null and createTime != ''"> createTime = #{createTime}, </if>  
            <if test="updateTime != null and updateTime != ''"> updateTime = #{updateTime}, </if>  
            <if test="createUser != null and createUser != ''"> createUser = #{createUser}, </if>  
            <if test="updateUser != null and updateUser != ''"> updateUser = #{updateUser}, </if>  
        </set>  
    where id = #{id}  
</update>
```
分析：这里所有的字段都判断了null和空串，意味着如果字段没有被赋值，就不更新，这样就能保证在数据库中，他一定有值。

有哪些字段是必须有值的？取决于项目的约定。
- 例1，项目约定status是账号状态，必须是0或者1，这样前端在传值的时候，必须明确设置字段，而不能传null或者空串，如果前端传了空串，这个动态SQL就会认为不更新此字段。
- 例2，项目约定phone可以为空，假如有一位用户，前期设置了号码为`12345678999`，现在数据库中该字段存着该号码。此时用户又想删掉该号码，那么前端传过来的字段就是 `{"phone": ""}` 而不是 `{"phone": null}` 因为前者可以被DTO序列化为空串，而后者被DTO识别为null。


所以，动态SQL语句优化后，变成如下语句。
```xml
    <update id="updateById">
        update employee
            <set>
                <if test="username != null"> username = #{username}, </if>
                <if test="name != null"> name = #{name}, </if>
                <if test="password != null"> password = #{password}, </if>
                <if test="phone != null"> phone = #{phone}, </if>
                <if test="sex != null"> sex = #{sex}, </if>
                <if test="idNumber != null"> idNumber = #{idNumber}, </if>
                <if test="status != null and status != ''"> status = #{status}, </if>
                <if test="createTime != null and createTime != ''"> createTime = #{createTime}, </if>
                <if test="updateTime != null and updateTime != ''"> updateTime = #{updateTime}, </if>
                <if test="createUser != null and createUser != ''"> createUser = #{createUser}, </if>
                <if test="updateUser != null and updateUser != ''"> updateUser = #{updateUser}, </if>
            </set>
        where id = #{id}
    </update>
```

意味着`status, createTime, updateTime, createUser, updateUser`都不允许为空串，必须赋值才更新，其余字段无所谓



## 总结
1. 前端如果要设置某个字段为空，不要传null，而是空串。
2. Mybatis动态SQL更新部分字段时，如果允许空串，那么仅仅用`xx !=null` 判断即可。如果不允许空串，则同时使用`xx !=null and xx != ''`





