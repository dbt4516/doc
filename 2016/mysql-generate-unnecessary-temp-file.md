## 1 现象
12月1日起观测到某服务器系统盘占用超过90%触发报警，但很快又自发回落，涉及到的文件大小应在3G以上。整个过程一般持续10至15分钟，每数小时出现一次，出现时机似无规律。<br>
## 2 定位问题组件
在问题主机上添加磁盘占用监控脚本，内容包括df及系统盘du信息，每分钟执行一次。观察结果为df显著增长时，du信息无任何变化。<br>
此时yilin提出检查lsof，将lsof写入监控脚本观察。后终有所获。<br>
![](https://github.com/dbt4516/doc/blob/master/2016/pic/mysql-generate-unnecessary-temp-file-1.png)
通过进程号查询可定位问题出现在该主机的mysql实例1上，请求由一台缓存IP信息的主机发起。<br>
Mysql临时文件会占用磁盘空间但却在du中无法显示这一现象的原因在于mysql在生成临时文件时会立即将其unlink，将清理临时文件的任务交给操作系统，防止运行中途异常结束，临时文件未及处理的情况发生。<br>
>MySQL arranges that temporary files are removed if mysqld is terminated. On platforms that support it (such as Unix), this is done by unlinking the file after opening it. The disadvantage of this is that the name does not appear in directory listings and you do not see a big temporary file that fills up the file system in which the temporary file directory is located. <br>


## 3 定位问题SQL

Mysql写临时文件的情况有两种，其一是写binlog，其二是查询时使用文件排序或生成临时表。但组内的设置使得binlog并不会打在系统盘内，可能性就只有因查询生成了。这时候就像看慢查询日志，但很遗憾，慢查询功能没开。于是改用土办法：磁盘占用量暴增时脚本连接mysql执行show processlist打出结果。经过漫长的守株待兔，截获这样一条可疑语句：
```sql
select ip_head,ip_tail,isp_id,city_id,province_id from ip_code order by ip_head limit 4500000,14683
```

之后手动执行，确实成功复现系统盘容量暴增的现象。查看该语句的执行记录，Created_tmp_files的值为4，确实生成了临时文件。<br>
## 4 SQL优化失效导致的外部排序
查看该数据表的索引信息，发现在用于排序的ip_head字段上是有B+索引的，但查看这条语句的执行计划，却发现执行中并未使用该索引，而是用了外部排序，而目前该数据库的临时文件就是存在/tmp目录。这就解释了为什么这条语句会导致磁盘占用暴增。执行计划如下图：
```
+----+-------------+---------+------+---------------+------+---------+------+---------+----------------+
| id | select_type | table   | type | possible_keys | key  | key_len | ref  | rows    | Extra          |
+----+-------------+---------+------+---------------+------+---------+------+---------+----------------+
|  1 | SIMPLE      | ip_code | ALL  | NULL          | NULL | NULL    | NULL | 4514694 | Using filesort | 
+----+-------------+---------+------+---------------+------+---------+------+---------+----------------+
```
在语句中增加use index (i_ip_head)，并不改变以上结果。改用force index (i_ip_head)，执行计划就比较符合预期：
```
+----+-------------+---------+-------+---------------+-----------+---------+------+---------+-------+
| id | select_type | table   | type  | possible_keys | key       | key_len | ref  | rows    | Extra |
+----+-------------+---------+-------+---------------+-----------+---------+------+---------+-------+
|  1 | SIMPLE      | ip_code | index | NULL          | i_ip_head | 5       | NULL | 4514694 |       | 
+----+-------------+---------+-------+---------------+-----------+---------+------+---------+-------+
```
在167上使用索引和不使用索引的执行时间对比，说明mysql在安排执行计划时，未能足够好地优化。（169查询速度非常慢）
```
+-----------+-----------------+
| use index | without index   |
+-----------+-----------------+
| 11s       | 124s            |
+-----------+-----------------+
```

此处也补充说明一下use index和force index的区别：二者都是对优化器的hint，前者只是在优化器选错索引时提出建议，后者则会使得优化器在能使用索引时就尽量使用索引，避免全表扫描，在对查询有充分把握时才能使用，强行使用索引在某些情况下，可能使得查询性能比全表扫描更差。<br>

## 5 Mysql的索引与排序
Mysql有两种文件排序算法（original和modified），两种做法的共性是都先用where过滤一遍表，将待排序的行分成若干份，分别快排，之后做归并排序。因此，如果order by与limit结合使用且limit的上限较小的话，排序速度比limit上限大时快得多。根据这一特点，改写之前的排序语句，在区间范围位于结果集后程时，将条件由升序改为降序，如：
```sql
order by ip_head desc limit 14683
```
可极大改善查询时间，经测试只需不到1秒，为所有方法中最快。<br>
Original和modified算法的差异在于，original在排序时只将order by子句中要用到的字段及行id放在排序缓冲区中，这使得它需要的缓冲区较小，但在返回结果之前，它也必须根据行id到数据表中将投影所需的字段查出来，返回，相当于要读表两次；而在modified算法中，排序缓冲区中同时存入了所有投影字段的值，排序结束后，可直接返回。当需要投影的字段过长，不宜全部放入缓冲区时，将倾向使用original算法。<br>
另，发现当投影字段只有ip_head时，优化器会主动使用ip_head上的索引（此时该索引其实就是覆盖索引）；并且，在limit上限达到表size之前（哪怕只差1条），优化器也会主动使用索引。只有当ip_head无法作为覆盖索引，且limit上限与表size相同时，才会使优化器放弃使用索引。<br>
![](https://github.com/dbt4516/doc/blob/master/2016/pic/mysql-generate-unnecessary-temp-file-2.png)

## 6 一些建议
1、开启慢查询日志，或能防范于未然（此处如果不是出发了磁盘报警，这条慢查询语句也不会被注意到），也能在定位问题时速度更快。<br>
2、启动mysql时追加--tmpdir参数，改变mysql临时文件存放路径，避免所有实例的临时文件都向系统盘写。<br>

## 7 参考材料
Mysql对limit子句的优化措施<br>
http://dev.mysql.com/doc/refman/5.5/en/limit-optimization.html<br>
Mysql的排序策略<br>
http://dev.mysql.com/doc/refman/5.7/en/order-by-optimization.html<br>
Mysql的索引提示说明<br>
http://dev.mysql.com/doc/refman/5.7/en/index-hints.html<br>
Mysql执行计划输出说明<br>
http://dev.mysql.com/doc/refman/5.5/en/explain-output.html<br>
一个现象十分相似的案例，但原因不同<br>
http://www.fromdual.com/beware-of-large-mysql-max-sort-length-parameter<br>
说服Mysql优化器使用索引的7中办法（挺有创意）<br>
http://code.openark.org/blog/mysql/7-ways-to-convince-mysql-to-use-the-right-index<br>
