---
name: edata9-data-analyst
description: "edata9 통신사 가상 데이터(data/ 의 parquet 12개, edata9-data 로 조회) 분석 스킬. 고객 · 청구 · 상담 · 만족도 질문에 답할 때 개체 정의, 지표 계산, 표를 조인하는 법, 자주 하는 실수를 알려 준다. 고객당 매출 · 월 청구 합계 · 월말 유지 고객 · 활성 고객 비율 · 상담 만족도(CSAT) · 해지 · 상담원 · 문의 같은 말이 나오거나, 이 데이터에 SQL을 돌릴 때는 이 스킬을 먼저 쓴다."
---

# edata9 데이터 분석

> 이 스킬은 `data-context-extractor`의 질문 다섯 개에 **예시 답**을 넣어 만든 것이다. 답은 이 폴더의 `metrics/` 정의서와 데이터를 조회해 확인한 사실로 채웠고, 회사 사람이 확인한 답은 아니다. 틀린 곳은 고쳐서 쓴다.

## SQL 방언: DuckDB (parquet)

- **조회 방법**: `edata9-data` MCP의 `query`는 읽기 전용 SQL 한 문장을 돌리고, 결과는 «전체 행 수 + 앞 20행»만 보여 준다. 긴 결과는 먼저 집계해서 줄인다. 표를 조회하기 전에 `list_tables`로 한 줄의 의미(grain)를 본다.
- **표 이름**: parquet 파일 이름이 표 이름이다(`customers`, `usage_monthly`…). 스키마 접두사는 없다.
- **날짜 리터럴**: `DATE '2025-12-31'`. 월 문자열은 `usage_monthly.year_month`처럼 `'2025-12'`(VARCHAR).
- **월 만들기**: `strftime(created_date, '%Y-%m')`, `date_trunc('month', d)`, `date_diff('day', a, b)`.
- **조건부 집계**: `COUNT(*) FILTER (WHERE s.csat >= 4)`.
- **나눗셈**: `/`는 정수끼리도 소수로 나온다. 0으로 나눌 수 있으면 `a / NULLIF(b, 0)`.
- **열 빼기**: `SELECT * EXCLUDE (comment)`.
- **불리언**: `is_churned`, `is_resigned`는 BOOLEAN이라 `WHERE NOT is_resigned`처럼 쓴다.

---

## 개체 구분: «고객»은 무엇인가

**«고객»이라고 하면** `customers` 한 행(`customer_id`)이다. 다른 말과 섞이기 쉬워서 구분한다.

| 말 | 뜻 | 표 : ID |
|---|---|---|
| 고객 | 가입한 사람 1명 | `customers` : `customer_id` (`C000001`) |
| 방문자 | 가입 전 웹 방문 단위. 고객 중 일부만 이어진다 | `sessions` : `visitor_id` (`customers.visitor_id`로 연결) |
| 문의 | 상담 1건 | `support_tickets` : `ticket_id` (`T000001`) |
| 설문 | 문의 1건에 대한 만족도 응답 | `csat_survey` : `survey_id` (`S000001`) |
| 상담원 | 문의를 맡은 직원 | `agents` : `agent_id` (`AG001`) |

**연결**
- 고객 → 문의: 1 : N (`support_tickets.customer_id`)
- 고객 → 월별 이용·청구: 1 : N (`usage_monthly.customer_id`, 한 줄 = 고객 × 월)
- 상담원 → 문의: 1 : N (`support_tickets.agent_id`)
- 문의 → 설문: 1 : 0~1 (`csat_survey.ticket_id`, 설문이 없는 문의가 많다)

자세한 내용은 `references/entities.md`.

---

## 용어

| 용어 | 뜻 | 메모 |
|---|---|---|
| 월말 유지 고객 | 그 달 마지막 날에 가입 상태인 고객 | 가입일 ≤ 말일 이고, 해지일이 없거나 말일보다 뒤 |
| 활성 고객 | 그 달 앱 로그인 8회 이상 | `usage_monthly.is_active`는 쓰지 않는다(아래 함정) |
| 해지 | `customers.churn_date`(2025년에만 있음) | `is_churned`는 지금 시점의 상태 |
| CSAT | 상담 만족도 점수 1~5 | 4점 이상을 «만족»으로 센다 |
| 응답 문의 | 설문이 달린 문의 | 응답률이 낮아(문의의 약 30%) 응답자 기준 지표다 |

---

## 항상 걸 조건

- **지표는 `metrics/` 정의서를 먼저 읽고 그대로 센다.** 답의 첫 줄에 따른 정의서의 id를 적는다. 정의서가 없는 지표는 세지 말고, 정해야 할 가정을 먼저 묻는다. (이 폴더의 `CLAUDE.md` 규칙)
- **월 청구 합계**: 월말 유지 고객의 청구만 더한다. 그 달 중에 해지한 고객의 청구는 뺀다.
- **상담 만족도**: 퇴사한 상담원(`agents.is_resigned`)이 응대한 문의는 뺀다.
- 이 5개 표에는 테스트 · 내부 직원을 가르는 열(`is_test` 등)이 없다. 따로 거를 것은 없다.

```sql
-- 월말 유지 고객 (말일 = :month_end)
SELECT customer_id FROM customers
WHERE signup_date <= DATE '2025-12-31'
  AND (churn_date IS NULL OR churn_date > DATE '2025-12-31')
```

