---
date: 2026-02-21T19:10:13+08:00
lastmod: 2026-02-21T19:10:13+08:00
categories:
  - 编程杂谈
  - Java
  - JavaEE
title: Mybatis如何一对多查询
draft: "false"
tags:
  - Mybatis
  - MySQL
series: []
---
## 案例：查询一名员工信息以及他的工作经历

实体类
```java
package com.tignioj.pojo;  
  
import lombok.Data;  
  
import java.time.LocalDate;  
import java.time.LocalDateTime;  
import java.util.List;  
  
@Data  
public class Emp {  
    private Integer id; //ID,主键  
    private String username; //用户名  
    private String password; //密码  
    private String name; //姓名  
    private Integer gender; //性别, 1:男, 2:女  
    private String phone; //手机号  
    private Integer job; //职位, 1:班主任,2:讲师,3:学工主管,4:教研主管,5:咨询师  
    private Integer salary; //薪资  
    private String image; //头像  
    private LocalDate entryDate; //入职日期  
    private Integer deptId; //关联的部门ID  
    private LocalDateTime createTime; //创建时间  
    private LocalDateTime updateTime; //修改时间  
  
    private String deptName; // 部门名称  
    private List<EmpExpr> exprList;  
}
```
工作经历实体类
```java
/**
 * 工作经历
 */
@Data
public class EmpExpr {
    private Integer id; //ID
    private Integer empId; //员工ID
    private LocalDate begin; //开始时间
    private LocalDate end; //结束时间
    private String company; //公司名称
    private String job; //职位
}

```

### 方法一，建立映射表
如果是查询一条数据中关联的多个数据，需要手动映射
首先建立映射表
```xml
    <resultMap id="empInfo" type="com.tignioj.pojo.Emp" >
        <id column="id" property="id"/>
        <result column="username" property="username"/>
        <result column="password" property="password"/>
        <result column="name" property="name"/>
        <result column="gender" property="gender"/>
        <result column="phone" property="phone"/>
        <result column="job" property="job"/>
        <result column="salary" property="salary"/>
        <result column="image" property="image"/>
        <result column="entry_date" property="entryDate"/>
        <result column="dept_id" property="deptId"/>
        <result column="create_time" property="createTime"/>
        <result column="update_time" property="updateTime"/>
        <result column="dept_name" property="deptName"/>

        <collection property="exprList" ofType="com.tignioj.pojo.EmpExpr">
            <result column="p_company" property="company"/>
            <result column="p_id" property="id"/>
            <result column="p_job" property="job"/>
            <result column="p_emp_id" property="empId"/>
            <result column="p_begin" property="begin"/>
            <result column="p_end" property="end"/>
        </collection>
    </resultMap>
```

接着查询结果中不再使用resultType，而是使用resultMap，并指定刚刚创建的映射表
```xml
    <select id="findById" resultMap="empInfo">
        select  e.*,
                d.name dept_name,
                p.company p_company,
                p.id p_id,
                p.job p_job,
                p.emp_id p_emp_id,
                p.begin p_begin,
                p.end p_end
        from emp e left join emp_expr p on e.id = p.emp_id
            left join dept d on d.id = e.dept_id
        where e.id = #{id}
    </select>
```

Service
```java
@Override  
public Emp getInfo(Integer id) {  
    return empMapper.findById(id);  
}
```

### 方法二：使用两条SQL语句
先查询员工id，再查询工作经历，然后把工作经历封装到员工信息。

```java
public Emp getInfo1(Integer id) {
	Emp emp = empMapper.findById(id);
	List<EmpExpr> exprList = empExprMapper.findByEmpId();
	emp.setExprList(exprList);
	return emp;
}
```