## 12강. 시계열에 다른 사용자 전체의 상태 변화 찾기
### 지속률과 정착률 산출하기
- 테이블
    - mst_users: user_id, register_date, ..
    - action_log: user_id, action_date, ...
- 쿼리
```
WITH repeat_interval AS(
    SELECT '01 day repeat' AS index_name, 1 AS interval_date
    UNION ALL
    SELECT '02 day repeat' AS index_name, 2 AS interval_date
    UNION ALL
    SELECT '03 day repeat' AS index_name, 3 AS interval_date
    UNION ALL
    SELECT '04 day repeat' AS index_name, 4 AS interval_date
    UNION ALL
    SELECT '05 day repeat' AS index_name, 5 AS interval_date
    UNION ALL
    SELECT '06 day repeat' AS index_name, 6 AS interval_date
    UNION ALL
    SELECT '07 day repeat' AS index_name, 7 AS interval_date
),
action_log_with_index_date AS(
    SELECT 
        m.user_id
        , register_date
        , CAST(a.stamp AS DATE) AS action_date
        , MAX(CAST(a.stamp AS DATE)) OVER() AS latest_date
        , r.index_name
        , DATE_ADD(CAST(register_date AS DATE), INTERVAL 1*r.interval_date DAY) AS index_date
    FROM mst_users m
    LEFT JOIN action_log a ON m.user_id = a.user_id
    CROSS JOIN repeat_interval r
    ORDER BY register_date
),
user_action_flag AS(
    SELECT
        user_id
        , register_date
        , index_name
        , SIGN(
            SUM(
                CASE WHEN index_date <= latest_date THEN
                    CASE WHEN action_date = index_date THEN 1 ELSE 0 END
                    END
            )
        ) AS index_date_action
    FROM action_log_with_index_date
    GROUP BY user_id, register_date, index_name
)
SELECT
    register_date
    , index_name
    , AVG(index_date_action) AS repeat_ratio
FROM user_action_flag
GROUP BY register_date, index_name
ORDER BY 3 DESC;

-- SELECT 
--     DISTINCT(m.user_id)
--     , CAST(register_date AS DATE) AS register_date
--     , CAST(stamp AS DATE) AS action_date
--     , CAST(stamp AS DATE) - CAST(register_date AS DATE)  AS substract
-- FROM mst_users m
-- JOIN action_log a ON m.user_id = a.user_id
-- ORDER BY 4 DESC
```
- 실수한 부분: 
    - CAST(stamp AS DATE) 해야하는데, CAST(stamp) AS DATE 처럼 괄호 밖에 씀
    - DATE_ADD(CAST(register_date AS DATE), INTERVAL 1 DAY) 로 써야 하는데, INTERVAL DAY 생략함
    - **DATE_ADD(여기)에 register_date 기준으로 해야하는데 action_date를 넣었음**