---

## 지표

6개 지표의 정의서가 `metrics/`에 있다. 값과 SQL의 기준은 정의서이고, 이 요약이 다르면 정의서를 따른다. 자세한 내용은 `references/metrics.md`.

| 지표 (id) | 한 줄 | 2025-12 |
|---|---|---|
| 신규 가입 고객 수 (`new_customers`) | 가입일이 그 달 안인 고객 | 587명 |
| 활성 고객 비율 (`active_customer_rate`) | 월말 유지 고객 중 로그인 8회 이상 | 96.84% |
| 월 청구 합계 (`monthly_billing_total`) | 월말 유지 고객의 `billing_amount` 합 | 1,436,855,200원 |
| 월말 유지 고객 수 (`month_end_retained_customers`) | 위 «월말 유지 고객» 수 | 42,744명 |
| 고객당 매출 (`revenue_per_customer`) | 월 청구 합계 ÷ 월말 유지 고객 수 | 33,615원 |
| 상담 만족도 (`consultation_satisfaction`) | 응답 문의 중 csat 4점 이상 비율 | 69.18% |

---

## 데이터 신선도

| 표 | 범위 | 비고 |
|---|---|---|
| `usage_monthly` | 2025-01 ~ 2025-12 | 한 줄 = 고객 × 월 |
| `customers` | 가입일 2018-12-25 ~ 2025-12-31 | 해지일은 2025-01-01 ~ 2025-12-31 |
| `support_tickets` | 접수일 2025-01-01 ~ 2025-12-31 | |
| `csat_survey` | 설문일 2025-01-02 ~ 2026-01-03 | 설문은 접수 1~3일 뒤에 온다 |

```sql
SELECT MAX(created_date) AS latest_ticket FROM support_tickets
```

---

## 참고 파일 안내

| 영역 | 파일 | 쓰는 때 |
|---|---|---|
| 개체 | `references/entities.md` | ID 형식, 연결 키, 개체 사이 관계 |
| 지표 | `references/metrics.md` | 6개 지표의 정의서 id, 공식, 주의 |
| 고객 · 청구 | `references/tables/customers-billing.md` | `customers`, `usage_monthly` 열과 조회 예 |
| 상담 · 만족도 | `references/tables/support.md` | `support_tickets`, `csat_survey`, `agents` 열과 조회 예 |

---

## 자주 쓰는 조회

### 월별 상담 만족도 (접수일 기준, 퇴사 상담원 제외)
```sql
SELECT strftime(t.created_date, '%Y-%m') AS month,
       COUNT(*) AS responded,
       COUNT(*) FILTER (WHERE s.csat >= 4) AS satisfied,
       ROUND(100.0 * COUNT(*) FILTER (WHERE s.csat >= 4) / COUNT(*), 2) AS satisfaction_pct
FROM support_tickets t
JOIN csat_survey s ON s.ticket_id = t.ticket_id
WHERE t.agent_id NOT IN (SELECT agent_id FROM agents WHERE is_resigned)
GROUP BY 1 ORDER BY 1
```

### 월말 유지 고객 수와 월 청구 합계 (2025-12)
```sql
WITH retained AS (
  SELECT customer_id FROM customers
  WHERE signup_date <= DATE '2025-12-31'
    AND (churn_date IS NULL OR churn_date > DATE '2025-12-31'))
SELECT COUNT(*) AS retained_customers, SUM(u.billing_amount) AS billing_total
FROM retained r
JOIN usage_monthly u ON u.customer_id = r.customer_id AND u.year_month = '2025-12'
```

---

## 문제 해결 · 자주 하는 실수

- **청구와 문의를 `customer_id`로만 조인하면 금액이 부풀어 오른다.** 문의 건수만큼 청구 행이 복제된다(12월 합계가 20.5% 큼). 문의는 `GROUP BY customer_id`로 먼저 줄인 뒤 조인하거나 `IN` · `EXISTS`를 쓴다.
- **`customers.is_churned`로 지난 달 유지 고객을 세지 않는다.** 지금 시점의 상태라 1월이 37,093명으로 나온다(정답 43,297명). 날짜(`signup_date`, `churn_date`)로 센다.
- **`usage_monthly.is_active`는 구별 열이 아니다.** 523,825행이 전부 TRUE다. 활성은 `app_login_count >= 8`.
- **`agents.is_resigned`도 지금 시점의 상태**이고 퇴사일이 없다. 퇴사 이전 달의 문의에도 똑같이 적용된다.
- **상담 만족도의 월은 문의 접수일 기준**이다. 설문일로 나누면 2026-01에 52건이 부분 월로 생긴다.
- **`customer_name`은 고유하지 않다**(50,000명에 이름 22,742개). 이름으로 조인하지 않는다.
- **`customers.visitor_id`는 6,485명만 있다.** 웹 방문 분석에 고객 전체를 기대하지 않는다.
- **12월 31일 가입자가 104명**으로 다른 달(9~22명)보다 훨씬 많다. «말일 이전»을 `<`로 쓰는지 `<=`로 쓰는지에 따라 12월 값이 달라진다. 정의서는 `<=`.

### 성능
- `usage_monthly`(약 52만 행), `sessions`(약 29만 행), `funnel_events`(약 30만 행)는 조인 전에 월 · 고객 조건으로 줄인다.
