```mysql
-- 它会删除重复的记录（会保留一条），然后建立唯一索引，高效而且人性化。（注mysql5.732版本以上语法无效）
alter ignore table t_aa add unique index index_name (aa,bb);

-- 查询大于一条的重复记录
SELECT 
	* 
from t_event
WHERE id not in(
	SELECT min_id from (SELECT MIN(id) as min_id from t_event GROUP BY time,type1,type2,forklift_box_id) t
)

-- 删除重复并保留id值最小的记录
DELETE from t_event
WHERE id not in(
	SELECT min_id from (SELECT MIN(id) as min_id from t_event GROUP BY time,type1,type2,forklift_box_id) t
)

-- 添加联合唯一索引(索引名可以不写)
alter table fms_forklift_device_run_record add unique index index_name (fms_forklift_id,start_time)
ALTER TABLE fms_forklift_device_run_record ADD UNIQUE INDEX (fms_forklift_id,start_time)
```

