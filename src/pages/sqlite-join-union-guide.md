---
title: SQLite JOIN & UNION 핵심 가이드
layout: '../layout/Base.astro'
---
# SQLite JOIN & UNION 핵심 가이드

## 목차
1. [JOIN 기본 개념](#join-기본-개념)
2. [INNER JOIN vs LEFT JOIN](#inner-join-vs-left-join)
3. [JOIN의 동작 원리](#join의-동작-원리)
4. [UNION 완전 가이드](#union-완전-가이드)
5. [FULL OUTER JOIN 구현](#full-outer-join-구현)
6. [실전 활용 패턴](#실전-활용-패턴)

---

## JOIN 기본 개념

JOIN은 두 개 이상의 테이블에서 관련된 데이터를 **가로로 결합**하는 연산입니다.

### 기본 테이블 예제

**customers 테이블:**
```
customer_id | name
------------|--------
1           | Alice
2           | Bob
3           | Charlie
```

**orders 테이블:**
```
order_id | customer_id | amount
---------|-------------|-------
101      | 1           | 500
102      | 1           | 300
103      | 2           | 700
```

---

## INNER JOIN vs LEFT JOIN

### INNER JOIN

**정의:** 양쪽 테이블에 모두 존재하는 행만 반환

**구문:**
```sql
SELECT columns
FROM table1
INNER JOIN table2 ON table1.key = table2.key;
```

**예제:**
```sql
SELECT customers.name, orders.amount
FROM customers
INNER JOIN orders ON customers.customer_id = orders.customer_id;
```

**결과:**
```
name  | amount
------|-------
Alice | 500
Alice | 300
Bob   | 700
```
- Charlie는 주문이 없으므로 결과에서 제외됨

### LEFT JOIN

**정의:** 왼쪽 테이블의 모든 행을 반환하고, 오른쪽 테이블에서 일치하는 행이 있으면 결합. 없으면 NULL로 채움

**구문:**
```sql
SELECT columns
FROM table1
LEFT JOIN table2 ON table1.key = table2.key;
```

**예제:**
```sql
SELECT customers.name, orders.amount
FROM customers
LEFT JOIN orders ON customers.customer_id = orders.customer_id;
```

**결과:**
```
name    | amount
--------|-------
Alice   | 500
Alice   | 300
Bob     | 700
Charlie | NULL
```
- Charlie는 주문이 없지만 LEFT JOIN이므로 결과에 포함됨

### 핵심 차이

| 구분 | INNER JOIN | LEFT JOIN |
|------|-----------|-----------|
| **기준** | 양쪽 모두 | 왼쪽 테이블 |
| **매칭 없는 경우** | 행 제외 | NULL로 채워서 포함 |
| **결과 행 수** | 적음 | 같거나 많음 |
| **사용 시기** | 교집합만 필요 | 전체 목록 필요 |

### 선택 기준

- **INNER JOIN**: "매칭되는 것만 보여줘"
  - 실제 주문한 고객만
  - 배정된 프로젝트가 있는 직원만
  
- **LEFT JOIN**: "왼쪽 전부 보여주되, 매칭되는 정보도 같이"
  - 모든 고객 목록 (주문 여부 무관)
  - 전체 직원 목록 (프로젝트 배정 여부 무관)

---

## JOIN의 동작 원리

### 개념적 실행 순서

#### INNER JOIN
```
1. CROSS JOIN 생성 (A × B - 모든 조합)
2. ON 조건을 만족하는 행만 필터링
3. 결과 반환
```

#### LEFT JOIN
```
1. 왼쪽 테이블의 각 행을 기준으로 시작
2. 오른쪽 테이블에서 조건을 만족하는 행 찾기
3. 찾으면 결합, 못 찾으면 오른쪽을 NULL로 채움
4. 조건 불만족으로 생긴 NULL 행들은 왼쪽 행당 1개만 유지
```

### CROSS JOIN 관점에서 이해

**INNER JOIN:**
```
CROSS JOIN → 조건 만족하는 것만 선택
```

**LEFT JOIN:**
```
CROSS JOIN → 조건 만족하는 것 선택 
           → 왼쪽 행에 매칭이 없으면 NULL 행 1개 추가
```

### 중요한 차이점

LEFT JOIN에서 1:N 관계가 있을 때:
- Alice가 주문 2개 → 2개 행으로 나타남
- Charlie가 주문 0개 → 1개 NULL 행으로 나타남
- **중복 제거 없음**: 매칭된 행은 모두 표시

---

## UNION 완전 가이드

### UNION이란?

두 개 이상의 SELECT 결과를 **세로로 합치는** 연산입니다.

```
JOIN:  테이블을 가로로 결합 (A + B = A|B)
UNION: 결과를 세로로 합침 (A ∪ B = A 위에 B)
```

### UNION vs UNION ALL

#### UNION (중복 제거)
```sql
SELECT column FROM table1
UNION
SELECT column FROM table2;
```
- 중복된 행을 자동으로 제거
- 정렬과 비교 과정이 필요하여 느림

#### UNION ALL (중복 포함)
```sql
SELECT column FROM table1
UNION ALL
SELECT column FROM table2;
```
- 모든 행을 포함
- 중복 검사 없이 빠름

### 예제

**테이블:**
```
employees_seoul: Alice, Bob, Charlie
employees_busan: Charlie, David, Eve
```

**UNION 결과 (5개):**
```
Alice, Bob, Charlie, David, Eve
```

**UNION ALL 결과 (6개):**
```
Alice, Bob, Charlie, Charlie, David, Eve
```

### UNION 규칙

| 항목 | 규칙 |
|------|------|
| **칼럼 개수** | ✅ 반드시 같아야 함 |
| **칼럼 이름** | ❌ 달라도 됨 (첫 번째 SELECT 따름) |
| **칼럼 타입** | ✅ 호환 가능해야 함 |
| **매칭 기준** | 위치(position) |
| **ORDER BY** | 마지막에 한 번만 |

### 올바른 예제

```sql
-- ✅ 칼럼 개수 동일, 이름 다름 (OK)
SELECT user_id, username FROM users
UNION
SELECT admin_id, admin_name FROM admins;

-- ✅ 별칭으로 의미 통일 (권장)
SELECT user_id as id, username as name FROM users
UNION
SELECT admin_id as id, admin_name as name FROM admins;
```

### 잘못된 예제

```sql
-- ❌ 칼럼 개수 불일치
SELECT id, name FROM table1
UNION
SELECT id FROM table2;  -- 에러!
```

---

## FULL OUTER JOIN 구현

SQLite는 FULL OUTER JOIN을 직접 지원하지 않습니다. UNION으로 구현해야 합니다.

### FULL OUTER JOIN이란?

양쪽 테이블의 **모든 행**을 포함하며, 매칭 없으면 NULL로 채웁니다.

### 구현 방법

#### 방법 1: UNION (자동 중복 제거)

```sql
SELECT c.id, c.name, o.id, o.amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id

UNION

SELECT c.id, c.name, o.id, o.amount
FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id;
```

#### 방법 2: UNION ALL + WHERE (더 빠름)

```sql
SELECT c.id, c.name, o.id, o.amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id

UNION ALL

SELECT c.id, c.name, o.id, o.amount
FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id
WHERE c.id IS NULL;  -- 중복 방지
```

### 예제 결과

**데이터:**
```
customers: 1-Alice, 2-Bob, 3-Charlie
orders: 101(customer_id=1), 102(customer_id=2), 103(customer_id=99)
```

**LEFT JOIN만:**
```
1 | Alice   | 101 | 500
2 | Bob     | 102 | 300
3 | Charlie | NULL| NULL
```
→ Charlie 포함, customer_id=99 주문 제외

**RIGHT JOIN 효과:**
```
1    | Alice | 101 | 500
2    | Bob   | 102 | 300
NULL | NULL  | 103 | 700
```
→ customer_id=99 주문 포함, Charlie 제외

**FULL OUTER JOIN:**
```
1    | Alice   | 101 | 500
2    | Bob     | 102 | 300
3    | Charlie | NULL| NULL
NULL | NULL    | 103 | 700
```
→ 양쪽 모두 포함

---

## 실전 활용 패턴

### 1. 여러 소스 통합

```sql
-- 여러 지역 매출 통합
SELECT '서울' as region, product, sales FROM seoul_sales
UNION ALL
SELECT '부산' as region, product, sales FROM busan_sales
UNION ALL
SELECT '대구' as region, product, sales FROM daegu_sales;
```

### 2. 조건별 분류

```sql
-- VIP와 일반 고객 구분
SELECT id, name, 'VIP' as grade
FROM customers
WHERE total_purchase > 1000000

UNION ALL

SELECT id, name, 'REGULAR' as grade
FROM customers
WHERE total_purchase <= 1000000;
```

### 3. 누락 데이터 찾기

```sql
-- 주문은 있는데 고객 정보가 없는 경우
SELECT o.customer_id, 'MISSING CUSTOMER' as status
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL;
```

### 4. 다른 구조 테이블 통합

```sql
-- 여러 사용자 타입 통합
SELECT user_id as id, username as name, 'USER' as type
FROM users

UNION ALL

SELECT admin_no as id, admin_name as name, 'ADMIN' as type
FROM admins

UNION ALL

SELECT guest_code as id, guest_display as name, 'GUEST' as type
FROM guests;
```

### 5. NULL 처리와 집계

```sql
-- 주문 여부 포함한 고객 통계
SELECT 
    c.name,
    COUNT(o.order_id) as order_count,
    COALESCE(SUM(o.amount), 0) as total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id;
```

---

## 성능 최적화 팁

### JOIN 최적화

1. **인덱스 생성**
```sql
CREATE INDEX idx_customer_id ON orders(customer_id);
```

2. **필요한 칼럼만 선택**
```sql
-- ❌ 비효율적
SELECT * FROM customers JOIN orders ...

-- ✅ 효율적
SELECT c.name, o.amount FROM customers c JOIN orders o ...
```

3. **WHERE 조건 주의**
```sql
-- ❌ LEFT JOIN 효과 상실
SELECT * FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.amount > 0;  -- NULL 행 제외됨

-- ✅ LEFT JOIN 유지
SELECT * FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id 
    AND o.amount > 0;
```

### UNION 최적화

| 상황 | 선택 |
|------|------|
| 중복 없음 확실 | UNION ALL |
| 중복 제거 필요 | UNION |
| 대용량 + 중복 방지 | UNION ALL + WHERE IS NULL |

---

## 빠른 참조표

### JOIN 타입 비교

| JOIN 타입 | 결과 | SQLite 지원 |
|-----------|------|-------------|
| INNER JOIN | 양쪽에 모두 있는 것만 | ✅ |
| LEFT JOIN | 왼쪽 전체 + 매칭되는 오른쪽 | ✅ |
| RIGHT JOIN | 오른쪽 전체 + 매칭되는 왼쪽 | ❌ (LEFT로 대체) |
| FULL OUTER JOIN | 양쪽 전체 | ❌ (UNION으로 구현) |
| CROSS JOIN | 모든 조합 (카테시안 곱) | ✅ |

### 언제 무엇을 사용할까?

```
교집합만 필요? → INNER JOIN
왼쪽 전체 필요? → LEFT JOIN
양쪽 전체 필요? → FULL OUTER JOIN (UNION으로 구현)
결과 합치기? → UNION / UNION ALL
```

---

## 일반적인 실수와 해결

### 실수 1: ON 조건 누락
```sql
-- ❌ CROSS JOIN 발생
SELECT * FROM customers, orders;

-- ✅ 명시적 JOIN
SELECT * FROM customers
INNER JOIN orders ON customers.id = orders.customer_id;
```

### 실수 2: LEFT JOIN 후 WHERE 절
```sql
-- ❌ INNER JOIN처럼 동작
SELECT * FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.amount > 0;

-- ✅ ON 절에 조건 추가
SELECT * FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id 
    AND o.amount > 0;
```

### 실수 3: UNION 칼럼 개수 불일치
```sql
-- ❌ 에러
SELECT id, name FROM table1
UNION
SELECT id FROM table2;

-- ✅ 칼럼 개수 맞추기
SELECT id, name FROM table1
UNION
SELECT id, NULL as name FROM table2;
```

---

## 요약

### JOIN 핵심
- **INNER JOIN**: 양쪽 모두 존재하는 행만
- **LEFT JOIN**: 왼쪽 기준, 매칭 없으면 NULL
- 개념적으로 CROSS JOIN + 필터링으로 이해 가능

### UNION 핵심
- 결과를 세로로 합침 (JOIN은 가로)
- 칼럼 개수만 같으면 됨 (이름은 무관)
- UNION: 중복 제거, UNION ALL: 모두 포함
- FULL OUTER JOIN은 LEFT + UNION + RIGHT로 구현

### 선택 가이드
```
"매칭되는 것만" → INNER JOIN
"왼쪽 전부" → LEFT JOIN
"양쪽 전부" → FULL OUTER JOIN (UNION 구현)
"결과 합치기" → UNION/UNION ALL
```
