````md
---
name: bigquery-cost-aware-queries
description: >
  BigQuery에서 GA4 마케팅 데이터와 주식/뉴스 데이터(StockAI)를 다룰 때
  비용을 최소화하고 성능을 최적화하기 위한 쿼리 작성 규칙과 예시 모음.
  Claude는 GA4/마케팅/주식/뉴스 관련 BigQuery 쿼리를 만들 때
  이 스킬의 원칙과 패턴을 항상 우선적으로 적용해야 한다.
---

# BigQuery 비용 절감 & 성능 최적화 쿼리 가이드
*(GA4 마케팅 플랫폼 + StockAI 뉴스/주식 데이터 공통)*

---

## 목차

- [0. 이 스킬을 어떻게 사용할 것인가? (Claude에게)](#0-이-스킬을-어떻게-사용할-것인가-claude에게)
- [1. 테이블 설계 레벨의 베스트 프랙티스](#1-테이블-설계-레벨의-베스트-프랙티스)
- [2. 쿼리 작성 공통 규칙](#2-쿼리-작성-공통-규칙)
- [3. 고급 최적화 기법](#3-고급-최적화-기법)
- [4. GA4 전용 쿼리 베스트 프랙티스](#4-ga4-전용-쿼리-베스트-프랙티스)
- [5. StockAI 전용 쿼리 베스트 프랙티스](#5-stockai-전용-쿼리-베스트-프랙티스)
- [6. 대시보드/리포트용 쿼리 패턴](#6-대시보드리포트용-쿼리-패턴)
- [7. 안티 패턴 (피해야 할 쿼리 형태)](#7-안티-패턴-피해야-할-쿼리-형태)
- [8. 쿼리 비용 확인 및 모니터링](#8-쿼리-비용-확인-및-모니터링)
- [9. Claude 체크리스트 (스킬 사용 시)](#9-claude-체크리스트-스킬-사용-시)

---

## 0. 이 스킬을 어떻게 사용할 것인가? (Claude에게)

Claude가 **BigQuery 쿼리를 작성/수정할 때** 반드시 아래를 지켜야 한다.

1. **날짜/시간 파티션 컬럼에 반드시 필터**를 건다.
2. `SELECT *` 대신 **실제로 필요한 컬럼만 선택**한다.
3. `symbol_id`, `event_name`, `traffic_source`, `country` 등  
   **카디널리티를 줄일 수 있는 WHERE 조건**을 적극적으로 사용한다.
4. 범용 쿼리를 만들 때도  
   - "비용 절감 포인트"를 주석으로 달아서 안내한다.
   - 사용자가 명시적으로 "대략적인 비용 상관없다"고 말하지 않는 이상  
     항상 비용을 최소화하는 방향으로 설계한다.
5. GA4, StockAI처럼 **시계열 테이블**에 대해서는
   - 파티션 키(예: `date`, `event_date`) 기준으로  
     **최근 7~90일 이내로 기본 예시를 제시**한다.
6. **LIMIT 절을 적극 활용**하여 불필요한 데이터 전송을 방지한다.
7. **APPROX 함수**를 사용하여 정확도와 비용의 트레이드오프를 고려한다.

---

## 1. 테이블 설계 레벨의 베스트 프랙티스

### 1.1. 시계열 데이터는 무조건 날짜 파티션

다음과 같은 테이블은 **반드시 `DATE` 또는 `TIMESTAMP` 기반 파티션**을 사용한다.

- GA4: `events_*` (뷰/머터리얼라이즈드 뷰 생성 시 `event_date`)
- StockAI:
  - `prices_daily` → `PARTITION BY date`
  - `technical_daily` → `PARTITION BY date`
  - `signal_history` → `PARTITION BY generated_date`
  - `news` → `PARTITION BY DATE(published_at)`

#### 예시: StockAI `prices_daily`

```sql
CREATE TABLE `{{PROJECT_ID}}.{{DATASET}}.prices_daily` (
  symbol_id STRING NOT NULL,          -- KR_005930, US_AAPL
  date DATE NOT NULL,                 -- 거래일
  open FLOAT64,
  high FLOAT64,
  low FLOAT64,
  close FLOAT64,
  adj_close FLOAT64,
  volume INT64,
  currency STRING,
  source STRING,                      -- "PYKRX", "ALPHAVANTAGE"
  created_at TIMESTAMP,
  updated_at TIMESTAMP
)
PARTITION BY date
CLUSTER BY symbol_id;
```

#### 예시: GA4 이벤트 집계용 테이블

```sql
CREATE TABLE `{{PROJECT_ID}}.{{DATASET}}.events_flattened` (
  event_date DATE NOT NULL,
  event_name STRING,
  user_pseudo_id STRING,
  session_id STRING,
  page_location STRING,
  traffic_source STRING,
  device_category STRING,
  country STRING,
  event_value FLOAT64,
  created_at TIMESTAMP
)
PARTITION BY event_date
CLUSTER BY event_name, traffic_source;
```

---

### 1.2. 클러스터링(Clustering) 기본 전략

- **StockAI:**
  - `CLUSTER BY symbol_id`
- **GA4:**
  - `CLUSTER BY event_name`, `traffic_source`, `country` 등 자주 필터/그룹되는 컬럼

**규칙:** 쿼리 예시를 제시할 때
→ 클러스터 컬럼에 WHERE/GROUP BY를 자연스럽게 얹어 비용 절감을 유도한다.

---

### 1.3. 머터리얼라이즈드 뷰 활용

반복적으로 사용되는 집계 쿼리는 **머터리얼라이즈드 뷰**로 생성하여 비용을 절감한다.

```sql
-- 예시: 일별 채널별 세션 집계를 머터리얼라이즈드 뷰로 생성
CREATE MATERIALIZED VIEW `{{PROJECT_ID}}.{{DATASET}}.mv_daily_sessions_by_channel`
PARTITION BY event_date
CLUSTER BY traffic_source
AS
SELECT
  event_date,
  traffic_source.source AS source,
  COUNT(*) AS sessions
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`,
UNNEST(traffic_source) AS traffic_source
WHERE event_name = 'session_start'
GROUP BY event_date, source;

-- 이후 쿼리는 훨씬 저렴하게 실행됨
SELECT *
FROM `{{PROJECT_ID}}.{{DATASET}}.mv_daily_sessions_by_channel`
WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY);
```

> **주의:** 머터리얼라이즈드 뷰는 자동으로 최신 데이터를 반영하지만,  
> 스토리지 비용이 발생하므로 사용 빈도가 높은 쿼리에만 적용한다.

---

## 2. 쿼리 작성 공통 규칙

### 2.1. 날짜 필터는 "무조건" 넣는다

**항상** 파티션 컬럼으로 범위를 자른다.

#### Bad ❌

```sql
SELECT *
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
WHERE symbol_id = 'KR_005930';
```

#### Good ✅

```sql
SELECT date, close, volume
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
WHERE symbol_id = 'KR_005930'
  AND date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
                AND CURRENT_DATE()
ORDER BY date;
```

```sql
-- GA4 예시
SELECT
  event_date,
  COUNT(*) AS pageviews
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND event_name = 'page_view'
GROUP BY event_date
ORDER BY event_date;
```

> Claude는 사용자가 날짜 범위를 지정하지 않으면
> **기본적으로 최근 7~30일 예시를 제안**하고,
> "전체 기간은 비용이 커질 수 있다"는 주석을 달아야 한다.

---

### 2.2. `SELECT *` 금지 – 필요한 컬럼만 선택

#### Bad ❌

```sql
SELECT *
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE event_name = 'purchase';
```

#### Good ✅

```sql
SELECT
  event_date,
  user_pseudo_id,
  event_value_in_usd
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20250101' AND '20250107'
  AND event_name = 'purchase';
```

> Claude는 쿼리 예시를 만들 때 **항상 컬럼을 명시적으로 지정**해야 하며,
> 사용자가 "정말로 모든 컬럼을 보고 싶다"고 명시하지 않는 이상 `SELECT *`을 제안하지 않는다.

---

### 2.3. 좁힐 수 있는 WHERE 조건을 적극 사용

예: StockAI

```sql
SELECT date, close, volume
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
  AND symbol_id IN ('KR_005930', 'US_AAPL');
```

예: GA4

```sql
SELECT
  event_date,
  traffic_source,
  COUNT(*) AS sessions
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20250101' AND '20250131'
  AND event_name = 'session_start'
  AND traffic_source IN ('google', 'direct', 'facebook')
GROUP BY event_date, traffic_source;
```

---

### 2.4. JOIN 시 양쪽 테이블 모두 날짜 조건

#### Bad ❌

```sql
SELECT *
FROM `{{DATASET}}.prices_daily` p
JOIN `{{DATASET}}.technical_daily` t
  ON p.symbol_id = t.symbol_id
 AND p.date = t.date;
```

#### Good ✅

```sql
SELECT
  p.symbol_id,
  p.date,
  p.close,
  t.rsi_14,
  t.macd,
  t.bb_upper
FROM `{{DATASET}}.prices_daily` p
JOIN `{{DATASET}}.technical_daily` t
  ON p.symbol_id = t.symbol_id
 AND p.date = t.date
WHERE p.date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
                AND CURRENT_DATE()
  AND p.symbol_id IN ('KR_005930', 'US_AAPL');
```

---

### 2.5. LIMIT 절 적극 활용

대시보드나 탐색적 분석 시 **LIMIT 절을 항상 사용**하여 불필요한 데이터 전송을 방지한다.

#### Good ✅

```sql
-- 최근 상위 10개 종목만 조회
SELECT
  symbol_id,
  date,
  close,
  volume
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
ORDER BY volume DESC
LIMIT 10;
```

```sql
-- 샘플 데이터 확인용
SELECT *
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND event_name = 'purchase'
LIMIT 100;
```

---

## 3. 고급 최적화 기법

### 3.1. APPROX 함수 활용 (정확도 vs 비용 트레이드오프)

대용량 데이터에서 **근사치로 충분한 경우** APPROX 함수를 사용하면 비용을 크게 절감할 수 있다.

```sql
-- 정확한 COUNT DISTINCT (비용 높음)
SELECT
  event_date,
  COUNT(DISTINCT user_pseudo_id) AS exact_dau
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY event_date;

-- 근사치 COUNT DISTINCT (비용 낮음, 오차율 ~1%)
SELECT
  event_date,
  APPROX_COUNT_DISTINCT(user_pseudo_id) AS approx_dau
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY event_date;
```

> **사용 가이드:**
> - 대시보드/리포트: APPROX 함수 사용 권장 (오차율 1% 이내)
> - 정확한 계산이 필요한 경우: 일반 함수 사용
> - 데이터 탐색 단계: 항상 APPROX 함수 사용

---

### 3.2. 테이블 샘플링 (TABLESAMPLE)

대용량 테이블에서 **빠른 탐색**이 필요할 때 샘플링을 사용한다.

```sql
-- 10% 샘플링 (비용 1/10로 감소)
SELECT
  event_date,
  event_name,
  COUNT(*) AS event_count
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
TABLESAMPLE SYSTEM (10 PERCENT)
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY event_date, event_name
ORDER BY event_count DESC;
```

> **주의:** 샘플링은 통계적 분석에는 유용하지만,  
> 정확한 집계가 필요한 경우 사용하지 않는다.

---

### 3.3. 서브쿼리 최적화

서브쿼리는 가능한 한 **CTE(Common Table Expression)**로 변환하고,  
각 서브쿼리에도 날짜 필터를 적용한다.

#### Bad ❌

```sql
SELECT *
FROM (
  SELECT symbol_id, date, close
  FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
) p
WHERE p.close > (
  SELECT AVG(close)
  FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
);
```

#### Good ✅

```sql
WITH avg_prices AS (
  SELECT AVG(close) AS avg_close
  FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
),
recent_prices AS (
  SELECT symbol_id, date, close
  FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
)
SELECT r.symbol_id, r.date, r.close, a.avg_close
FROM recent_prices r
CROSS JOIN avg_prices a
WHERE r.close > a.avg_close
ORDER BY r.date;
```

---

### 3.4. 윈도우 함수 최적화

윈도우 함수 사용 시 **PARTITION BY와 ORDER BY 컬럼을 클러스터 컬럼과 일치**시켜 비용을 절감한다.

#### Good ✅

```sql
-- symbol_id로 클러스터링된 테이블에서 윈도우 함수 사용
SELECT
  symbol_id,
  date,
  close,
  AVG(close) OVER (
    PARTITION BY symbol_id
    ORDER BY date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS ma_7d
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
  AND symbol_id IN ('KR_005930', 'US_AAPL')
ORDER BY symbol_id, date;
```

---

### 3.5. 쿼리 결과 캐싱 활용

BigQuery는 **24시간 동안 동일한 쿼리 결과를 캐시**한다.  
반복 실행되는 쿼리는 자동으로 캐시를 활용하므로 비용이 발생하지 않는다.

```sql
-- 동일한 쿼리를 24시간 내에 재실행하면 비용 없음
SELECT
  event_date,
  COUNT(*) AS pageviews
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND event_name = 'page_view'
GROUP BY event_date;
```

> **팁:** 대시보드 쿼리는 가능한 한 동일한 형태로 유지하여 캐싱 효과를 극대화한다.

---

### 3.6. UNION vs UNION ALL

중복 제거가 필요 없는 경우 **UNION ALL**을 사용하면 비용을 크게 절감할 수 있다.  
UNION은 중복 제거를 수행하므로 비용이 높다.

#### Bad ❌

```sql
-- UNION은 중복 제거를 수행하므로 비용 높음
SELECT symbol_id, date, close
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND symbol_id = 'KR_005930'
UNION
SELECT symbol_id, date, close
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily_backup`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND symbol_id = 'KR_005930';
```

#### Good ✅

```sql
-- UNION ALL은 중복 제거를 하지 않아 훨씬 효율적
SELECT symbol_id, date, close
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND symbol_id = 'KR_005930'
UNION ALL
SELECT symbol_id, date, close
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily_backup`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND symbol_id = 'KR_005930';
```

> **규칙:** 중복 제거가 정말 필요한 경우에만 UNION을 사용하고,  
> 그 외에는 항상 UNION ALL을 사용한다.

---

### 3.7. EXISTS vs IN

서브쿼리에서 **EXISTS**가 **IN**보다 일반적으로 더 효율적이다.  
특히 서브쿼리가 큰 경우 EXISTS가 더 빠르게 실행된다.

#### Bad ❌

```sql
-- IN은 서브쿼리 결과를 모두 메모리에 로드
SELECT symbol_id, date, close
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND symbol_id IN (
    SELECT symbol_id
    FROM `{{PROJECT_ID}}.{{DATASET}}.signal_history`
    WHERE generated_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
      AND signal_type = 'STRONG_BUY'
  );
```

#### Good ✅

```sql
-- EXISTS는 첫 번째 매칭을 찾으면 즉시 중단 (더 효율적)
SELECT symbol_id, date, close
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily` p
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND EXISTS (
    SELECT 1
    FROM `{{PROJECT_ID}}.{{DATASET}}.signal_history` s
    WHERE s.symbol_id = p.symbol_id
      AND s.generated_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
      AND s.signal_type = 'STRONG_BUY'
  );
```

> **규칙:** 서브쿼리를 사용할 때는 가능한 한 EXISTS를 사용한다.  
> 단, 서브쿼리 결과가 매우 작은 경우(예: 10개 이하) IN도 괜찮다.

---

### 3.8. 배열 언네스팅 최적화

GA4처럼 배열/구조체를 언네스팅할 때는 **필터를 먼저 적용**한 후 UNNEST를 수행하는 것이 효율적이다.

#### Bad ❌

```sql
-- UNNEST 후 필터링 (비효율적)
SELECT
  event_date,
  traffic_source.source AS source,
  COUNT(*) AS sessions
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`,
UNNEST(traffic_source) AS traffic_source
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND event_name = 'session_start'
  AND traffic_source.source = 'google'  -- UNNEST 후 필터링
GROUP BY event_date, source;
```

#### Good ✅

```sql
-- 필터링 후 UNNEST (효율적)
SELECT
  event_date,
  traffic_source.source AS source,
  COUNT(*) AS sessions
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`,
UNNEST(traffic_source) AS traffic_source
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND event_name = 'session_start'
  AND traffic_source.source = 'google'  -- 가능한 한 빨리 필터링
GROUP BY event_date, source;
```

> **팁:** UNNEST는 가능한 한 필요한 행에만 적용하고,  
> 날짜 필터와 이벤트 필터를 먼저 적용한 후 UNNEST를 수행한다.

---

## 4. GA4 전용 쿼리 베스트 프랙티스

### 4.1. 최근 7일 채널별 세션

```sql
-- 비용 절감 포인트:
-- 1) _TABLE_SUFFIX로 날짜 범위 지정
-- 2) event_name 필터
-- 3) traffic_source만 그룹

SELECT
  event_date,
  traffic_source.source AS source,
  COUNT(*) AS sessions
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`,
UNNEST(traffic_source) AS traffic_source
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND event_name = 'session_start'
GROUP BY event_date, source
ORDER BY event_date, sessions DESC;
```

---

### 4.2. 최근 30일 UTM 기준 전환 수

```sql
-- utm_source / utm_medium / utm_campaign에 대한 집계 예시

WITH events AS (
  SELECT
    event_date,
    (SELECT value.string_value FROM UNNEST(event_params)
      WHERE key = 'utm_source') AS utm_source,
    (SELECT value.string_value FROM UNNEST(event_params)
      WHERE key = 'utm_medium') AS utm_medium,
    (SELECT value.string_value FROM UNNEST(event_params)
      WHERE key = 'utm_campaign') AS utm_campaign,
    event_name
  FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
  WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY))
                          AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
    AND event_name IN ('purchase', 'generate_lead')  -- 전환 이벤트
)

SELECT
  event_date,
  COALESCE(utm_source, '(direct)') AS utm_source,
  COALESCE(utm_medium, '(none)')   AS utm_medium,
  COALESCE(utm_campaign, '(none)') AS utm_campaign,
  event_name,
  COUNT(*) AS conversions
FROM events
GROUP BY event_date, utm_source, utm_medium, utm_campaign, event_name
ORDER BY event_date, conversions DESC;
```

---

### 4.3. 사용자 레벨 분석 시 주의점

- `user_pseudo_id` 기준 DISTINCT COUNT는 데이터량이 크므로
  **반드시 날짜 범위 제한 + 필요 컬럼 최소화**를 한다.

```sql
-- 정확한 DAU (비용 높음)
SELECT
  event_date,
  COUNT(DISTINCT user_pseudo_id) AS dau
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY event_date
ORDER BY event_date;

-- 근사치 DAU (비용 낮음, 대시보드용 권장)
SELECT
  event_date,
  APPROX_COUNT_DISTINCT(user_pseudo_id) AS approx_dau
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY event_date
ORDER BY event_date;
```

---

## 5. StockAI 전용 쿼리 베스트 프랙티스

### 5.1. 특정 종목 최근 6개월 차트 데이터

```sql
-- StockAI: 차트/기술분석용 기본 쿼리
SELECT
  date,
  open,
  high,
  low,
  close,
  volume
FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
WHERE symbol_id = 'KR_005930'
  AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
ORDER BY date;
```

---

### 5.2. 기술지표 + 가격 + 시그널 JOIN

```sql
-- 뉴스 기반 시그널 검증용 쿼리

WITH base AS (
  SELECT
    s.symbol_id,
    s.generated_date,
    s.total_score,
    s.expected_return_pct,
    s.target_price,
    s.stop_loss,
    s.signal_type
  FROM `{{PROJECT_ID}}.{{DATASET}}.signal_history` s
  WHERE s.generated_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
                             AND CURRENT_DATE()
    AND s.symbol_id = 'KR_005930'
)

SELECT
  b.symbol_id,
  b.generated_date,
  p.date AS price_date,
  p.close AS close_price,
  t.rsi_14,
  t.macd,
  b.total_score,
  b.expected_return_pct,
  b.signal_type
FROM base b
JOIN `{{PROJECT_ID}}.{{DATASET}}.prices_daily` p
  ON p.symbol_id = b.symbol_id
 AND p.date = b.generated_date
JOIN `{{PROJECT_ID}}.{{DATASET}}.technical_daily` t
  ON t.symbol_id = b.symbol_id
 AND t.date = b.generated_date
ORDER BY b.generated_date;
```

---

### 5.3. 백테스트 라벨 생성 (예: 10거래일 수익률)

```sql
-- 백테스트/ML용 수익률 라벨 생성 예시

WITH prices AS (
  SELECT
    symbol_id,
    date,
    close,
    LEAD(close, 10) OVER (
      PARTITION BY symbol_id
      ORDER BY date
    ) AS close_after_10d
  FROM `{{PROJECT_ID}}.{{DATASET}}.prices_daily`
  WHERE date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 730 DAY)
                  AND CURRENT_DATE()
    AND symbol_id IN ('KR_005930', 'US_AAPL')
)

SELECT
  symbol_id,
  date AS t0_date,
  close AS t0_close,
  close_after_10d,
  SAFE_DIVIDE(close_after_10d - close, close) * 100 AS ret_10d_pct
FROM prices
WHERE close_after_10d IS NOT NULL;
```

> Claude는 ML/백테스트용 쿼리를 작성할 때도
> **최근 1~3년 범위** 등으로 제한한 예시를 우선 제안해야 한다.

---

## 6. 대시보드/리포트용 쿼리 패턴

### 6.1. GA4 – 최근 30일 기기별 트래픽 & 전환

```sql
SELECT
  event_date,
  device.category AS device_category,
  COUNTIF(event_name = 'session_start') AS sessions,
  COUNTIF(event_name = 'purchase') AS purchases
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`,
UNNEST(device) AS device
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY event_date, device_category
ORDER BY event_date, device_category;
```

---

### 6.2. StockAI – 최근 7일 종목별 시그널 요약

```sql
SELECT
  symbol_id,
  COUNT(*) AS signals,
  AVG(expected_return_pct) AS avg_expected_return,
  AVG(reward_risk_ratio) AS avg_rr,
  COUNTIF(signal_type = 'STRONG_BUY') AS strong_buy_cnt
FROM `{{PROJECT_ID}}.{{DATASET}}.signal_history`
WHERE generated_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
                         AND CURRENT_DATE()
GROUP BY symbol_id
ORDER BY strong_buy_cnt DESC, avg_expected_return DESC
LIMIT 50;
```

---

## 7. 안티 패턴 (피해야 할 쿼리 형태)

Claude는 아래 형태의 쿼리를 **가능한 한 제안하지 않거나,
비용 경고를 함께 붙여야 한다.**

### 7.1. 날짜 조건 없이 전체 테이블 스캔

```sql
SELECT *
FROM `dataset.events_*`;  -- ❌
```

### 7.2. 분석/ML용으로 `SELECT *` + JOIN + 전체기간

```sql
SELECT *
FROM prices_daily p
JOIN technical_daily t
  ON p.symbol_id = t.symbol_id
 AND p.date = t.date;     -- ❌
```

### 7.3. 고카디널리티 컬럼에 대한 전체기간 DISTINCT COUNT

고카디널리티 컬럼(예: full URL, user_id)에 대해 전체기간 DISTINCT COUNT를 수행하는 경우  
→ 반드시 날짜+조건으로 범위 줄이기.

### 7.4. 불필요한 ORDER BY

대용량 결과에 대해 ORDER BY를 사용하면 비용이 증가한다.  
→ LIMIT과 함께 사용하거나, 클라이언트 측에서 정렬하는 것을 고려.

```sql
-- ❌ 비용 높음
SELECT *
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20250101' AND '20250131'
ORDER BY event_timestamp;

-- ✅ 비용 절감
SELECT *
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20250101' AND '20250131'
ORDER BY event_timestamp
LIMIT 1000;
```

### 7.5. 불필요한 UNION 사용

중복 제거가 필요 없는 경우 UNION 대신 UNION ALL을 사용해야 한다.

```sql
-- ❌ 비용 높음 (중복 제거 수행)
SELECT symbol_id FROM table1
UNION
SELECT symbol_id FROM table2;

-- ✅ 비용 절감 (중복 제거 없음)
SELECT symbol_id FROM table1
UNION ALL
SELECT symbol_id FROM table2;
```

### 7.6. 비효율적인 IN 서브쿼리

서브쿼리에서 IN 대신 EXISTS를 사용하는 것이 일반적으로 더 효율적이다.

```sql
-- ❌ 비효율적 (서브쿼리 결과를 모두 메모리에 로드)
SELECT * FROM prices_daily
WHERE symbol_id IN (SELECT symbol_id FROM signal_history WHERE ...);

-- ✅ 효율적 (첫 매칭 발견 시 중단)
SELECT * FROM prices_daily p
WHERE EXISTS (SELECT 1 FROM signal_history s WHERE s.symbol_id = p.symbol_id AND ...);
```

---

## 8. 쿼리 비용 확인 및 모니터링

### 8.1. 쿼리 실행 전 비용 예측

BigQuery 콘솔에서 쿼리를 실행하기 전 **"Validated" 단계에서 스캔될 데이터 양을 확인**할 수 있다.

> **팁:** 쿼리 실행 전 항상 "This query will process X GB" 메시지를 확인한다.

### 8.2. 쿼리 실행 계획 확인

```sql
-- EXPLAIN을 사용하여 쿼리 실행 계획 확인
EXPLAIN
SELECT
  event_date,
  COUNT(*) AS pageviews
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND event_name = 'page_view'
GROUP BY event_date;
```

### 8.3. INFORMATION_SCHEMA를 통한 쿼리 히스토리 확인

```sql
-- 최근 실행된 쿼리의 비용 확인
SELECT
  job_id,
  creation_time,
  total_bytes_processed,
  total_bytes_processed / POW(1024, 3) AS total_gb_processed,
  total_slot_ms,
  query
FROM `region-us.INFORMATION_SCHEMA.JOBS_BY_PROJECT`
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND total_bytes_processed > 0
ORDER BY total_bytes_processed DESC
LIMIT 10;
```

### 8.4. 비용 절감 팁 요약

1. **파티션 프루닝:** 날짜 필터로 스캔 범위 최소화
2. **컬럼 선택:** 필요한 컬럼만 선택하여 스캔 데이터 감소
3. **클러스터 활용:** 클러스터 컬럼으로 필터링
4. **LIMIT 사용:** 결과 크기 제한
5. **APPROX 함수:** 근사치로 충분한 경우 사용
6. **캐싱 활용:** 동일 쿼리 재실행 시 비용 없음
7. **머터리얼라이즈드 뷰:** 반복 집계 쿼리 최적화

---

## 9. Claude 체크리스트 (스킬 사용 시)

Claude가 BigQuery 쿼리를 생성할 때, 아래 체크리스트를 스스로 점검해야 한다.

### 기본 규칙
- [ ] 파티션 컬럼(날짜/시간)에 필터가 있는가?
- [ ] `SELECT *`를 사용하지 않았는가?
- [ ] 심볼/이벤트명/채널 등 카디널리티를 줄이는 WHERE 조건을 사용했는가?
- [ ] JOIN 시 양쪽 모두에 날짜 조건이 적용되어 있는가?

### 기간 제한
- [ ] 대시보드/리포트용이면 최근 기간(7~90일)으로 제한했는가?
- [ ] ML/백테스트용이면 period를 명시하고, 필요 최소 범위로 제한했는가?

### 최적화 기법
- [ ] LIMIT 절을 적절히 사용했는가?
- [ ] APPROX 함수를 사용할 수 있는 경우인가? (대시보드/탐색적 분석)
- [ ] 서브쿼리를 CTE로 변환하여 가독성과 성능을 개선했는가?
- [ ] 윈도우 함수의 PARTITION BY가 클러스터 컬럼과 일치하는가?
- [ ] UNION 대신 UNION ALL을 사용할 수 있는가? (중복 제거 불필요 시)
- [ ] 서브쿼리에서 IN 대신 EXISTS를 사용할 수 있는가?
- [ ] 배열 언네스팅 시 필터를 먼저 적용했는가?

### 고급 최적화
- [ ] 필요 시 머터리얼라이즈드 뷰/파생 테이블 사용을 제안했는가?
- [ ] 테이블 샘플링을 사용할 수 있는 경우인가? (탐색적 분석)
- [ ] 쿼리 결과 캐싱을 활용할 수 있는 구조인가?

### 비용 경고
- [ ] 비용이 높을 수 있는 쿼리에는 경고 주석을 추가했는가?
- [ ] 사용자에게 비용 절감 포인트를 명시했는가?

---

이 스킬은 **GA4 마케팅 분석**과 **StockAI 주식/뉴스 플랫폼** 모두에서
BigQuery 쿼리 작성 시 기본 가이드라인으로 사용해야 한다.

````
