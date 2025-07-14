## 9강. 시계열 기반으로 데이터 집계하기
### 날짜별 매출 집계하기
- 로직: GROUP BY + SUM 활용한 집계
- 테이블: purchase_log 테이블: dt, order_id, user_id, purchase_amount

```
SELECT
    dt
    , COUNT(*) AS purchase_cnt
    , SUM(purchase_amount) AS total_amt
    , AVG(purchase_amount) AS avg_amt
    FROM purchage_log
    GROUP BY dt
    ORDER BY dt
```

### 이동평균을 사용한 날짜별 평균 보기
- 로직: AVG() OVER() 윈도우 함수 활용 집계.
    - 더 정교하게 하기 위해 CASE WHEN 7 = COUNT(*) OVER ()로 조건을 줄 수 있음.
- 시즈널리티가 있으면 트렌드를 확인하기 어려움. 이를 위해 이동평균을 활용할 수 있음.
- 참고: 윈도우 함수 안에 집계함수 사용 가능(예컨대 주문테이블에서 일간평균 가격의 이동평균을 구할 경우)
- 예제: 날짜별로 합계한 값에 대해 7일 이동 평균을 구하는 쿼리
```
SELECT 
    dt 
    , SUM(purchase_amount) AS purchase_tot
    , CASE 
        WHEN 
            7 = COUNT(*) 
            OVER(ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
        THEN
            AVG(SUM(purchase_amount)) 
            OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) END AS seven_day_avg_amt
FROM purchase_log
GROUP BY dt
ORDER BY dt;
```

### 매출 누계 구하기
- 로직: 
    - 윈도우 함수로 누계 구하기. 월별 SUM(매출) OVER(PARTITION BY 연,월 ORDER BY 연,월 ROWS UNBOUNDED PRECEDING).
    - 날짜는 SUBSTRING 혹은 YEAR/MONTH/DAY(CAST(~~ AS DATE))로 날짜 구분. CONCAT(ord_year, "-", ord_month)도 가능.
    - order raw의 경우 한 일자에도 여러 로우가 있기에 일자별 합산인 SUM(purchase_amount) 로 한 핏이 나올 수 있고 일자별 합산의 누적인 SUM(SUM(purchase_amount)) OVER(ROWS UNBOUNDED PRECEDING)이 나올 수 있음. 모양이 이상해서 헷갈렸음. 윈도우 안에 집계함수를 상수취급하며 넣어주는 형태임.
- 쿼리:
```
SELECT
    dt
    , SUM(purchase_amount)
    , SUM(SUM(purchase_amount)) OVER(ROWS UNBOUNDED PRECEDING) AS tot
FROM purchase_log
GROUP BY dt
```


### 월별 매출의 작대비 구하기
- 로직: 
    - CTE로 year, month를 나눈 테이블 생성 후 메인쿼리에서 CASE WHEN으로 연도별 분리 후 집계.
    - **메인 쿼리에서 CASE WHEN 을 다시 SUM으로 묶어주고 GROUP BY 시키면 tidy해짐!!!!!!**
```
WITH daily_purchase AS(
    SELECT
        dt
        , YEAR(CAST(dt AS DATE)) AS yy
        , MONTH(CAST(dt AS DATE)) AS mm
        , DAY(CAST(dt AS DATE)) AS dd
        , SUM(purchase_amount) AS daily_amt
    FROM purchase_log
    GROUP BY dt
)
SELECT
    mm
    , SUM(CASE WHEN yy = 2015 THEN daily_amt END) AS '2015'
    , SUM(CASE WHEN yy = 2014 THEN daily_amt END) AS '2014'
FROM daily_purchase
GROUP BY mm
```
![alt text](image.png)
![alt text](image-1.png)

### z차트로 업적 추이 확인
- 계절에 따른 매출 변동은 월차매출, 매출누계, 이동년계로 구성된 Z차트를 통해 시즈널리티를 배제하고 트렌드를 분석할 수 있음
    - 월차매출: 월 매출
    - 매출 누계: 기준년도 월 매출 누계
    - 이동년계: 최근 12개월 누계
