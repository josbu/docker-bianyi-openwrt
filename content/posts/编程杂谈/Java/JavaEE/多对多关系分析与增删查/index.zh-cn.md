---
date: 2026-02-22T16:18:14+08:00
lastmod: 2026-02-22T17:54:34+08:00
categories:
  - 编程杂谈
  - Java
  - JavaEE
title: 多对多关系分析与增删查
draft: "false"
tags:
  - MySQL
series: []
---
## 案例引入-删除一个菜品，如果有关联的套餐则禁止删除
菜品和套餐的关系究竟是怎样的？删除一个菜品时，需要操作哪些表？下面先分析多对多关系，最后回到案例执行删除操作。

## 多对多关系分析
- 菜品，例如青菜、煎蛋、烤肠、牛奶
- 套餐：青菜+烤肠、煎蛋+牛奶、青菜 +煎蛋...

所以菜品与套餐的关系就是多对多的关系，一个套餐可能关联多个菜品，一个菜品也可能被多个套餐包含。

## 在设计数据库时，如何表达这种多对多的关系？
使用额外的一张表来描述它们的关系，把多对多拆分成了一对多。
- dish：菜品表
- setmeal：套餐表
- setmeal_dish: 菜品-套餐表。




![](Pasted%20image%2020260222162434.png)



这张多对多关系描述表实际上有冗余字段，后面会讲到为什么冗余
```sql
-- auto-generated definition  
create table setmeal_dish  
(  
    id         bigint auto_increment comment '主键'  
        primary key,  
    setmeal_id bigint         null comment '套餐id',  
    dish_id    bigint         null comment '菜品id',  
    name       varchar(32)    null comment '菜品名称 （冗余字段）',  
    price      decimal(10, 2) null comment '菜品单价（冗余字段）',  
    copies     int            null comment '菜品份数'  
)  
    comment '套餐菜品关系' collate = utf8mb3_bin;
```


## 多对多关系的新增操作

### 存放套餐
存放套餐的时候，应该操作哪些表？例如现在打算创建两个套餐，分别为
- `套餐1：青菜+烤肠`
- `套餐2：青菜+煎蛋`

那么SQL语句为

```sql
insert into setmeal(category_id, name, price, description, image, create_time, update_time, create_user, update_user)  
VALUES 
(2, '青菜爆炒烤肠', 4.0, '青菜烤肠套餐，好吃！', '1.jpg', now(), now(), 1,1),  
(2, '青菜煎鸡蛋', 5, '青菜煎鸡蛋，也好吃！', '2.jpg', now(), now(), 1,1)
```
查询数据库 
```
select id,name from setmeal
```
输出
```
33,青菜煎鸡蛋
32,青菜爆炒烤肠
```


此时只是插入了套餐`setmeal`自身的数据。但是菜品信息在哪里？显然，如果要创建套餐，必然保证有这个菜品，因此，还需要插入菜品信息

### 存放菜品

```sql
insert into dish(name, category_id, price, image, description, status, create_time, update_time, create_user, update_user)   
VALUES 
('青菜', 1, 2, 'qingcai.jpg', '青色的菜', 1,now(), now(), 1,1),  
('烤肠', 1, 3, 'kaochang.jpg', '烤的肠', 1, now(), now(), 1,1),  
('煎蛋', 1, 3, 'jiandan.jpg', '煎的蛋', 1, now(), now(), 1,1)
```

查询数据库
```sql
select id,name from dish
```
输出
```
80,煎蛋
79,烤肠
78,青菜
```


现在它们在数据库中是独立的数据，毫无关联。因此，需要在第三方表格中，真正建立起关系
### 关联套餐1与青菜、烤肠
```sql
insert into setmeal_dish(setmeal_id, dish_id, name, price, copies)  
VALUES  
    (32, 78, '', 1, 99), -- 关联套餐1与青菜  
    (32, 79, '', 1, 99) -- 关联套餐1与烤肠
```
### 关联套餐2与青菜、煎蛋
```sql
insert into setmeal_dish(setmeal_id, dish_id, name, price, copies)  
VALUES  
    (33, 78, '', 1, 99), -- 关联套餐2与青菜  
    (33, 80, '', 1, 99) -- 关联套餐2与煎蛋
```
现在套餐和菜品的关系才完全建立起来。分析发现name字段和price字段是多余的，因为不清楚name到底描述的是套餐的name还是菜品的name，价格也同理。



## 多对多查询操作
查询套餐1: `青菜爆炒烤肠`关联的菜品


```sql
-- 查询青菜爆炒烤肠关联的菜品id  
select sd.dish_id from setmeal_dish sd left join setmeal s on sd.setmeal_id = s.id where s.name = '青菜爆炒烤肠';  
  
-- 使用嵌套查询方式查询青菜爆炒烤肠关联的菜品名称与id  
select id,name from dish where id in (select sd.dish_id  
                                from setmeal_dish sd  
                                         left join setmeal s on sd.setmeal_id = s.id  
                                where s.name = '青菜爆炒烤肠')
```

查询套餐2: `青菜煎鸡蛋`关联的菜品
```sql
-- 使用嵌套查询方式查询青菜煎鸡蛋关联的菜品名称与id
select id,name from dish where id in (select sd.dish_id
                                from setmeal_dish sd
                                         left join setmeal s on sd.setmeal_id = s.id
                                where s.name = '青菜煎鸡蛋')
```



## 多对多关系删除操作
### 删除菜品-青菜，如果有关联套餐就不删除
通过以上的分析，我们知道，一个菜品，可能被多个套餐关联，例如`青菜`同时被套餐1和套餐2引用了，因此在删除青菜的时候，应该先查询是否存在关联，再执行操作。可以使用嵌套查询
```sql
delete from dish where name = '青菜' and id not in (select dish_id from setmeal_dish)
```







