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

## 0. 이 스킬을 어떻게 사용할 것인가? (Claude에게)

Claude가 **BigQuery 쿼리를 작성/수정할 때** 반드시 아래를 지켜야 한다.

1. **날짜/시간 파티션 컬럼에 반드시 필터**를 건다.
2. `SELECT *` 대신 **실제로 필요한 컬럼만 선택**한다.
3. `symbol_id`, `event_name`, `traffic_source`, `country` 등  
   **카디널리티를 줄일 수 있는 WHERE 조건**을 적극적으로 사용한다.
4. 범용 쿼리를 만들 때도  
   - “비용 절감 포인트”를 주석으로 달아서 안내한다.
   - 사용자가 명시적으로 “대략적인 비용 상관없다”고 말하지 않는 이상  
     항상 비용을 최소화하는 방향으로 설계한다.
5. GA4, StockAI처럼 **시계열 테이블**에 대해서는
   - 파티션 키(예: `date`, `event_date`) 기준으로  
     **최근 7~90일 이내로 기본 예시를 제시**한다.

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
````

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

* StockAI:

  * `CLUSTER BY symbol_id`
* GA4:

  * `CLUSTER BY event_name`, `traffic_source`, `country` 등 자주 필터/그룹되는 컬럼

**규칙:** 쿼리 예시를 제시할 때
→ 클러스터 컬럼에 WHERE/GROUP BY를 자연스럽게 얹어 비용 절감을 유도한다.

---

## 2. 쿼리 작성 공통 규칙

### 2.1. 날짜 필터는 “무조건” 넣는다

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
> “전체 기간은 비용이 커질 수 있다”는 주석을 달아야 한다.

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
> 사용자가 “정말로 모든 컬럼을 보고 싶다”고 명시하지 않는 이상 `SELECT *`을 제안하지 않는다.

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

## 3. GA4 전용 쿼리 베스트 프랙티스

### 3.1. 최근 7일 채널별 세션

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

### 3.2. 최근 30일 UTM 기준 전환 수

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

### 3.3. 사용자 레벨 분석 시 주의점

* user_pseudo_id 기준 DISTINCT COUNT는 데이터량이 크므로
  **반드시 날짜 범위 제한 + 필요 컬럼 최소화**를 한다.

```sql
SELECT
  event_date,
  COUNT(DISTINCT user_pseudo_id) AS dau
FROM `{{PROJECT_ID}}.{{DATASET}}.events_*`
WHERE _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 14 DAY))
                        AND FORMAT_DATE('%Y%m%d', CURRENT_DATE())
GROUP BY event_date
ORDER BY event_date;
```

---

## 4. StockAI 전용 쿼리 베스트 프랙티스

### 4.1. 특정 종목 최근 6개월 차트 데이터

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

### 4.2. 기술지표 + 가격 + 시그널 JOIN

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

### 4.3. 백테스트 라벨 생성 (예: 10거래일 수익률)

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

## 5. 대시보드/리포트용 쿼리 패턴

### 5.1. GA4 – 최근 30일 기기별 트래픽 & 전환

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

### 5.2. StockAI – 최근 7일 종목별 시그널 요약

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

## 6. 안티 패턴 (피해야 할 쿼리 형태)

Claude는 아래 형태의 쿼리를 **가능한 한 제안하지 않거나,
비용 경고를 함께 붙여야 한다.**

1. 날짜 조건 없이 전체 테이블 스캔

   ```sql
   SELECT *
   FROM `dataset.events_*`;  -- ❌
   ```

2. 분석/ML용으로 `SELECT *` + JOIN + 전체기간

   ```sql
   SELECT *
   FROM prices_daily p
   JOIN technical_daily t
     ON p.symbol_id = t.symbol_id
    AND p.date = t.date;     -- ❌
   ```

3. 고카디널리티 컬럼(예: full URL, user_id)에 대해 전체기간 DISTINCT COUNT
   → 반드시 날짜+조건으로 범위 줄이기.

---

## 7. Claude 체크리스트 (스킬 사용 시)

Claude가 BigQuery 쿼리를 생성할 때, 아래 체크리스트를 스스로 점검해야 한다.

* [ ] 파티션 컬럼(날짜/시간)에 필터가 있는가?
* [ ] `SELECT *`를 사용하지 않았는가?
* [ ] 심볼/이벤트명/채널 등 카디널리티를 줄이는 WHERE 조건을 사용했는가?
* [ ] JOIN 시 양쪽 모두에 날짜 조건이 적용되어 있는가?
* [ ] 대시보드/리포트용이면 최근 기간(7~90일)으로 제한했는가?
* [ ] ML/백테스트용이면 period를 명시하고, 필요 최소 범위로 제한했는가?
* [ ] 필요 시 머터리얼라이즈드 뷰/파생 테이블 사용을 제안했는가?

이 스킬은 **GA4 마케팅 분석**과 **StockAI 주식/뉴스 플랫폼** 모두에서
BigQuery 쿼리 작성 시 기본 가이드라인으로 사용해야 한다.


