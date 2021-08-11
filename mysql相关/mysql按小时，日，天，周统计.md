```mysql
-- 按小时统计每个盒子超速，重复计1
SELECT
	t.hours,
	COUNT(t.forklift_box_id) day_cs_count
FROM
	(	SELECT 
				date_format(t_event.time,'%Y-%m-%d %H') as hours ,
				forklift_box_id,		
				COUNT(forklift_box_id) num_count
		FROM
				t_event 						
		WHERE
				t_event.flag=1 and t_event.guest in(1,2,3,4,5,6,7,8)                               
				and type1=1 and type2=2 AND durationtime>=5 and currentvalue<=50
				and (time  between  '2021-01-26 00:00:00' and '2021-07-26 23:59:59')
		GROUP BY hours,forklift_box_id 
  ) t 
GROUP BY t.hours

-- 按天统计每个盒子超速，重复计1
SELECT
	t.days,
	COUNT(t.forklift_box_id) day_cs_count
FROM
	(	SELECT 
				date_format(t_event.time,'%Y-%m-%d') as days ,
				forklift_box_id,		
				COUNT(forklift_box_id) num_count
		FROM
				t_event 						
		WHERE
				t_event.flag=1 and t_event.guest in(1,2,3,4,5,6,7,8)																							                                                                                                                                             
				and type1=1 and type2=2 AND durationtime>=5 and currentvalue<=50
				and (time  between  '2021-01-26 00:00:00' and '2021-07-26 23:59:59')
		GROUP BY days,forklift_box_id 
  ) t 
GROUP BY t.days

-- 按月统计每个盒子超速，重复计1
SELECT
	t.months,
	COUNT(t.forklift_box_id) month_cs_count
FROM
	(	SELECT 
				date_format(t_event.time,'%Y-%m') as months ,
				forklift_box_id,		
				COUNT(forklift_box_id) num_count
		FROM
				t_event 						
		WHERE
				t_event.flag=1 and t_event.guest in(1,2,3,4,5,6,7,8)																							                                                                                                                                             
				and type1=1 and type2=2 AND durationtime>=5 and currentvalue<=50
				and (time  between  '2021-01-26 00:00:00' and '2021-07-26 23:59:59')
		GROUP BY months,forklift_box_id 
  ) t 
GROUP BY t.months

-- 按周统计每个盒子超速，重复计1
SELECT
	t.weeks,
	COUNT(t.forklift_box_id) weeks_cs_count
FROM
	(	SELECT 
				date_format(t_event.time,'%X-第%V周') as weeks ,
				forklift_box_id,		
				COUNT(forklift_box_id) num_count
		FROM
				t_event 						
		WHERE
				t_event.flag=1 and t_event.guest in(1,2,3,4,5,6,7,8)																							                                                                                                                                             
				and type1=1 and type2=2 AND durationtime>=5 and currentvalue<=50
				and (time  between  '2021-01-26 00:00:00' and '2021-07-26 23:59:59')
		GROUP BY weeks,forklift_box_id 
  ) t 
GROUP BY t.weeks
```

日期格式：

```shell
格式	描述
%a	缩写星期名
%b	缩写月名
%c	月，数值
%D	带有英文前缀的月中的天
%d	月的天，数值（00-31）
%e	月的天，数值（0-31）
%f	微秒
%H	小时（00-23）
%h	小时（01-12）
%I	小时（01-12）
%i	分钟，数值（00-59）
%j	年的天（001-366）
%k	小时（0-23）
%l	小时（1-12）
%M	月名
%m	月，数值（00-12）
%p	AM 或 PM
%r	时间，12-小时（hh:mm:ss AM 或 PM）
%S	秒（00-59）
%s	秒（00-59）
%T	时间, 24-小时（hh:mm:ss）
%U	周（00-53）星期日是一周的第一天
%u	周（00-53）星期一是一周的第一天
%V	周（01-53）星期日是一周的第一天，与 %X 使用
%v	周（01-53）星期一是一周的第一天，与 %x 使用
%W	星期名
%w	周的天（0=星期日, 6=星期六）
%X	年，其中的星期日是周的第一天，4 位，与 %V 使用
%x	年，其中的星期一是周的第一天，4 位，与 %v 使用
%Y	年，4 位
%y	年，2 位
```

