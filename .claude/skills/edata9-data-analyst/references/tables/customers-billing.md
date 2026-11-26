# 고객 · 청구 표

가입한 고객과 월별 이용·청구를 다룬다.

---

## 빠른 참고

### 사업 맥락
통신사 가상 데이터다. `customers`가 고객 목록이고, `usage_monthly`가 고객별 월 이용량과 청구액이다. 데이터 범위는 2025년이다.

### 헷갈리는 말
- **«고객»**: `customers` 한 행. 가입 전 방문자와 다르다.
- **«고객 수»**: 가입자 전체, 월말 유지 고객, 청구가 있는 고객이 다르다. 정의서는 월말 유지 고객이다.

### 표준 조건
월 단위 지표는 월말 유지 고객으로 좁힌다.
```sql
WHERE signup_date <= DATE '2025-12-31'
  AND (churn_date IS NULL OR churn_date > DATE '2025-12-31')
```

---

## 표

### customers
**위치**: `data/customers.parquet` (50,000행)
**설명**: 고객 1명당 한 행. 가입 정보와 해지 상태.
**기본 키**: `customer_id`
**범위**: 가입일 2018-12-25 ~ 2025-12-31

| 열 | 형식 | 뜻 | 메모 |
|---|---|---|---|
| **customer_id** | VARCHAR | 고객 ID (`C000001`) | 중복 · 빈 값 없음 |
| signup_date | DATE | 가입일 | 말일 가입자가 12월에 104명 |
| plan_type | VARCHAR | 요금제 | LTE기본 · LTE무제한 · 5G스탠다드 · 알뜰 · 5G프리미엄 |
| monthly_fee | INTEGER | 월 요금(정가) | 청구액(`billing_amount`)과 같지 않다 |
| is_churned | BOOLEAN | 해지 여부 | **지금 시점의 상태.** 지난 달 계산에 쓰지 않는다 |
| churn_date | DATE | 해지일 | 해지하지 않았으면 NULL. 2025-01-01 ~ 2025-12-31 |
| visitor_id | VARCHAR | 가입 전 웹 방문자 ID | 6,485명만 있다 |
| customer_name | VARCHAR | 이름 | 고유하지 않다. 조인에 쓰지 않는다 |

해지한 고객은 7,256명이고, `is_churned`와 `churn_date`가 서로 어긋난 행은 없다.

**연결**
- `usage_monthly`, `support_tickets`, `csat_survey`와 `customer_id`로 잇는다.
- `sessions`와는 `visitor_id`로 잇는다.

### usage_monthly
**위치**: `data/usage_monthly.parquet` (523,825행)
**설명**: 고객 × 월 한 행. 이용량과 청구액.
**기본 키**: (`customer_id`, `year_month`), 중복 없음
**범위**: 2025-01 ~ 2025-12

| 열 | 형식 | 뜻 | 메모 |
|---|---|---|---|
| **customer_id** | VARCHAR | 고객 ID | 49,985명이 행을 갖는다 |
| **year_month** | VARCHAR | `'2025-12'` 형식 | 문자열 |
| billing_amount | INTEGER | 그 달 청구액(원) | 12월 최소 10,450 ~ 최대 98,880, 0원은 없다 |
| app_login_count | INTEGER | 앱 로그인 횟수 | 활성 고객 비율의 분자 기준 |
| data_gb · voice_min · sms_count | | 이용량 | |
| is_active | BOOLEAN | | **전부 TRUE.** 쓰지 않는다 |
| service_count | BIGINT | 가입 서비스 수 | |

해지한 달에도 행이 있다. 해지한 고객의 해지 이후 달에는 행이 없다.

---

## 조회 예

### 12월 월말 유지 고객의 청구 합계
```sql
SELECT COUNT(*) AS retained_customers, SUM(u.billing_amount) AS billing_total
FROM customers c
JOIN usage_monthly u ON u.customer_id = c.customer_id AND u.year_month = '2025-12'
WHERE c.signup_date <= DATE '2025-12-31'
  AND (c.churn_date IS NULL OR c.churn_date > DATE '2025-12-31')
-- 결과: 42,744명, 1,436,855,200원
```

### 요금제별 12월 월말 유지 고객 수
```sql
SELECT plan_type, COUNT(*) AS customers
FROM customers
WHERE signup_date <= DATE '2025-12-31'
  AND (churn_date IS NULL OR churn_date > DATE '2025-12-31')
GROUP BY 1 ORDER BY 2 DESC
```

### 문의를 조인할 때 행이 복제되지 않게 먼저 줄이기
```sql
WITH tk AS (
  SELECT customer_id, COUNT(*) AS n_tickets
  FROM support_tickets
  WHERE created_date BETWEEN DATE '2025-12-01' AND DATE '2025-12-31'
  GROUP BY 1)
SELECT u.customer_id, u.billing_amount, COALESCE(tk.n_tickets, 0) AS n_tickets
FROM usage_monthly u
LEFT JOIN tk ON tk.customer_id = u.customer_id
WHERE u.year_month = '2025-12'
```

---

## 자주 하는 실수

1. **지난 달 유지 고객을 `is_churned`로 센다.**
   - 틀림: `WHERE NOT is_churned AND signup_date <= DATE '2025-01-31'` → 37,093명
   - 맞음: `churn_date IS NULL OR churn_date > DATE '2025-01-31'` → 43,297명
2. **청구와 문의를 바로 조인한다.** 문의 건수만큼 청구 행이 복제된다. 위의 «먼저 줄이기» 패턴을 쓴다.
3. **`is_active`로 활성 고객을 센다.** 전부 TRUE다. `app_login_count >= 8`을 쓴다.
