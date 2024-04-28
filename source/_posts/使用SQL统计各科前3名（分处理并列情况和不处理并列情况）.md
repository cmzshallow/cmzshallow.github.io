---
title: 统计各科前3名
date: 2024-04-29 10:03
tags:
  - MySQL
categories:
  - 算法
---

```sql
-- 3.统计各科前3名（分处理并列情况和不处理并列情况）
drop table if exists sql_learn_3;
create table sql_learn_3 (
	`id` int not null auto_increment primary key,
	`name` varchar(32) null,
	`subject` varchar(32) null,
	`score` float null
) engine = innodb;
	
begin;
insert into sql_learn_3 (`name`, `subject`, `score`) values 
("Jack", "Java", 100), ("Mary", "Java", 90), ("Marty", "Java", 90), ("Angelia", "Java", 80),
("Jack", "Python", 100), ("Mary", "Python", 95), ("Marty", "Python", 90), ("Angelia", "Python", 85);
commit;

select * from sql_learn_3;

-- 不处理并列情况
-- 解法一：
select
	name, subject, score
from
	(
		select 
			name, subject, score, rank() over(partition by subject order by score desc) as `rank`
		from sql_learn_3
	) as rank_table
where
	rank_table.`rank` <= 3;

-- 解法二：
select
	name, subject, score, (
		select 
			count(1)
		from 
			`sql_learn_3` s2 
		where 
			s1.name != s2.name 
			and s1.subject = s2.subject 
			and s2.score > s1.score
	) + 1 as `rank`
from 
	`sql_learn_3` s1
having
	`rank` <= 3
order by
	s1.subject, `rank`;

-- 处理并列情况
select
	name, subject, score, (
		select count(1) from (
			select 
				distinct(score) 
			from 
				`sql_learn_3` s2 
			where 
				s1.name != s2.name 
				and s1.subject = s2.subject 
				and s2.score > s1.score
			) as score_count
	) + 1 as `rank`
from 
	`sql_learn_3` s1
having
	`rank` <= 3
order by
	s1.subject, `rank`;
```
