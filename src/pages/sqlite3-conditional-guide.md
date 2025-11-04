---
title: SQLite3 조건 분기 완전 가이드
layout: '../layout/Base.astro'
---
# SQLite3 조건 분기 완전 가이드

## 1. CASE 표현식

### 1-1. 단순 CASE 문
```sql
CASE expression
    WHEN value1 THEN result1
    WHEN value2 THEN result2
    ELSE default_result
END
```

**예시:**
```sql
SELECT 
    name,
    CASE status
        WHEN 1 THEN '활성'
        WHEN 0 THEN '비활성'
        ELSE '알 수 없음'
    END AS status_text
FROM users;
```

### 1-2. 검색 CASE 문 (더 유연함)
```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE default_result
END
```

**예시:**
```sql
SELECT 
    name,
    age,
    CASE
        WHEN age < 20 THEN '청소년'
        WHEN age < 40 THEN '청년'
        WHEN age < 60 THEN '중년'
        ELSE '노년'
    END AS age_group
FROM users;
```

## 2. IIF 함수 (SQLite 3.32.0+)

간단한 조건 분기용 함수입니다.

```sql
IIF(condition, true_result, false_result)
```

**예시:**
```sql
SELECT 
    name,
    IIF(age >= 18, '성인', '미성년자') AS age_status
FROM users;
```

## 3. COALESCE와 IFNULL

### 3-1. COALESCE
첫 번째 NULL이 아닌 값을 반환합니다.

```sql
COALESCE(value1, value2, value3, ..., default_value)
```

**예시:**
```sql
SELECT 
    name,
    COALESCE(phone, mobile, email, '연락처 없음') AS contact
FROM users;
```

### 3-2. IFNULL
두 개의 인자만 받는 간단한 버전입니다.

```sql
IFNULL(value, default_value)
```

**예시:**
```sql
SELECT 
    name,
    IFNULL(nickname, name) AS display_name
FROM users;
```

## 4. NULLIF

두 값이 같으면 NULL을 반환하고, 다르면 첫 번째 값을 반환합니다.

```sql
NULLIF(value1, value2)
```

**예시:**
```sql
SELECT 
    name,
    NULLIF(score, 0) AS valid_score  -- 0을 NULL로 변환
FROM test_results;
```

## 5. WHERE 절에서의 조건 분기

```sql
-- AND, OR, NOT 사용
SELECT * FROM users 
WHERE (age >= 18 AND status = 'active') 
   OR (vip = 1);

-- IN 연산자
SELECT * FROM users 
WHERE country IN ('KR', 'JP', 'CN');

-- BETWEEN
SELECT * FROM products 
WHERE price BETWEEN 1000 AND 5000;

-- LIKE 패턴 매칭
SELECT * FROM users 
WHERE email LIKE '%@gmail.com';

-- EXISTS
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM order_items oi 
    WHERE oi.order_id = o.id
);
```

## 6. 복잡한 조건 분기 예시

### 6-1. 가격 등급 계산
```sql
SELECT 
    product_name,
    price,
    CASE
        WHEN price < 10000 THEN '저가'
        WHEN price < 50000 THEN '중가'
        WHEN price < 100000 THEN '고가'
        ELSE '프리미엄'
    END AS price_tier,
    CASE
        WHEN stock > 100 THEN '충분'
        WHEN stock > 10 THEN '보통'
        WHEN stock > 0 THEN '부족'
        ELSE '품절'
    END AS stock_status
FROM products;
```

### 6-2. 집계 함수와 조건 분기
```sql
SELECT 
    category,
    COUNT(*) AS total,
    SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) AS active_count,
    SUM(CASE WHEN status = 'inactive' THEN 1 ELSE 0 END) AS inactive_count,
    AVG(CASE WHEN price > 0 THEN price ELSE NULL END) AS avg_valid_price
FROM products
GROUP BY category;
```

### 6-3. 중첩 CASE
```sql
SELECT 
    name,
    CASE
        WHEN age < 18 THEN '미성년자'
        ELSE 
            CASE
                WHEN vip = 1 THEN 'VIP 회원'
                WHEN membership_years > 5 THEN '우수 회원'
                ELSE '일반 회원'
            END
    END AS member_grade
FROM users;
```

