## 11강. 사용자 전체의 특징과 경향 찾기
- 사용자의 속성과 경향을 알아야 함.
    - 사용자 속성: 성별, 나이 등
    - 사용자 행동: 구매 금액, 빈도, 구매 상품 등
### 사용자의 액션 집계하기
- total_uu: 전체 유니크 유저 얼마인지(session기준). session_id 의 DISTINCT COUNT.

- action_count: 총 액션별 session COUNT.
- action_uu: `액션별` DISTINCT session_id COUNT.
- usage_rate: 액션별 비중이 얼마나 되는지. action_uu / total_uu (윈도우 못 씀. COUNT 내 DISTINCT 못씀)
- count_per_user: 한 유저당 그 액션을 몇번 했는지. action_count / action_uu (토탈을 DISTINCT로 나눠준다 생각)

- 주요 로직: 전체 uu CTE와 액션별 CTE 구성 및 CROSS JOIN. 윈도우는 못씀.
```
WITH stats AS(
    SELECT COUNT(DISTINCT session) AS total_uu
    FROM action_log
    WHERE CAST(stamp AS DATE) = '2016-11-03'
)
SELECT
    action
    , COUNT(DISTINCT session) AS action_uu -- 액션별 DISTINCT 세션
    , COUNT(1) AS action_cnt -- 액션별 전체 세션
    , s.total_uu -- 전체 DISTINCT 세션
    , 100.0 * COUNT(DISTINCT session) / s.total_uu AS usage_rate -- 전체 DISTINCT 세션 중 액션별 DISTINCT 세션 비중
    , 1.0 *  COUNT(1) / COUNT(DISTINCT session) AS count_per_user -- 액션별 전체 세션을 액션별 DISTINCT 세션으로 나눠줌
FROM action_log
CROSS JOIN stats s
GROUP BY action, total_uu
```

### 로그인 사용자와 비로그인 사용자를 구분해서 집계하기
- 로얄티에 따른 경향 분석
    - 로그인 / 비로그인 구분하여 액션 분석하기
- 로직: 조건문을 활용해 결측치를 비로그인 처리. 
` CASE WHEN COALESCE(user_id,'') <> '' THEN 'login' ELSE 'guest' END `
- COALESCE + ROLLUP 조합 사용 가능. 상위 카테고리만 있고 하위 카테고리가 없을 경우, 즉 상위 카테고리 기준으로 집계할 경우 하위 카테고리가 별도 이름이 없으니 꼭 COALESCE로 별칭 설정해주기
- 쿼리
```
WITH action_with_login_status AS(
    SELECT
        session
        , user_id
        , action
        , CASE WHEN COALESCE(user_id, '')<>'' THEN 'login' ELSE 'guest' END AS login_status
    FROM action_log
)
SELECT
    COALESCE(action, 'all') AS action
    , COALESCE(login_status, 'all') AS login_status
    , COUNT(DISTINCT session) AS action_uu
    , COUNT(1) AS action_count
FROM action_with_login_status
GROUP BY ROLLUP(action, login_status)
```
- 주의!! all <> guest + login 
    - 비로그인 유저가 로그인하면 각각 액션에 1씩 추가됨. 반대로 all은 session 기반으로 집계됨. 
- 한번이라도 로그인한 유저는 회원으로 보기
    - CASE WHEN + COALESCE + MAX(user_id) OVER(PARTITION BY session ORDER BY stamp ROWS UNBOUNDED PRECEDING)
```
WITH action_with_login_status AS(
    SELECT
        session
        , user_id
        , action
        , CASE WHEN 
            COALESCE(MAX(user_id)
                OVER(PARTITION BY session ORDER BY stamp
                    ROWS UNBOUNDED PRECEDING)
                , '')<>'' 
                THEN 'member' 
            ELSE 'none' 
        END AS login_status
        , stamp
    FROM action_log
)
SELECT * FROM action_with_login_status
```
- 위 쿼리에서 만약 PARTITION BY session까지만 한다면, 세션 통틀어 MAX(user_id)를 반환할 것이고, PARTITION BY session ORDER BY stamp ROWS UNBOUNDED PRECEDING까지 포함될 경우 순차적으로 보여지기에 그때까지 로그인 안했으면 none 출력될 수 있음.

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
