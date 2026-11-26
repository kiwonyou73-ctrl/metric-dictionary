# 개체 정의와 연결

이 데이터는 통신사 가상 데이터다. 조인 전에 한 줄이 무엇인지(grain)를 먼저 본다(`edata9-data`의 `list_tables`).

## 개체

### 고객
- **정의**: 가입한 사람 1명
- **주 표**: `customers` (50,000행, 한 줄 = 고객 1명)
- **ID**: `customer_id` (`C` + 6자리, 예 `C000001`). 중복 · 빈 값 없음.
- **다른 식별자**: `visitor_id` — 가입 전 웹 방문과 이어진다. 6,485명만 값이 있고 모두 `sessions.visitor_id`와 맞는다. `customer_name`은 고유하지 않다(이름 22,742개).
- **상태**: `is_churned`(지금 시점), `churn_date`(해지일, 2025년), `signup_date`
- **일반 제외**: 없음. 월 청구와 유지 고객 지표는 «월말 유지 고객»으로 좁힌다.

### 문의
- **정의**: 고객이 상담 채널로 접수해 상담원 1명이 맡은 건
- **주 표**: `support_tickets` (32,789행)
- **ID**: `ticket_id` (`T` + 6자리)
- **연결**: `customer_id`, `agent_id`

### 설문
- **정의**: 문의 1건에 대한 만족도 응답
- **주 표**: `csat_survey` (9,687행)
- **ID**: `survey_id` (`S` + 6자리)
- **연결**: `ticket_id`(문의당 설문 최대 1건), `customer_id`(문의의 고객과 모두 같다)

### 상담원
- **정의**: 상담 직원 1명
- **주 표**: `agents` (120행)
- **ID**: `agent_id` (`AG001` 형식). 120명 전원이 문의를 맡고 있다.
- **상태**: `is_resigned`(지금 시점, 13명)

## 연결 요약

| 부모 | 자식 | 키 | 관계 | 확인 |
|---|---|---|---|---|
| customers | support_tickets | customer_id | 1 : N | 없는 값 0건 |
| agents | support_tickets | agent_id | 1 : N | 없는 값 0건 |
| support_tickets | csat_survey | ticket_id | 1 : 0~1 | 문의 1건에 설문 최대 1건 |
| customers | csat_survey | customer_id | 1 : N | 문의의 고객과 전부 같다 |
| customers | usage_monthly | customer_id | 1 : N | 49,985명이 이용 행을 갖는다(15명은 없음) |

`csat_survey`에는 `agent_id`가 없다. 설문을 상담원에 잇으려면 `support_tickets`를 거친다.

## 같은 말이 다른 것을 가리키는 경우

- «고객 수»: 전체 가입자(50,000)인지, 월말 유지 고객(12월 42,744)인지, 그 달 청구가 있는 고객(12월 43,412)인지 먼저 정한다. 지표 정의서는 월말 유지 고객을 쓴다.
- «만족도»: 설문에 응답한 문의 기준이다. 상담을 받은 고객 전체의 만족도가 아니다.
