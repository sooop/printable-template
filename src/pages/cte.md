---
title: SQLite CTE 완전 가이드
layout: '../layout/Base.astro'
---
# SQLite CTE(Common Table Expression) 완전 가이드

## 목차
1. [CTE란 무엇인가?](#cte란-무엇인가)
2. [기본 문법](#기본-문법)
3. [CTE의 장점](#cte의-장점)
4. [실전 예제](#실전-예제)
5. [다중 CTE](#다중-cte)
6. [재귀 CTE](#재귀-cte)
7. [CTE vs 서브쿼리](#cte-vs-서브쿼리)
8. [실무 활용 패턴](#실무-활용-패턴)

---

## CTE란 무엇인가?

CTE(Common Table Expression)는 쿼리 범위 내에서 정의되는 **임시 결과 집합**입니다. 쿼리가 실행되는 동안만 존재하며, 쿼리가 끝나면 사라집니다.

### 주요 특징
- 쿼리의 가독성을 크게 향상
- 복잡한 쿼리를 논리적 단위로 분해
- 서브쿼리를 대체하여 더 명확한 코드 작성
- 재귀 쿼리 작성 가능

---

## 기본 문법

```sql
WITH cte_name AS (
    -- CTE 쿼리 정의
    SELECT column1, column2
    FROM table_name
    WHERE condition
)
-- CTE를 사용하는 메인 쿼리
SELECT * FROM cte_name;
```

### 구조 설명
1. **WITH 절**: CTE 정의 시작
2. **cte_name**: CTE의 이름 지정
3. **AS (...)**: CTE를 생성하는 쿼리
4. **메인 쿼리**: CTE를 테이블처럼 참조

---

## CTE의 장점

### 1. 가독성 향상
복잡한 중첩 서브쿼리를 명확한 단계로 분리:

```sql
-- 서브쿼리 방식 (복잡함)
SELECT *
FROM (
    SELECT *
    FROM (
        SELECT customerid, SUM(total) as sum_total
        FROM invoices
        GROUP BY customerid
    ) AS sub1
    WHERE sum_total > 100
) AS sub2;

-- CTE 방식 (명확함)
WITH customer_totals AS (
    SELECT customerid, SUM(total) as sum_total
    FROM invoices
    GROUP BY customerid
),
high_value_customers AS (
    SELECT *
    FROM customer_totals
    WHERE sum_total > 100
)
SELECT * FROM high_value_customers;
```

### 2. 재사용성
동일한 CTE를 여러 번 참조 가능:

```sql
WITH sales_summary AS (
    SELECT productid, SUM(quantity) as total_qty
    FROM sales
    GROUP BY productid
)
SELECT
    a.productid,
    a.total_qty,
    a.total_qty * 100.0 / (SELECT SUM(total_qty) FROM sales_summary) as percentage
FROM sales_summary a
WHERE a.total_qty > (SELECT AVG(total_qty) FROM sales_summary);
```

### 3. 유지보수성
각 단계를 독립적으로 테스트하고 수정 가능

---

## 실전 예제

### 예제 1: 기본 사용

```sql
-- 상위 5개 트랙 조회
WITH top_tracks AS (
    SELECT trackid, name, albumid
    FROM tracks
    ORDER BY trackid
    LIMIT 5
)
SELECT
    t.trackid,
    t.name,
    a.title as album_title
FROM top_tracks t
LEFT JOIN albums a ON t.albumid = a.albumid;
```

### 예제 2: 집계와 조인

```sql
-- 고객별 총 구매액 계산 (상위 5명)
WITH customer_sales AS (
    SELECT
        c.customerid,
        c.firstname || ' ' || c.lastname AS customer_name,
        ROUND(SUM(ii.unitprice * ii.quantity), 2) AS total_sales
    FROM customers c
    INNER JOIN invoices i ON c.customerid = i.customerid
    INNER JOIN invoice_items ii ON i.invoiceid = ii.invoiceid
    GROUP BY c.customerid, c.firstname, c.lastname
)
SELECT
    customer_name,
    total_sales,
    RANK() OVER (ORDER BY total_sales DESC) as sales_rank
FROM customer_sales
ORDER BY total_sales DESC
LIMIT 5;
```

### 예제 3: 필터링과 변환

```sql
-- 평균 이상 판매 아티스트 찾기
WITH artist_sales AS (
    SELECT
        ar.artistid,
        ar.name as artist_name,
        COUNT(DISTINCT t.trackid) as track_count,
        SUM(ii.quantity) as total_sold
    FROM artists ar
    JOIN albums al ON ar.artistid = al.artistid
    JOIN tracks t ON al.albumid = t.albumid
    JOIN invoice_items ii ON t.trackid = ii.trackid
    GROUP BY ar.artistid, ar.name
),
sales_stats AS (
    SELECT AVG(total_sold) as avg_sales
    FROM artist_sales
)
SELECT
    a.artist_name,
    a.track_count,
    a.total_sold,
    ROUND(a.total_sold - s.avg_sales, 2) as diff_from_avg
FROM artist_sales a
CROSS JOIN sales_stats s
WHERE a.total_sold > s.avg_sales
ORDER BY a.total_sold DESC;
```

---

## 다중 CTE

하나의 쿼리에서 여러 CTE를 정의할 수 있습니다:

```sql
WITH
-- 첫 번째 CTE
monthly_sales AS (
    SELECT
        strftime('%Y-%m', invoicedate) as month,
        SUM(total) as monthly_total
    FROM invoices
    GROUP BY strftime('%Y-%m', invoicedate)
),
-- 두 번째 CTE (첫 번째 CTE 참조 가능)
sales_growth AS (
    SELECT
        month,
        monthly_total,
        LAG(monthly_total) OVER (ORDER BY month) as prev_month_total
    FROM monthly_sales
),
-- 세 번째 CTE
growth_rate AS (
    SELECT
        month,
        monthly_total,
        prev_month_total,
        CASE
            WHEN prev_month_total > 0
            THEN ROUND((monthly_total - prev_month_total) * 100.0 / prev_month_total, 2)
            ELSE NULL
        END as growth_percentage
    FROM sales_growth
)
-- 메인 쿼리
SELECT *
FROM growth_rate
WHERE growth_percentage IS NOT NULL
ORDER BY month;
```

### 다중 CTE 작성 규칙
- 쉼표(,)로 구분
- 이전 CTE를 다음 CTE에서 참조 가능
- 순서가 중요 (앞에 정의된 CTE만 참조 가능)

---

## 재귀 CTE

재귀 CTE는 자기 자신을 참조하여 계층적 데이터나 반복 작업을 처리합니다.

### 기본 문법

```sql
WITH RECURSIVE cte_name AS (
    -- 기본 케이스 (앵커 멤버)
    SELECT ...

    UNION ALL

    -- 재귀 케이스 (재귀 멤버)
    SELECT ...
    FROM cte_name
    WHERE termination_condition
)
SELECT * FROM cte_name;
```

### 예제 1: 숫자 시퀀스 생성

```sql
-- 1부터 10까지 숫자 생성
WITH RECURSIVE numbers AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1
    FROM numbers
    WHERE n < 10
)
SELECT * FROM numbers;
```

### 예제 2: 조직도 (계층 구조)

```sql
-- 직원 테이블 가정
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name TEXT,
    manager_id INTEGER
);

-- 특정 매니저 아래의 모든 직원 찾기
WITH RECURSIVE employee_hierarchy AS (
    -- 앵커: 최상위 매니저
    SELECT id, name, manager_id, 0 as level, name as path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- 재귀: 하위 직원들
    SELECT
        e.id,
        e.name,
        e.manager_id,
        eh.level + 1,
        eh.path || ' > ' || e.name
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT
    SUBSTR('          ', 1, level * 2) || name as hierarchy,
    level,
    path
FROM employee_hierarchy
ORDER BY path;
```

### 예제 3: 날짜 범위 생성

```sql
-- 지난 30일 날짜 생성
WITH RECURSIVE date_series AS (
    SELECT DATE('now', '-30 days') as date
    UNION ALL
    SELECT DATE(date, '+1 day')
    FROM date_series
    WHERE date < DATE('now')
)
SELECT
    date,
    strftime('%w', date) as day_of_week,
    CASE strftime('%w', date)
        WHEN '0' THEN '일요일'
        WHEN '6' THEN '토요일'
        ELSE '평일'
    END as day_type
FROM date_series;
```

### 재귀 CTE 주의사항
- **무한 루프 방지**: 반드시 종료 조건 필요
- **최대 재귀 깊이**: SQLite는 기본적으로 1000회 제한
- **성능**: 큰 계층 구조에서는 성능 저하 가능

---

## CTE vs 서브쿼리

### 서브쿼리 방식

```sql
SELECT
    c.firstname || ' ' || c.lastname as customer,
    sub.total_purchases
FROM customers c
JOIN (
    SELECT
        customerid,
        COUNT(*) as total_purchases
    FROM invoices
    GROUP BY customerid
    HAVING COUNT(*) > 5
) sub ON c.customerid = sub.customerid
ORDER BY sub.total_purchases DESC;
```

### CTE 방식

```sql
WITH customer_purchases AS (
    SELECT
        customerid,
        COUNT(*) as total_purchases
    FROM invoices
    GROUP BY customerid
    HAVING COUNT(*) > 5
)
SELECT
    c.firstname || ' ' || c.lastname as customer,
    cp.total_purchases
FROM customers c
JOIN customer_purchases cp ON c.customerid = cp.customerid
ORDER BY cp.total_purchases DESC;
```

### 비교

| 특징 | CTE | 서브쿼리 |
|------|-----|---------|
| 가독성 | ⭐⭐⭐⭐⭐ 매우 좋음 | ⭐⭐ 복잡하면 어려움 |
| 재사용 | ✅ 여러 번 참조 가능 | ❌ 반복 작성 필요 |
| 유지보수 | ✅ 쉬움 | ❌ 어려움 |
| 디버깅 | ✅ 단계별 실행 가능 | ❌ 전체 쿼리 실행 필요 |
| 재귀 | ✅ 가능 | ❌ 불가능 |
| 성능 | 비슷함 | 비슷함 |

---

## 실무 활용 패턴

### 패턴 1: 데이터 정제 파이프라인

```sql
WITH
-- 1단계: 원본 데이터 필터링
raw_data AS (
    SELECT *
    FROM sales
    WHERE sale_date >= DATE('now', '-1 year')
),
-- 2단계: 데이터 정규화
normalized_data AS (
    SELECT
        customerid,
        UPPER(TRIM(product_name)) as product_name,
        quantity,
        price,
        quantity * price as revenue
    FROM raw_data
    WHERE quantity > 0 AND price > 0
),
-- 3단계: 집계
aggregated_data AS (
    SELECT
        customerid,
        product_name,
        SUM(quantity) as total_quantity,
        SUM(revenue) as total_revenue,
        COUNT(*) as purchase_count
    FROM normalized_data
    GROUP BY customerid, product_name
)
-- 최종 결과
SELECT
    customerid,
    product_name,
    total_quantity,
    total_revenue,
    ROUND(total_revenue / total_quantity, 2) as avg_price,
    purchase_count
FROM aggregated_data
WHERE total_revenue > 100
ORDER BY total_revenue DESC;
```

### 패턴 2: 랭킹과 분석

```sql
WITH
product_sales AS (
    SELECT
        p.productid,
        p.name,
        p.category,
        SUM(s.quantity) as total_sold,
        SUM(s.quantity * s.price) as total_revenue
    FROM products p
    JOIN sales s ON p.productid = s.productid
    GROUP BY p.productid, p.name, p.category
),
ranked_products AS (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY total_revenue DESC) as rank_in_category,
        ROUND(100.0 * total_revenue / SUM(total_revenue) OVER (PARTITION BY category), 2) as pct_of_category
    FROM product_sales
)
SELECT
    category,
    name,
    total_sold,
    total_revenue,
    rank_in_category,
    pct_of_category
FROM ranked_products
WHERE rank_in_category <= 3
ORDER BY category, rank_in_category;
```

### 패턴 3: 시계열 분석

```sql
WITH
daily_metrics AS (
    SELECT
        DATE(order_date) as date,
        COUNT(DISTINCT orderid) as orders,
        COUNT(DISTINCT customerid) as customers,
        SUM(amount) as revenue
    FROM orders
    GROUP BY DATE(order_date)
),
moving_averages AS (
    SELECT
        date,
        orders,
        customers,
        revenue,
        AVG(revenue) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) as ma_7day,
        AVG(revenue) OVER (
            ORDER BY date
            ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
        ) as ma_30day
    FROM daily_metrics
)
SELECT
    date,
    orders,
    customers,
    ROUND(revenue, 2) as revenue,
    ROUND(ma_7day, 2) as moving_avg_7day,
    ROUND(ma_30day, 2) as moving_avg_30day,
    ROUND((revenue - ma_7day) / ma_7day * 100, 2) as pct_diff_from_7day_avg
FROM moving_averages
WHERE date >= DATE('now', '-90 days')
ORDER BY date DESC;
```

### 패턴 4: 코호트 분석

```sql
WITH
first_purchase AS (
    SELECT
        customerid,
        MIN(DATE(purchase_date)) as cohort_date
    FROM purchases
    GROUP BY customerid
),
customer_cohorts AS (
    SELECT
        p.customerid,
        fp.cohort_date,
        DATE(p.purchase_date) as purchase_date,
        (JULIANDAY(p.purchase_date) - JULIANDAY(fp.cohort_date)) / 30 as months_since_first
    FROM purchases p
    JOIN first_purchase fp ON p.customerid = fp.customerid
),
cohort_analysis AS (
    SELECT
        cohort_date,
        CAST(months_since_first AS INTEGER) as month_number,
        COUNT(DISTINCT customerid) as customers
    FROM customer_cohorts
    GROUP BY cohort_date, CAST(months_since_first AS INTEGER)
),
cohort_sizes AS (
    SELECT
        cohort_date,
        customers as cohort_size
    FROM cohort_analysis
    WHERE month_number = 0
)
SELECT
    ca.cohort_date,
    ca.month_number,
    ca.customers,
    cs.cohort_size,
    ROUND(100.0 * ca.customers / cs.cohort_size, 2) as retention_rate
FROM cohort_analysis ca
JOIN cohort_sizes cs ON ca.cohort_date = cs.cohort_date
WHERE ca.month_number <= 12
ORDER BY ca.cohort_date, ca.month_number;
```

---

## 성능 최적화 팁

### 1. 인덱스 활용
CTE 내부 쿼리도 인덱스를 활용합니다:

```sql
-- customers 테이블의 customerid에 인덱스가 있다고 가정
WITH active_customers AS (
    SELECT customerid, firstname, lastname
    FROM customers
    WHERE status = 'active'  -- status에도 인덱스가 있으면 좋음
)
SELECT * FROM active_customers;
```

### 2. 불필요한 컬럼 제외
CTE에서 필요한 컬럼만 선택:

```sql
-- 나쁜 예
WITH customer_data AS (
    SELECT *  -- 모든 컬럼 선택
    FROM customers
)
SELECT firstname, lastname FROM customer_data;

-- 좋은 예
WITH customer_data AS (
    SELECT firstname, lastname  -- 필요한 컬럼만
    FROM customers
)
SELECT * FROM customer_data;
```

### 3. 조기 필터링
가능한 한 빨리 데이터를 필터링:

```sql
WITH filtered_orders AS (
    SELECT orderid, customerid, amount
    FROM orders
    WHERE order_date >= DATE('now', '-1 year')  -- 먼저 필터링
    AND amount > 100
)
SELECT
    customerid,
    COUNT(*) as order_count,
    SUM(amount) as total_amount
FROM filtered_orders
GROUP BY customerid;
```

---

## 일반적인 실수와 해결책

### 실수 1: CTE를 뷰처럼 사용
```sql
-- 잘못된 사용: CTE는 쿼리 범위 내에서만 존재
WITH temp_data AS (
    SELECT * FROM table1
);  -- 여기서 끝

SELECT * FROM temp_data;  -- ❌ 오류: temp_data를 찾을 수 없음
```

**해결책**: WITH와 SELECT를 하나의 문장으로:
```sql
WITH temp_data AS (
    SELECT * FROM table1
)
SELECT * FROM temp_data;  -- ✅ 올바름
```

### 실수 2: 재귀 CTE에서 종료 조건 누락
```sql
-- 위험: 무한 루프
WITH RECURSIVE infinite AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM infinite  -- ❌ 종료 조건 없음
)
SELECT * FROM infinite;
```

**해결책**: 항상 WHERE 절로 종료:
```sql
WITH RECURSIVE safe AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM safe
    WHERE n < 100  -- ✅ 종료 조건
)
SELECT * FROM safe;
```

### 실수 3: CTE 이름 중복
```sql
-- 오류: 같은 이름 재사용
WITH data AS (SELECT * FROM table1),
     data AS (SELECT * FROM table2)  -- ❌ 중복
SELECT * FROM data;
```

**해결책**: 고유한 이름 사용:
```sql
WITH data1 AS (SELECT * FROM table1),
     data2 AS (SELECT * FROM table2)  -- ✅ 고유한 이름
SELECT * FROM data1 UNION SELECT * FROM data2;
```

---

## 요약

### CTE를 사용해야 하는 경우
✅ 복잡한 쿼리를 논리적 단계로 분리할 때
✅ 동일한 서브쿼리를 여러 번 사용할 때
✅ 재귀적 쿼리가 필요할 때
✅ 코드 가독성과 유지보수성이 중요할 때

### CTE의 핵심 특징
- 쿼리 범위 내에서만 존재하는 임시 결과 집합
- 여러 CTE를 연결하여 데이터 처리 파이프라인 구성 가능
- 재귀 CTE로 계층적 데이터 처리
- 성능은 일반 서브쿼리와 유사

### 베스트 프랙티스
1. **명확한 이름 사용**: CTE 이름은 그 목적을 분명히 표현
2. **논리적 순서**: 데이터 흐름을 따라 CTE 배치
3. **적절한 주석**: 복잡한 로직은 주석으로 설명
4. **단계별 테스트**: 각 CTE를 독립적으로 실행하여 검증
5. **성능 고려**: 필요한 컬럼만 선택하고 조기에 필터링

---

## 추가 학습 자료

- [SQLite 공식 문서 - SELECT](https://www.sqlite.org/lang_select.html)
- [SQLite 윈도우 함수](https://www.sqlite.org/windowfunctions.html)
- [SQLite 재귀 쿼리 제한](https://www.sqlite.org/limits.html)

이 가이드를 통해 SQLite CTE를 효과적으로 활용하여 더 읽기 쉽고 유지보수하기 쉬운 SQL 쿼리를 작성할 수 있습니다.