- 로직: 매출 누계 시 윈도우 함수 안에 기준연도를 CASE WHEN 으로 조건문으로 넣을 수 있음. 
- 실수: 윈도우 함수 SUM 함수 내부에 CASE 식을 넣을 수 있음.
- 쿼리
```
/*z차트를 만들어 보자
    - 월 매출
    - 월 누계 매출 (특정년도 기준)
    - 이동년계 매출*/
WITH daily_purchase AS(
    SELECT
        dt
        , YEAR(CAST(dt AS DATE)) AS yy
        , MONTH(CAST(dt AS DATE)) AS mm
        , DAY(CAST(dt AS DATE)) AS dd
        , CONCAT(YEAR(CAST(dt AS DATE)), "-", MONTH(CAST(dt AS DATE))) AS yymm
        , SUM(purchase_amount) AS daily_amt
    FROM purchase_log
    GROUP BY 1,2,3,4,5
)
, monthly_purchase AS(
    SELECT
        yy
        , mm
        , SUM(daily_amt) AS amt
    FROM daily_purchase
    GROUP BY yy, mm
)
, cal_index AS(
    SELECT
        yy
        , mm
        , amt
        , SUM(CASE WHEN yy='2015' THEN amt END) 
            OVER(ORDER BY yy, mm ROWS UNBOUNDED PRECEDING) AS agg_amt
        , SUM(amt)
            OVER(ORDER BY yy, mm ROWS BETWEEN 11 PRECEDING AND CURRENT ROW) AS year_avg_amt
    FROM
        monthly_purchase
    ORDER BY yy, mm
)
SELECT * FROM cal_index
```

```
/*z차트를 만들어 보자
    - 월 매출
    - 월 누계 매출 (특정년도 기준)
    - 이동년계 매출*/
    
WITH daily_purchase AS(
    SELECT
        dt
        , YEAR(CAST(dt AS DATE)) AS yy
        , MONTH(CAST(dt AS DATE)) AS mm
        , DAY(CAST(dt AS DATE)) AS dd
        , CONCAT(YEAR(CAST(dt AS DATE)), "-", MONTH(CAST(dt AS DATE))) AS yymm
        , SUM(purchase_amount) AS daily_amt
        , COUNT(*) AS daily_cnt
    FROM purchase_log
    GROUP BY 1,2,3,4,5
)
, monthly_purchase AS(
    SELECT
        yy
        , mm
        , SUM(daily_amt) AS monthly_amt
        , SUM(daily_cnt) AS monthly_cnt
    FROM daily_purchase
    GROUP BY yy, mm
)
SELECT
    CONCAT(yy, "-", mm) AS yy_mm
    , monthly_amt
    , LAG(monthly_amt, 12) OVER(ORDER BY yy,mm) AS 1y_ago
    , 100 * monthly_amt / LAG(monthly_amt, 12) OVER(ORDER BY yy,mm) AS yoy
FROM monthly_purchase;
```



### 매출을 파악할 때 중요 포인트
- 매출 집계만으론 상승/하락이라는 결과만 알 수 있음.
- 상승/하락의 원인 파악을 위해선 주변 데이터를 고려해야 함.
    - 예시:  구매 횟수, 구매 단가, 방문횟수, 회원 등록 수

    - 매출 변화 일정
        - 판매 횟수 증가
        - 평균 구매 감소
    - 매출 하락
        - 판매 횟수 일정
        - 평균 구매액 감소
- 로직: 
    - order_amount의 sum, avg, 윈도우 sum, lag, 
    - order_id의 cnt
- 쿼리: LAG(monthly_amt, 12) OVER(ORDER BY yy,mm)
```
WITH monthly_total AS(
    SELECT
        YEAR(CAST(dt AS DATETIME)) AS order_year
        , MONTH(CAST(dt AS DATETIME)) AS order_month
        , AVG(purchase_amount) AS monthly_avg
        , SUM(purchase_amount) AS monthly_purchase
        , COUNT(DISTINCT order_id) AS monthly_cnt
    FROM purchase_log
    GROUP BY 1,2
)
SELECT
    CONCAT(order_year, "-", order_month) AS "판매월"
    , monthly_avg AS "평균 구매액"
    , monthly_cnt AS "판매 횟수"
    , monthly_purchase AS "매출액"
    , SUM(monthly_purchase) OVER(ROWS UNBOUNDED PRECEDING) AS "누계 매출액"
    , LAG(monthly_purchase, 12) OVER(ORDER BY order_year, order_month) AS "전년 매출액" 
    , 100.0 * monthly_purchase / LAG(monthly_purchase, 12) OVER(ORDER BY order_year, order_month) AS "작대비"
FROM monthly_total;
```
