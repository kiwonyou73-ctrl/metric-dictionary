# 상담 · 만족도 표

상담 문의, 만족도 설문, 상담원을 다룬다.

---

## 빠른 참고

### 사업 맥락
고객이 전화 · 챗봇 · 앱 · 이메일로 문의를 접수하면 상담원 1명이 맡는다. 일부 문의(약 30%)에 만족도 설문이 달린다. 만족도 지표는 설문에 응답한 문의만 대상이라서, 응답하지 않은 고객의 만족도는 알 수 없다.

### 헷갈리는 말
- **«만족도»**: 설문 응답 문의 중 csat 4점 이상 비율. 상담받은 고객 전체의 만족도가 아니다.
- **«상담원»**: `agents`의 한 행. `is_resigned`는 지금 시점의 퇴사 여부다.

### 표준 조건
상담 만족도는 퇴사한 상담원의 문의를 뺀다.
```sql
WHERE t.agent_id NOT IN (SELECT agent_id FROM agents WHERE is_resigned)
```

---

## 표

### support_tickets
**위치**: `data/support_tickets.parquet` (32,789행)
**설명**: 문의 1건당 한 행
**기본 키**: `ticket_id`
**범위**: 접수일 2025-01-01 ~ 2025-12-31

| 열 | 형식 | 뜻 | 메모 |
|---|---|---|---|
| **ticket_id** | VARCHAR | 문의 ID (`T000001`) | 중복 없음 |
| customer_id | VARCHAR | 문의한 고객 | `customers`에 없는 값 0건 |
| agent_id | VARCHAR | 담당 상담원 | `agents`에 없는 값 0건, 빈 값 없음 |
| created_date | DATE | 접수일 | 상담 만족도의 기준 날짜 |
| contact_channel | VARCHAR | 채널 | 전화 · 챗봇 · 앱 · 이메일 |
| ticket_category | VARCHAR | 분류 | 요금 · 품질 · 가입변경 · 해지 · 기타 |
| status | VARCHAR | 상태 | 완료 · 진행중 · 보류 |
| duration_min · resolution_hours | | 상담 시간 · 해결 시간 | |
| is_recontact | BOOLEAN | 재문의 여부 | |

### csat_survey
**위치**: `data/csat_survey.parquet` (9,687행)
**설명**: 문의에 대한 만족도 설문 응답 1건당 한 행
**기본 키**: `survey_id`
**범위**: 설문일 2025-01-02 ~ 2026-01-03

| 열 | 형식 | 뜻 | 메모 |
|---|---|---|---|
| **survey_id** | VARCHAR | 설문 ID (`S000001`) | |
| ticket_id | VARCHAR | 대상 문의 | 중복 없음 = 문의 1건에 설문 최대 1건 |
| customer_id | VARCHAR | 응답 고객 | 문의의 고객과 모두 같다 |
| survey_date | DATE | 설문일 | 문의 접수 1~3일 뒤. 2026-01은 52건뿐인 부분 월 |
| csat | INTEGER | 만족도 1~5 | 5점 46%, 1점 15%로 양쪽에 몰린다. 평균 3.79 |
| nps | INTEGER | 추천 의향 0~10 | csat와 상관 0.92라 사실상 같은 정보 |
| comment | VARCHAR | 자유 의견 | 70%가 비어 있고, 값은 정해진 문장 12개뿐이다(점수에서 만들어진 것으로 보임) |

### agents
**위치**: `data/agents.parquet` (120행)
**설명**: 상담원 1명당 한 행
**기본 키**: `agent_id`

| 열 | 형식 | 뜻 | 메모 |
|---|---|---|---|
| **agent_id** | VARCHAR | 상담원 ID (`AG001`) | 120명 전원이 문의를 맡았다(130~446건) |
| team | VARCHAR | 팀 | 일반 37 · 가입팀 23 · 요금팀 23 · 품질팀 19 · 해지방어팀 18 |
| is_resigned | BOOLEAN | 퇴사 여부 | **지금 시점의 상태.** 13명. 퇴사일 열은 없다 |
| hire_date · job_satisfaction · burnout_score · training_hours | | 입사일 등 | |

**연결**
- 문의 → 설문: `csat_survey.ticket_id = support_tickets.ticket_id`
- 문의 → 상담원: `support_tickets.agent_id = agents.agent_id`
- `csat_survey`에는 `agent_id`가 없다. 설문을 상담원에 잇으려면 `support_tickets`를 거친다.

---

## 조회 예

### 12월 팀별 상담 만족도
```sql
SELECT a.team,
       COUNT(*) AS responded,
       ROUND(AVG(s.csat), 2) AS avg_csat,
       ROUND(100.0 * COUNT(*) FILTER (WHERE s.csat >= 4) / COUNT(*), 1) AS satisfaction_pct
FROM support_tickets t
JOIN csat_survey s ON s.ticket_id = t.ticket_id
JOIN agents a ON a.agent_id = t.agent_id
WHERE t.created_date BETWEEN DATE '2025-12-01' AND DATE '2025-12-31'
  AND NOT a.is_resigned
GROUP BY 1 ORDER BY 4 DESC
-- 결과: 가입팀 73.8% · 요금팀 72.8% · 품질팀 67.8% · 해지방어팀 65.5% · 일반 62.5%
```

### 12월 응답률
```sql
SELECT COUNT(*) AS tickets, COUNT(s.survey_id) AS responded
FROM support_tickets t
LEFT JOIN csat_survey s ON s.ticket_id = t.ticket_id
WHERE t.created_date BETWEEN DATE '2025-12-01' AND DATE '2025-12-31'
  AND t.agent_id NOT IN (SELECT agent_id FROM agents WHERE is_resigned)
```

---

## 자주 하는 실수

1. **설문일로 월을 나눈다.** 12월 말 접수 문의의 설문이 다음 달로 넘어가 2026-01이 부분 월이 된다. 접수일(`created_date`)로 나눈다.
2. **`LEFT JOIN`으로 만족도를 센다.** 응답하지 않은 문의가 분모에 들어가 응답하지 않은 것이 불만족으로 계산된다. 응답한 문의만 `JOIN`한다.
3. **퇴사 상담원 문의를 그대로 둔다.** 상담 만족도는 뺀다. `is_resigned`는 지금 시점의 상태라 퇴사 이전 문의에도 같게 적용된다.
4. **팀별 · 채널별 비교에서 응답 수를 안 본다.** 채널별로 나누면 한 달 응답이 몇십~몇백 건으로 줄어든다. 최소 표본 300을 넘는지 본다.
