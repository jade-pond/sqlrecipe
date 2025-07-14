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

# 연령별 구분 집계하기
- 사용자를 파악해 서비스가 의도한대로 사용되는지 확인해야 함. 
- 광고 디자인, 캐치 프레이즈 검토를 위해 속성을 집계해야 함.
- 연령 구하기 방법 2가지
    - 책: 현재 date, birth_date를 string -> unsigned로 바꾼 후 10000 나누고 FLOOR 해주기
- 연령, 성별 기반 고객 구분
    - TIP: 작은 것 비교할 땐 작은 것 먼저. SQL은 분기 처리.
```
SELECT
    user_id
    , sex
    , user_age
    , CONCAT(
        CASE WHEN 20 <= user_age THEN sex ELSE '' END
        ,
        CASE 
            WHEN user_age < 13 THEN 'C'
            WHEN user_age < 20 THEN 'T'
            WHEN user_age < 35 THEN '1'
            WHEN user_age < 50 THEN '2'
            ELSE 3 END ) AS category
FROM (
    SELECT
        *
        , TIMESTAMPDIFF(YEAR, birth_date, CURDATE()) AS user_age
    FROM mst_users) AS tmp
```
- 서브 쿼리를 중첩해서 사용 가능
```
SELECT category, COUNT(*)
FROM (
  SELECT user_id, sex, age,
         CASE WHEN age < 18 THEN 'teen' 
              WHEN age < 40 THEN 'adult' 
              ELSE 'senior' END AS category
  FROM (
    SELECT user_id, sex,
           TIMESTAMPDIFF(YEAR, birth_date, CURDATE()) AS age
    FROM users
  ) AS base
) AS categorized
GROUP BY category;
```
### 연령별 구분의 특징 추출하기
- 사용자 속성에 따라 서비스 이용 형태가 다를 수 있음. 
    - 연령별, 성별 구분을 통해 각각 구매한 상품의 카테고리를 집계하면 새로운 리포트를 만들 수 있음.
```
SELECT
    user_category
    , category AS product_category
    , COUNT(*) AS purchase_count
FROM (
    SELECT
        user_id
        , sex
        , CONCAT(
            CASE WHEN 20 <= user_age THEN sex ELSE '' END
            ,
            CASE 
                WHEN user_age < 4 THEN NULL
                WHEN user_age < 13 THEN 'C'
                WHEN user_age < 20 THEN 'T'
                WHEN user_age < 35 THEN '1'
                WHEN user_age < 50 THEN '2'
                ELSE 3 END ) AS user_category
    FROM (
        SELECT 
            user_id
            ,sex
            ,birth_date
            , TIMESTAMPDIFF(YEAR, birth_date, CURDATE()) AS user_age
        FROM mst_users) AS tmp1
    
    ) AS tmp2
JOIN action_log a on tmp2.user_id = a.user_id
WHERE action= 'purchase'
GROUP BY 1,2
```
### 사용자의 방문 빈도 집계하기
- 7일 간 N번 방문한 유저 구하기
```
/*한 주에 며칠 사용되었는지 집계하는 쿼리*/
SELECT 
    cnt_day
    , COUNT(user_id) AS cnt_user
    , 100.0 * COUNT(user_id) / SUM(COUNT(user_id)) OVER() AS composition_ratio
    , 100.0 * SUM(COUNT(user_id)) OVER(ORDER BY cnt_day ROWS UNBOUNDED PRECEDING) / SUM(COUNT(user_id)) OVER() AS cumulative_ratio
FROM (
    SELECT
        user_id
        ,  COUNT(DISTINCT action_date) AS cnt_day
    FROM (
        SELECT 
            user_id
            , CAST(stamp AS DATE) AS action_date
        FROM action_log
    ) AS tmp
    WHERE action_date BETWEEN '2016-11-01' AND '2016-11-08'
    GROUP BY user_id
) AS tmp2
GROUP BY cnt_day
```
### 벤 다이어그램으로 사용자 액션 집계하기
- has_favorite, has_purchase, has_review
- CUBE 혹은 UNION ALL 사용
- 로직: USER ID 기준으로 CASE WHEN 조건문을 통해 액션을 1 해줌 > SUM > SIGN
```
/*벤 다이어그램으로 나타낸 액션*/
SELECT
    COALESCE(has_favorite, NULL) AS has_favorite
    , COALESCE(has_purchase, NULL) AS has_purchase
    , COALESCE(has_review, NULL) AS has_review
    , COUNT(1)
FROM
    (SELECT
        user_id
        ,SIGN(SUM(CASE WHEN action = 'review' THEN 1 ELSE 0 END)) AS has_review
        ,SIGN(SUM(CASE WHEN action = 'add_cart' THEN 1 END)) AS  has_favorite
        ,SIGN(SUM(CASE WHEN action = 'purchase' THEN 1 END)) AS has_purchase
    FROM action_log
    GROUP BY user_id) tmp
GROUP BY CUBE(has_favorite, has_purchase, has_review)
```

