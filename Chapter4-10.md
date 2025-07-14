## 10강. 다면적인 축을 사용해 데이터 집약하기
### 카테고리 매출과 소계 계산하기
- 로직: UNION ALL 
    - 가장 작은 단위까지 GROUP BY 한 후 집계 한 테이블
    - 두번째 단위까지 GROUP BY 하고 집계, 작은 단위는 'all'로 한 테이블
    - 전체 'all', 'all'로 테이블 명 규정 후 전체 대상 집계

```
SELECT
    category
    , sub_category
    , SUM(price) AS total
FROM purchase_detail_log
GROUP BY 1,2

UNION ALL

SELECT
    category
    , 'all' AS sub_category
    , SUM(price) AS total
FROM purchase_detail_log
GROUP BY 1

UNION ALL

SELECT
    'all' AS category
    , 'all' AS sub_category
    , SUM(price) AS total
FROM purchase_detail_log

ORDER BY category, sub_category;
```
- 로직: 위 과정을 GROUP BY ROLLUP(cat, sub_cat) + COALEASCE 조합으로 구현 가능.
```
SELECT
    COALESCE(category, 'all') AS category
    , COALESCE(sub_category, 'all') AS sub_category
    , SUM(price) AS total
FROM purchase_detail_log
GROUP BY ROLLUP(category, sub_category);
```

### ABC분석으로 잘 팔리는 상품 판별하기
- 재고관리에서 매출 중요성에 따라 상품을 분석하기 위해 사용
- 결과물: 카테고리, 판매금액, 구성비, 구성비누계, 등급
- 첫번째 실수: 판매금액을 구하고 판매금액/전체합계 구하고, `이것의 누계`를 구하려하니 재귀적으로 돌아가서 cte가 너무 많아짐.
- 두번째 실수: 순서 무시. `카테고리별 매출 합계의 내림차순 기준으로 누계`를 구해야 함. CTE를 해도 되고, SUM 윈도우 함수 OVER절 안에 `ORDER BY SUM(PRICE) DESC`을 해도 됨.
- 제대로 된 로직: 판매금액(내림차순), 판매금액/전체합계, `판매누계/전체합계`, 등급으로 하면 깔끔.
- CASE는 첫 매치(first match)에서 종료. 
```
SELECT *
    , CASE 
        WHEN cum_ratio <70 THEN 'A'
        WHEN cum_ratio <90 THEN 'B'
        ELSE 'C' END AS grade

FROM (
    SELECT 
        category
        , sub_category
        , SUM(price) AS total_amount
        , 100.0 * SUM(price) / SUM(SUM(price)) OVER() AS composition_ratio
        , 100.0 * SUM(SUM(price)) OVER(ORDER BY SUM(price) DESC ROWS UNBOUNDED PRECEDING) 
        / SUM(SUM(price)) OVER() AS cumulative_ratio
    FROM purchase_detail_log
    GROUP BY 1,2
    ORDER BY ratio DESC
) tmp
```
### 팬 차트
- 팬 차트는 특정 시점을 100으로 두고 이후의 숫자 변동을 확인할 수 있게 해주는 그래프.
- 팬 차트는 백분율을 사용해 작은 변화도 쉽게 인지할 수 있게 해줌.
- 로직: 시간, 카테고리 순으로 정렬, First_value를 사용해 기준점을 잡아준 뒤 나눠줌
- 실수: 오버절에 반드시 `정렬조건` 추가!!!! FIRST_VALUE OVER(PARTITION BY ORDER BY)
```
SELECT
    CONCAT(order_year, "-" ,order_month) AS order_ym
    , category
    , total
    , FIRST_VALUE(total) OVER(PARTITION BY category ORDER BY order_year, order_month ) AS fst_value
    , 100.0 * total/FIRST_VALUE(total) OVER(PARTITION BY category) AS ratio
FROM (
    SELECT
        YEAR(CAST(dt AS DATETIME)) AS order_year
        , MONTH(CAST(dt AS DATETIME)) AS order_month
        , category
        , SUM(price) AS total
    FROM purchase_detail_log
    GROUP BY 1,2,3
    ORDER BY 1,2,3
) AS tmp
ORDER BY order_ym, category
```
