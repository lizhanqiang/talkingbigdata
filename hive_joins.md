```
create table student(id string,name string,class_id string) stored as orc tblproperties('transactional=true');


create table class(class_id string,class_name string) stored as orc tblproperties('transactional=true');


insert into student values('id_1','name_1','class_id_1'),('id_2','name_2','class_id_2'),('id_3','name_3','class_id_3'),('id_4','name_4','class_id_4'),('id_5','name_5',NULL);

insert into class values('class_id_1','class one'),('class_id_2','class two'),('class_id_3','class three');
```





`select * from student s inner join class c on s.class_id=c.class_id;`

![](imgs\hive\join\inner_join.png)

`select * from student s left join class c on s.class_id=c.class_id;`

![](imgs\hive\join\left_join.png)