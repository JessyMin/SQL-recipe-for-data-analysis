## 9. 시계열 기반으로 데이터 집계하기
<br>

### 9-1. 날짜별 매출 집계하기
```sql
SELECT
  dt,
  COUNT(order_id) AS purchase_count,
  SUM(purchase_amount) AS total_amount,
  AVG(purchase_amount) AS avg_amount
FROM purchase_log
GROUP BY dt
ORDER BY dt;
```

* `COUNT(*)`와 `COUNT(order_id)`의 차이는?
<br>

### 9-2. 이동평균을 사용한 날짜별 추이 보기
```sql
SELECT
  dt,
  SUM(purchase_amount) AS total_amount,
  -- 최근 최대 7일 동안의 평균 계산
  AVG(SUM(purchase_amount))
    OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
  AS seven_day_avg_amount,
  -- 최근 최대 7일 동안의 평균, 7일이 모두 존재하는 경우만 계산
  CASE
    WHEN 7 = COUNT(*)
      OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
    THEN AVG(SUM(purchase_amount))
      OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
    ELSE ''
  END AS seven_day_avg_amount_strict,
FROM purchase_log
GROUP BY dt
ORDER BY dt;
```

```sql
--MySQL 8.0
SELECT
  p1.dt,
  p1.daily_amount AS total_amount,
  SUM(p2.daily_amount)/COUNT(DISTINCT p2.dt) AS seven_day_avg_amount,
  CASE
    WHEN COUNT(p2.dt) = 7
      THEN SUM(p2.daily_amount)/COUNT(DISTINCT p2.dt)
	  ELSE ''
  END AS seven_day_avg_amount_strict   
FROM (
      SELECT
        dt,
        SUM(purchase_amount) AS daily_amount
      FROM purchase_log
      GROUP BY dt
     ) p1
JOIN (
      SELECT
        dt,
        SUM(purchase_amount) AS daily_amount
      FROM purchase_log
      GROUP BY dt
     ) p2
  ON DATEDIFF(p1.dt, p2.dt) BETWEEN 0 AND 6
GROUP BY p1.dt
ORDER BY p1.dt;
```
<br>

### 9-3. 당월 매출 누계 구하기
날짜별 매출 및 해당 월에 누적된 매출을 동시에 확인할 수 있는 리포트
```sql
WITH
daily_purchase AS (
  SELECT
    dt,
    SUBSTR(dt, 1, 4) AS years,
    SUBSTR(dt, 6, 2) AS months,
    SUBSTR(dt, 9, 2) AS dates,
    SUM(purchase_amount) AS purchase_amount
  FROM purchase_log
  GROUP BY dt
  ORDER BY dt
)  
SELECT
  dt,
  CONCAT(years, '-', months) AS year_month,
  purchase_amount,
  SUM(purchase_amount)
    OVER (PARTITION BY years, months ORDER BY dt ROWS UNBOUNDED PRECEDING)
  AS agg_amount
FROM daily_purchase
ORDER BY dt;
```
* `date` 자료형에도 `SUBSTRING`이 가능하다.
* 책에서는 필요할 때마다 `SUBSTR(dt, 1, 7)` 또는 `DATE_FORMAT(dt, '%Y-%m')`로 `year_month`를 추출하는 대신, 가독성을 위해 임시 테이블을 만들면서 날짜를 `year`/`month`/`date`로 분할해 놓고, 다시 `CONCAT`로 결합해서 사용하고 있다.

"서비스를 운용, 개발하기 위해 사용하는 SQL과 비교했을 때, 빅데이터 분석 SQL은 성능이 조금 떨어지더라도 가독성과 재사용성을 중시해서 작성하는 경우가 많습니다."   - 147p.

<br>

### 9-4. 월별 매출의 전년 동월 대비 구하기

```sql
WITH daily_purchase AS (
  SELECT
    dt,
    SUBSTR(dt, 1, 4) AS years,
    SUBSTR(dt, 6, 2) AS months,
    SUBSTR(dt, 9, 2) AS dates,
    SUM(purchase_amount) AS purchase_amount
  FROM purchase_log
  GROUP BY dt
  ORDER BY dt
)
SELECT
  months,
  SUM(CASE WHEN years='2014' THEN purchase_amount END) AS amount_2014,
  SUM(CASE WHEN years='2015' THEN purchase_amount END) AS amount_2015,
  ROUND(100
    * SUM(CASE WHEN years='2014' THEN purchase_amount END)
    /SUM(CASE WHEN years='2015' THEN purchase_amount END), 2)
FROM daily_purchase
GROUP BY months
ORDER BY months;
```

이 실습을 통해 SUM() 내에 CASE WHEN을 쓸 수 있다는 걸 알게 되었다.
아래는 처음에 짰던 시행착오 쿼리이다.

```sql
WITH daily_purchase AS (
  SELECT
    dt,
    SUBSTR(dt, 1, 4) AS years,
    SUBSTR(dt, 6, 2) AS months,
    SUBSTR(dt, 9, 2) AS dates,
    SUM(purchase_amount) AS purchase_amount
  FROM purchase_log
  GROUP BY dt
  ORDER BY dt
)
SELECT
  months,
  SUM(amount_2014) AS amount_2014,
  SUM(amount_2015) AS amount_2015,
  ROUND(SUM(amount_2014)/SUM(amount_2015)*100, 2) AS rate
FROM (
      SELECT
        months,
        CASE WHEN years = '2014' THEN purchase_amount
        END AS amount_2014,
        CASE WHEN years = '2015' THEN purchase_amount
        END AS amount_2015  
      FROM daily_purchase
      ) tmp
GROUP BY months
ORDER BY months;
```
<br>

### 9-5. Z 차트로 업적의 추이 확인하기
- Z 차트 : 판매 데이터 분석 기법의 하나. 계절에 따라 매출이 변동하는 경우 Z차트를 이용해 계절 변동의 영향을 배제하고 매출 추이를 분석함.
- 구성요소 :
  1) 월별매출 : 각 월의 매출
  2) 누계매출 : 해당월을 포함한 당해년도 매출 누계
  3) 이동년계: 해당월을 포함한 과겨 1년간 매출의 누계
- 참고 : <a href='http://jungwoo5394.blogspot.kr/2015/05/z-z-chart.html'>차트 읽는 법</a>

``` sql  

```


### 9-6. 매출을 파악할 때 중요 포인트

- 매출 집계와 함께 관련 데이터를 분석해야 상승/하락의 원인을 알 수 있음
- ex) 매출 증가 <- 판매횟수 증가
  - 판매횟수 증가 <- 방문횟수, 상품 수, 회원등록 수 변화
- ex) 평균 구매액 감소 <- 기간 내 판매된 상품내역 확인
- 한 쿼리를 여러 용도로 사용할 수 있게 해두면 활용도가 높아짐
