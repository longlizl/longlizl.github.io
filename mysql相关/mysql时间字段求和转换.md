```mysql
# TIME_TO_SEC 时间格式转秒
# SEC_TO_TIME 秒转时间格式
SELECT
    fms_forklift_id,
    SEC_TO_TIME(SUM(TIME_TO_SEC(travel_time)))运行总时长 ,
    SEC_TO_TIME(SUM(TIME_TO_SEC(load_time))) 负载总时长,
    SEC_TO_TIME(SUM(TIME_TO_SEC(move_time))) 移动总时长,
    SEC_TO_TIME(SUM(TIME_TO_SEC(load_move_time))) 负载移动总时长,
    SEC_TO_TIME(SUM(TIME_TO_SEC(idle_time))) 空闲总时长
FROM
    fms_forklift_device_run_record
WHERE
    start_time BETWEEN '2021-07-28 00:00:00'
    AND '2021-07-28 23:59:59'
    AND fms_forklift_id = 159
GROUP BY fms_forklift_id
```