## 7. UPDATE/DELETE에서의 조건 분기

### 7-1. UPDATE with CASE
```sql
UPDATE products
SET discount = CASE
    WHEN category = 'electronics' THEN 0.1
    WHEN category = 'clothing' THEN 0.2
    WHEN stock > 100 THEN 0.15
    ELSE 0.05
END
WHERE active = 1;
```

### 7-2. DELETE with conditions
```sql
DELETE FROM logs
WHERE created_at < date('now', '-30 days')
   OR (level = 'debug' AND created_at < date('now', '-7 days'));
```

## 8. 고급 활용 예시

### 8-1. 동적 정렬
```sql
SELECT *
FROM products
ORDER BY 
    CASE 
        WHEN @sort_order = 'price_asc' THEN price 
    END ASC,
    CASE 
        WHEN @sort_order = 'price_desc' THEN price 
    END DESC,
    CASE 
        WHEN @sort_order = 'name' THEN product_name 
    END ASC;
```

### 8-2. 조건부 JOIN
```sql
SELECT 
    o.*,
    CASE
        WHEN o.status = 'completed' THEN od.delivery_date
        ELSE NULL
    END AS delivery_date
FROM orders o
LEFT JOIN order_delivery od ON o.id = od.order_id 
    AND o.status = 'completed';
```

### 8-3. 피벗 테이블 생성
```sql
SELECT 
    date,
    SUM(CASE WHEN category = 'A' THEN amount ELSE 0 END) AS category_a,
    SUM(CASE WHEN category = 'B' THEN amount ELSE 0 END) AS category_b,
    SUM(CASE WHEN category = 'C' THEN amount ELSE 0 END) AS category_c
FROM sales
GROUP BY date;
```

### 8-4. 복잡한 비즈니스 로직
```sql
SELECT 
    customer_id,
    order_count,
    total_amount,
    CASE
        WHEN order_count >= 10 AND total_amount >= 1000000 THEN 'VIP'
        WHEN order_count >= 5 OR total_amount >= 500000 THEN 'Gold'
        WHEN order_count >= 2 AND total_amount >= 100000 THEN 'Silver'
        WHEN order_count >= 1 THEN 'Bronze'
        ELSE 'New'
    END AS customer_tier,
    CASE
        WHEN order_count >= 10 AND total_amount >= 1000000 THEN 0.15
        WHEN order_count >= 5 OR total_amount >= 500000 THEN 0.10
        WHEN order_count >= 2 AND total_amount >= 100000 THEN 0.05
        ELSE 0.00
    END AS discount_rate
FROM (
    SELECT 
        customer_id,
        COUNT(*) AS order_count,
        SUM(amount) AS total_amount
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
) customer_stats;
```

## 9. 성능 최적화 팁

### 9-1. 인덱스 활용
```sql
-- CASE 문에서 자주 사용되는 컬럼에 인덱스 생성
CREATE INDEX idx_users_age ON users(age);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_orders_status ON orders(status);
```

### 9-2. CASE 대신 조인 사용 고려
```sql
-- 비효율적
SELECT 
    p.*,
    CASE p.category_id
        WHEN 1 THEN '전자제품'
        WHEN 2 THEN '의류'
        WHEN 3 THEN '식품'
        -- ... 수십 개의 WHEN
    END AS category_name
FROM products p;

-- 효율적
SELECT 
    p.*,
    c.name AS category_name
FROM products p
JOIN categories c ON p.category_id = c.id;
```

### 9-3. 조건 순서 최적화
```sql
-- 가장 빈번한 조건을 먼저 배치
SELECT 
    CASE
        WHEN status = 'active' THEN 1  -- 90%의 데이터
        WHEN status = 'pending' THEN 2  -- 8%의 데이터
        WHEN status = 'inactive' THEN 3  -- 2%의 데이터
        ELSE 0
    END AS status_order
FROM users;
```

## 10. 주의사항

### 10-1. ELSE 생략 시 NULL 반환
```sql
SELECT 
    CASE status
        WHEN 1 THEN '활성'
        -- ELSE가 없으면 status가 1이 아닐 때 NULL 반환
    END AS status_text
FROM users;
```

