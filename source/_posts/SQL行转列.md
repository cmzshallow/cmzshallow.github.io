---
title: SQL行转列
date: 2024-04-29 10:03
tags:
  - MySQL
categories:
  - 算法
---

```sql
-- 2.SQL行转列
drop table if exists sql_learn_2;
create table sql_learn_2 (
	`id` int not null auto_increment primary key,
	`name` varchar(32) null,
	`subject` varchar(32) null,
	`score` float null
) engine = innodb;

begin;
insert into sql_learn_2 (`name`, `subject`, `score`) values 
("Jack", "Math", 80.5), ("Jack", "Chinese", 70.5), ("Jack", "English", 70.5),
("Mary", "Math", 90.5), ("Mary", "Chinese", 85.5), ("Mary", "English", 75.5),
("Jack", "Math", 85.5), ("Jack", "Chinese", 75.5), ("Jack", "English", 90.5);
commit;

select * from sql_learn_2;
 
drop procedure if exists sql_learn_2;
create procedure sql_learn_2()
begin
    set @sql = null;
    select group_concat(
        distinct concat(
            'max(case when subject = ''', subject ,''' then score else null end) as ', subject
        )
    ) into @sql
    from sql_learn_2;
    
    set @sql = concat('select name, ', @sql, ' from sql_learn_2 group by name');
 
    prepare stmt1 from @sql;
    execute stmt1;
    deallocate prepare stmt1;
end;
 
call sql_learn_2();
```