- 시각화 용
```
WITH action_flag AS( 
    SELECT
        user_id
        , SIGN(SUM(CASE WHEN action = 'add_cart' THEN 1 ELSE 0 END)) AS has_add_cart
        , SIGN(SUM(CASE WHEN action = 'purchase' THEN 1 ELSE 0 END)) AS has_purchase
        , SIGN(SUM(CASE WHEN action = 'review' THEN 1 ELSE 0 END)) AS has_review
    FROM action_log
    GROUP BY user_id
)
SELECT
    CONCAT('add_cart_', has_add_cart, '_purchase_', has_purchase, '_review_', has_review) AS action_group,
    COUNT(*) AS users
FROM action_flag
GROUP BY has_add_cart, has_purchase, has_review
ORDER BY users DESC;
```
- UNION ALL
```
WITH action_flag AS( 
    SELECT
        user_id
        , SIGN(SUM(CASE WHEN action = 'add_cart' THEN 1 ELSE 0 END)) AS has_add_cart
        , SIGN(SUM(CASE WHEN action = 'purchase' THEN 1 ELSE 0 END)) AS has_purchase
        , SIGN(SUM(CASE WHEN action = 'review' THEN 1 ELSE 0 END)) AS has_review
    FROM action_log
    GROUP BY user_id
)
,
action_venn_diagram AS(
    -- 1. Full combination (3 cols)
    SELECT has_add_cart, has_purchase, has_review, COUNT(1) AS users FROM action_flag GROUP BY has_add_cart, has_purchase, has_review

    UNION ALL
    -- 2. Two-column combinations
    SELECT has_add_cart, has_purchase, NULL, COUNT(1) AS users FROM action_flag GROUP BY has_add_cart, has_purchase

    UNION ALL
    SELECT has_add_cart, NULL, has_review, COUNT(1) AS users FROM action_flag GROUP BY has_add_cart, has_review

    UNION ALL
    SELECT NULL, has_purchase, has_review, COUNT(1) AS users FROM action_flag GROUP BY has_purchase, has_review
    -- 3. One-column combinations
    UNION ALL
    SELECT has_add_cart, NULL, NULL, COUNT(1) AS users FROM action_flag GROUP BY has_add_cart

    UNION ALL
    SELECT NULL, has_purchase, NULL, COUNT(1) AS users FROM action_flag GROUP BY has_purchase

    UNION ALL
    SELECT NULL, NULL, has_review, COUNT(1) AS users FROM action_flag GROUP BY has_review
    -- 4. Total summary row
    UNION ALL
    SELECT NULL, NULL, NULL, COUNT(1) AS users FROM action_flag

    ORDER BY has_add_cart, has_purchase, has_review
)
SELECT 
    CASE has_add_cart 
        WHEN 1 THEN 'add_cart' WHEN 0 THEN 'not_cart' ELSE 'any'
    END AS has_add_cart
    , CASE has_purchase
        WHEN 1 THEN 'purchased' WHEN 0 THEN 'not_purchased' ELSE 'any'
    END AS has_purchase
    , CASE has_review
        WHEN 1 THEN 'reviewed' WHEN 0 THEN 'not_reviewd' ELSE 'any'
    END AS has_review
    , users
    , 100.0 * users / NULLIF(
        SUM(CASE WHEN has_purchase IS NULL
                AND has_review IS NULL
                AND has_add_cart IS NULL
                THEN users ELSE 0 END) OVER(), 0) AS ratio
FROM action_venn_diagram
ORDER BY has_purchase, has_review, has_add_cart;
```

- 이슈: 속성이 4개 이상되면 해석 어려움
    - **해결1: 행동을 우선 순위별로 묶음**
        - 관심행동: add_carte, favorite, view
        - 전환행동: subscribe, purchase
        - 관여행동: review, share
    - **해결2: 상위 80% 행동만 선별 (cumulative ratio)**
    - **해결3: python**
        - upset plot, binary matrix heatmap