### 10-2. 데이터 타입 일관성
```sql
-- 잘못된 예: 타입 불일치
SELECT 
    CASE
        WHEN age < 18 THEN '미성년자'  -- TEXT
        ELSE age  -- INTEGER
    END
FROM users;

-- 올바른 예: 타입 통일
SELECT 
    CASE
        WHEN age < 18 THEN '미성년자'
        ELSE CAST(age AS TEXT)
    END
FROM users;
```

### 10-3. NULL 비교
```sql
-- 잘못된 예
SELECT * FROM users WHERE phone = NULL;  -- 항상 false

-- 올바른 예
SELECT * FROM users WHERE phone IS NULL;

-- CASE 문에서 NULL 처리
SELECT 
    CASE
        WHEN phone IS NOT NULL THEN phone
        WHEN mobile IS NOT NULL THEN mobile
        ELSE '연락처 없음'
    END AS contact
FROM users;
```

### 10-4. 복잡한 CASE 문의 가독성
```sql
-- 복잡한 경우 CTE(Common Table Expression) 사용 권장
WITH user_metrics AS (
    SELECT 
        user_id,
        COUNT(*) AS order_count,
        SUM(amount) AS total_amount
    FROM orders
    GROUP BY user_id
)
SELECT 
    u.name,
    CASE
        WHEN m.order_count >= 10 THEN 'VIP'
        WHEN m.order_count >= 5 THEN 'Gold'
        ELSE 'Regular'
    END AS tier
FROM users u
LEFT JOIN user_metrics m ON u.id = m.user_id;
```

## 11. 실전 활용 시나리오

### 11-1. 동적 필터링
```sql
-- 파라미터에 따른 동적 필터링
SELECT *
FROM products
WHERE 1=1
    AND (@category IS NULL OR category = @category)
    AND (@min_price IS NULL OR price >= @min_price)
    AND (@max_price IS NULL OR price <= @max_price)
    AND (
        @search_term IS NULL 
        OR name LIKE '%' || @search_term || '%'
        OR description LIKE '%' || @search_term || '%'
    );
```

### 11-2. 재고 경고 시스템
```sql
SELECT 
    product_id,
    product_name,
    current_stock,
    reorder_level,
    CASE
        WHEN current_stock = 0 THEN '긴급'
        WHEN current_stock < reorder_level * 0.5 THEN '높음'
        WHEN current_stock < reorder_level THEN '중간'
        ELSE '정상'
    END AS alert_level,
    CASE
        WHEN current_stock = 0 THEN '즉시 발주 필요'
        WHEN current_stock < reorder_level * 0.5 THEN '조속한 발주 권장'
        WHEN current_stock < reorder_level THEN '발주 검토 필요'
        ELSE '재고 충분'
    END AS action_required
FROM inventory;
```

### 11-3. 시간대별 요금 계산
```sql
SELECT 
    usage_id,
    usage_time,
    kwh,
    CASE
        WHEN CAST(strftime('%H', usage_time) AS INTEGER) BETWEEN 23 AND 7 
            THEN kwh * 50  -- 심야 요금
        WHEN CAST(strftime('%H', usage_time) AS INTEGER) BETWEEN 10 AND 12 
            OR CAST(strftime('%H', usage_time) AS INTEGER) BETWEEN 13 AND 17
            THEN kwh * 150  -- 피크 요금
        ELSE kwh * 100  -- 일반 요금
    END AS charge
FROM electricity_usage;
```

## 12. 요약

SQLite3의 조건 분기는 다음과 같이 요약할 수 있습니다:

| 기능 | 용도 | 복잡도 |
|------|------|--------|
| CASE | 다중 조건 분기 | 높음 |
| IIF | 단순 이진 분기 | 낮음 |
| COALESCE | NULL 처리 (다중) | 중간 |
| IFNULL | NULL 처리 (단순) | 낮음 |
| NULLIF | 특정 값을 NULL로 변환 | 낮음 |
| WHERE | 행 필터링 | 중간 |

이러한 조건 분기 문법들을 적절히 조합하면 복잡한 비즈니스 로직을 SQL로 구현할 수 있습니다.