- 팁(SNS)
    - 게시글 작성, 게시글 열람, 댓글 작성 여부에 따른 분석


### Decile 분석
- 사용자를 구매 금액 DESC 순 정렬
- 정렬된 상위 사용자로부터 10%씩 Decile 1부터 10까지 그룹 할당
- Decile별 구매 금액 합계 / 전체
- Decile별 누적 구매 금액 합계 / 전체
```
WITH user_purchase_amt AS(
    SELECT
        user_id
        , SUM(amount) AS purchase_amount
    FROM action_log
    WHERE action = 'purchase'
    GROUP BY user_id
)
, user_with_decile AS(
    SELECT
        user_id
        , purchase_amount
        , ntile(10) OVER(ORDER BY purchase_amount DESC) AS decile
    FROM user_purchase_amt
)
, decile_with_purchase_amount AS(
    SELECT
        decile
        , SUM(purchase_amount) AS amount
        , AVG(purchase_amount) AS avg_amount
        , SUM(SUM(purchase_amount)) OVER(ORDER BY decile ROWS UNBOUNDED PRECEDING) AS cumulative_amount
        , SUM(SUM(purchase_amount)) OVER() AS total_amount
    FROM user_with_decile
    GROUP BY decile
)
SELECT
    decile
    , amount
    , avg_amount
    , 100.0 * amount / total_amount AS composition_ratio
    , 100.0 * cumulative_amount / total_amount AS cumulative_ratio
FROM decile_with_purchase_amount
```

- RFM 분석
    - Recency: 최근 구매일
    - Frequency: 구매 횟수
    - Monetary: 구매 금액 합계

```
WITH
    purchase_log AS(
        SELECT
            user_id
            , amount
            , SUBSTRING(stamp, 1, 10) AS dt 
        FROM 
            action_log
        WHERE 
            action = 'purchase'
    )
, user_rfm AS(
    SELECT
        user_id
        , MAX(dt) AS recent_date
        , DATEDIFF(CAST('2016-11-20' AS DATETIME), MAX(CAST(dt AS DATETIME))) AS recency
        , COUNT(dt) AS frequency
        , SUM(amount) AS monetary
    FROM
        purchase_log
    GROUP BY 
        user_id
)
, user_rfm_rank AS(
    SELECT
        user_id
        , recent_date
        , recency
        , frequency
        , monetary
        , CASE 
            WHEN recency < 14 THEN 5
            WHEN recency < 28 THEN 4
            WHEN recency < 60 THEN 3
            WHEN recency < 90 THEN 2
            ELSE  1
        END AS r
        , CASE 
            WHEN frequency >= 5 THEN 5
            WHEN frequency >= 4 THEN 4
            WHEN frequency >= 3 THEN 3
            WHEN frequency >= 2 THEN 2
            WHEN frequency = 1 THEN 1
            END AS f
        , CASE
            WHEN monetary >= 30000 THEN 5
            WHEN monetary >= 10000 THEN 4
            WHEN monetary >= 3000 THEN 3
            WHEN monetary >= 500 THEN 2
            ELSE 1 END AS m
    FROM user_rfm
)
, mst_rfm_index AS(
    SELECT 1 AS rfm_index
    UNION ALL SELECT 2 AS rfm_index
    UNION ALL SELECT 3 AS rfm_index
    UNION ALL SELECT 4 AS rfm_index
    UNION ALL SELECT 5 AS rfm_index
)
, rfm_flag AS(
    SELECT
        m.rfm_index
        , CASE WHEN m.rfm_index = r.r THEN 1 ELSE 0 END AS r_flag
        , CASE WHEN m.rfm_index = r.f THEN 1 ELSE 0 END AS f_flag
        , CASE WHEN m.rfm_index = r.m THEN 1 ELSE 0 END AS m_flag
    FROM mst_rfm_index AS m 
    CROSS JOIN user_rfm_rank AS r
)
-- SELECT
--     rfm_index
--     , SUM(r_flag) AS r
--     , SUM(f_flag) AS f
--     , SUM(m_flag) AS m
-- FROM
--     rfm_flag
-- GROUP BY 
--     rfm_index
-- ORDER BY
--     rfm_index DESC;
SELECT
    r + f + m  AS total_rank
    , r, f, m
    , COUNT(user_id)
FROM 
    user_rfm_rank
GROUP BY 
    r, f, m
ORDER BY
    1 DESC, 2 DESC, 3 DESC, 4 DESC;
```
