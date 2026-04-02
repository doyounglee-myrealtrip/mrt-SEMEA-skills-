---
name: tna-country-analysis
description: 특정 국가의 T&A 공헌이익을 퍼널 6단계(UV→CVR→주문→AOV→CFR→마진율)로 진단하고, 도시→카테고리→상품 레벨로 드릴다운하여 문제를 특정하고 해결 방향을 도출하는 종합 분석 skill. "스페인 T&A 분석해줘", "일본 T&A 공헌이익", "/tna-country-analysis 태국" 등에 사용.
---

# T&A 국가별 공헌이익 종합 분석

> 특정 국가의 T&A 공헌이익을 퍼널 6단계로 진단하고, 문제를 특정하여 해결 방향을 도출한다.

## 사용법

```
/tna-country-analysis 스페인
/tna-country-analysis Japan
/tna-country-analysis 태국
```

## 입력 파라미터

| 파라미터 | 필수 | 설명 |
|---------|:----:|------|
| 국가명 | 필수 | 한글 또는 영문. 한글 입력 시 자동으로 BigQuery `COUNTRY_NM` 값으로 매핑 |

### 한글 → 영문 매핑 (주요 국가)

| 한글 | COUNTRY_NM |
|------|-----------|
| 스페인 | Spain |
| 일본 | Japan |
| 태국 | Thailand |
| 베트남 | Viet Nam |
| 프랑스 | France |
| 이탈리아 | Italy |
| 영국 | United Kingdom |
| 미국 | United States |
| 호주 | Australia |
| 대만 | Taiwan |
| 싱가포르 | Singapore |
| 홍콩 | Hong Kong |
| 터키 | Turkey |
| 크로아티아 | Croatia |
| 포르투갈 | Portugal |
| 그리스 | Greece |
| 체코 | Czech Republic |
| 헝가리 | Hungary |
| 모로코 | Morocco |
| 인도네시아 | Indonesia |

위 목록에 없는 국가는 BigQuery에서 `COUNTRY_NM` 값을 직접 확인한다:
```sql
SELECT DISTINCT COUNTRY_NM FROM mrtdata.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE FPNA_DOMAIN_NM = 'TNA' AND LOWER(COUNTRY_NM) LIKE '%입력값%'
LIMIT 10
```

---

## 분석 프레임워크

### 퍼널 구조 (6단계)

```
유입(상품 상세 UV) → 전환율(CVR) → 주문건수 → 객단가(AOV) → 확정율(CFR) → 마진율 → 공헌이익
```

| 단계 | 지표 | 정의 | 데이터 소스 |
|------|------|------|-----------|
| ① 유입 | 상품 상세페이지 UV (PID 기준) | 해당 국가 T&A 상품을 본 유저 수 | `MART_BIZ_LOG_PID_CONVERSION_D` + `MART_PRODUCT_D` |
| ② 전환율 | CVR = 주문건수 / UV | 본 사람 중 주문한 비율 | ①과 FP&A 조합 |
| ③ 주문건수 | Orders (ORDER_ID 기준) | 절대 주문 볼륨 | `MART_FPNA_NONAIR_PROFIT_D` |
| ④ 객단가 | AOV = GMV / 주문건수 | 건당 평균 거래액 | `MART_FPNA_NONAIR_PROFIT_D` |
| ⑤ 확정율 | CFR = CGMV / GMV | 확정거래액 비중 | `MART_FPNA_NONAIR_PROFIT_D` |
| ⑥ 마진율 | CON_MARGIN / CGMV | 비용 차감 후 남는 비율 | `MART_FPNA_NONAIR_PROFIT_D` |

### 드릴다운 구조 (3단계)

```
Level 1: 도시 (CITY_NM)            ← Phase 1
  └─ Level 2: 카테고리 (CATEGORY_NM) ← Phase 2
       └─ Level 3: 개별 상품 (GID)    ← Phase 2
```

### 시간축

- 모든 비교는 **YoY(전년 동기 대비)** 기준
- 시즌은 **도시별 주문건수** 기준으로 독립 분류 (월평균 ± 0.5σ)
- 분석 기간: 최근 24개월 (오늘 기준 2년)

### 유입 경로 분석 (추가 차원)

퍼널 ①②에 **유입 채널(UTM source)** 차원을 추가하여 채널별 유입량과 전환 품질을 분석한다.

---

## 실행 절차

### Phase 0: 사전 준비

1. **국가명 매핑**: 한글 입력이면 위 매핑 테이블로 `COUNTRY_NM` 확정
2. **오늘 날짜 확인**: `date -u -v+9H +%Y-%m-%d` (KST 기준). 배치 데이터에서 오늘 제외.
3. **분석 기간 설정**: `ANALYSIS_END = 오늘 - 1일의 월 마지막 날`, `ANALYSIS_START = ANALYSIS_END - 24개월`
4. **데이터 존재 확인**: 해당 국가에 T&A 데이터가 충분한지 빠르게 확인

```sql
SELECT COUNT(DISTINCT ORDER_ID) AS total_orders, MIN(BASIS_DATE) AS min_date, MAX(BASIS_DATE) AS max_date
FROM mrtdata.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE COUNTRY_NM = '{COUNTRY}' AND FPNA_DOMAIN_NM = 'TNA'
  AND BASIS_DATE >= DATE_SUB(CURRENT_DATE(), INTERVAL 24 MONTH)
```

5. **Top 5 도시 식별**: GMV 기준 상위 5개 도시 자동 선정

```sql
SELECT CITY_NM, SUM(GMV) AS total_gmv, COUNT(DISTINCT ORDER_ID) AS total_orders
FROM mrtdata.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE COUNTRY_NM = '{COUNTRY}' AND FPNA_DOMAIN_NM = 'TNA'
  AND BASIS_DATE >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY 1 ORDER BY 2 DESC LIMIT 5
```

---

### Phase 1: 현황 진단 (L1 도시 레벨)

**3개 분석을 병렬 실행한다.**

#### 1-1. 시즌 패턴 정의 (도시별)

각 도시별로 독립적으로:
1. 같은 월끼리 2개년 평균 주문건수 산출
2. 해당 도시의 12개월 평균(μ)과 표준편차(σ) 계산
3. 분류: 성수기(μ+0.5σ 이상), 비수기(μ-0.5σ 이하), 준성수기(나머지)

```sql
SELECT CITY_NM, FORMAT_DATE('%Y-%m', BASIS_DATE) AS month,
  COUNT(DISTINCT ORDER_ID) AS order_cnt
FROM mrtdata.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE COUNTRY_NM = '{COUNTRY}' AND FPNA_DOMAIN_NM = 'TNA'
  AND CITY_NM IN ({TOP_5_CITIES})
  AND BASIS_DATE BETWEEN '{START}' AND '{END}'
GROUP BY 1, 2 ORDER BY 1, 2
```

#### 1-2. 퍼널 6단계 도시별 YoY

**주문/GMV/CFR/마진율** (③~⑥):
```sql
SELECT CITY_NM,
  CASE WHEN BASIS_DATE BETWEEN '{CURRENT_START}' AND '{CURRENT_END}' THEN 'current'
       WHEN BASIS_DATE BETWEEN '{PRIOR_START}' AND '{PRIOR_END}' THEN 'prior' END AS period,
  COUNT(DISTINCT ORDER_ID) AS order_cnt,
  SAFE_DIVIDE(SUM(GMV), COUNT(DISTINCT ORDER_ID)) AS aov,
  SAFE_DIVIDE(SUM(CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN GMV ELSE 0 END), SUM(GMV)) AS cfr,
  SAFE_DIVIDE(SUM(CON_MARGIN), SUM(CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN GMV ELSE 0 END)) AS margin_rate,
  SUM(GMV) AS total_gmv, SUM(CON_MARGIN) AS con_margin, SUM(CPD_AMOUNT) AS cpd
FROM mrtdata.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE COUNTRY_NM = '{COUNTRY}' AND FPNA_DOMAIN_NM = 'TNA'
  AND CITY_NM IN ({TOP_5_CITIES})
  AND BASIS_DATE BETWEEN '{PRIOR_START}' AND '{CURRENT_END}'
GROUP BY 1, 2 ORDER BY total_gmv DESC
```

**UV/CVR** (①②):
```sql
SELECT p.CITY_NM,
  CASE WHEN c.BASIS_DATE BETWEEN '{CURRENT_START}' AND '{CURRENT_END}' THEN 'current'
       WHEN c.BASIS_DATE BETWEEN '{PRIOR_START}' AND '{PRIOR_END}' THEN 'prior' END AS period,
  COUNT(DISTINCT c.PID) AS uv_pid,
  SUM(c.CHECKOUT_COMPLETE_FLAG) AS purchases
FROM mrtdata.edw_mart.MART_BIZ_LOG_PID_CONVERSION_D c
JOIN mrtdata.edw_mart.MART_PRODUCT_D p ON c.ITEM_ID = p.GID
WHERE c.OFFER_DETAIL_FLAG = 1
  AND p.COUNTRY_NM = '{COUNTRY}'
  AND p.STANDARD_CATEGORY_LV_1_CD IN ('TOUR','TICKET','ACTIVITY','CLASS','CONVENIENCE','SNAP')
  AND p.CITY_NM IN ({TOP_5_CITIES})
  AND c.BASIS_DATE BETWEEN '{PRIOR_START}' AND '{CURRENT_END}'
GROUP BY 1, 2 ORDER BY uv_pid DESC
```

> **주의**: BIZ_LOG는 대용량. 24개월 풀스캔이 너무 크면 12개월씩 나눠서 실행.

#### 1-3. 전체 YoY 성적표

국가 전체 기준 주문건수, GMV, CGMV, 공헌이익, CPD, 채널수수료 YoY 비교.

#### Phase 1 산출물

- 도시별 시즌 비교표 (월 × 도시 매트릭스)
- 퍼널 6단계 도시별 YoY 종합 표
- 도시별 한 줄 진단 (UV↑/CVR↓ 등)
- 문제 신호 목록 (심각도 + 도시 + 상세)

---

### Phase 2: 문제별 드릴다운 (L2 카테고리 + L3 상품)

Phase 1에서 식별된 문제 신호를 기반으로, **문제가 있는 도시만** L2/L3으로 드릴다운한다.

#### 드릴다운 대상 (Phase 1 결과에 따라 동적 결정)

| 신호 | 드릴다운 |
|------|---------|
| CVR 하락 도시 | 해당 도시의 카테고리별 UV/CVR + UTM별 CVR |
| 주문 역성장 도시 | 카테고리별 주문건수 YoY + UV vs CVR 분해 |
| CFR 하락 도시 | 카테고리별 CFR + 성수기 vs 비수기 + 취소 TOP 상품(L3) |
| CPD 급증 | 도시 × 카테고리별 CPD YoY + 월별 추이 |

#### 핵심 쿼리 템플릿

**카테고리별 주문/CFR/CPD** (FP&A):
```sql
SELECT CITY_NM, CATEGORY_NM,
  CASE WHEN BASIS_DATE BETWEEN '{CURRENT_START}' AND '{CURRENT_END}' THEN 'current'
       WHEN BASIS_DATE BETWEEN '{PRIOR_START}' AND '{PRIOR_END}' THEN 'prior' END AS period,
  COUNT(DISTINCT ORDER_ID) AS order_cnt, SUM(GMV) AS gmv,
  SUM(CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN GMV ELSE 0 END) AS cgmv,
  SAFE_DIVIDE(SUM(CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN GMV ELSE 0 END), SUM(GMV)) AS cfr,
  SUM(CON_MARGIN) AS con_margin, SUM(CPD_AMOUNT) AS cpd
FROM mrtdata.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE COUNTRY_NM = '{COUNTRY}' AND FPNA_DOMAIN_NM = 'TNA'
  AND CITY_NM = '{TARGET_CITY}'
  AND BASIS_DATE BETWEEN '{PRIOR_START}' AND '{CURRENT_END}'
GROUP BY 1, 2, 3 ORDER BY gmv DESC
```

**UTM별 UV/CVR** (BIZ_LOG):
```sql
SELECT IFNULL(c.UTM_SOURCE, '(direct/none)') AS utm_source,
  CASE WHEN c.BASIS_DATE BETWEEN '{RECENT_CURRENT}' AND '{CURRENT_END}' THEN 'current'
       WHEN c.BASIS_DATE BETWEEN '{RECENT_PRIOR}' AND '{PRIOR_END_6M}' THEN 'prior' END AS period,
  COUNT(DISTINCT c.PID) AS uv, SUM(c.CHECKOUT_COMPLETE_FLAG) AS purchases,
  ROUND(SAFE_DIVIDE(SUM(c.CHECKOUT_COMPLETE_FLAG), COUNT(DISTINCT c.PID)) * 100, 2) AS cvr_pct
FROM mrtdata.edw_mart.MART_BIZ_LOG_PID_CONVERSION_D c
JOIN mrtdata.edw_mart.MART_PRODUCT_D p ON c.ITEM_ID = p.GID
WHERE c.OFFER_DETAIL_FLAG = 1 AND p.COUNTRY_NM = '{COUNTRY}' AND p.CITY_NM = '{TARGET_CITY}'
  AND p.STANDARD_CATEGORY_LV_1_CD IN ('TOUR','TICKET','ACTIVITY','CLASS','CONVENIENCE','SNAP')
  AND c.BASIS_DATE BETWEEN '{RECENT_PRIOR}' AND '{CURRENT_END}'
GROUP BY 1, 2 HAVING uv >= 100 ORDER BY uv DESC
```

**취소 TOP 상품 (L3)**:
```sql
SELECT CITY_NM, CATEGORY_NM, GID, PRODUCT_TITLE,
  COUNT(DISTINCT ORDER_ID) AS order_cnt, SUM(GMV) AS gmv,
  SAFE_DIVIDE(SUM(CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN GMV ELSE 0 END), SUM(GMV)) AS cfr,
  SUM(GMV) - SUM(CASE WHEN RECENT_STATUS IN ('confirm','finish') THEN GMV ELSE 0 END) AS cancelled_gmv
FROM mrtdata.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE COUNTRY_NM = '{COUNTRY}' AND FPNA_DOMAIN_NM = 'TNA' AND CITY_NM = '{TARGET_CITY}'
  AND BASIS_DATE BETWEEN '{PEAK_START}' AND '{PEAK_END}'
GROUP BY 1, 2, 3, 4 HAVING order_cnt >= 5 ORDER BY cancelled_gmv DESC LIMIT 20
```

#### Phase 2 산출물

- 카테고리별 퍼널 지표 YoY 비교표
- UTM별 UV/CVR 변화표
- 취소 TOP 상품 목록 (L3)
- 문제 원인 특정 ("X 도시 CVR 하락의 Y%는 Z 카테고리의 W 채널에서 발생")

---

### Phase 3: 심층 분석

4개 심층 분석을 **병렬로** 실행한다.

#### 3-1. 광고비 × 유입 효율

> **중요: `mkt_dashboard_raw_data`의 `country_en` 값과 FP&A 테이블의 `COUNTRY_NM` 값이 다를 수 있다.** 반드시 아래 순서를 따른다.

**Step 1: 해당 국가의 `country_en` 값 확인**
```sql
SELECT DISTINCT country_en FROM mrtdata.business.mkt_dashboard_raw_data
WHERE biz_type = 'T&A' AND LOWER(country_en) LIKE LOWER('%{COUNTRY}%') LIMIT 5
```

**Step 2: country_en으로 결과가 없으면, city_en으로 fallback**
```sql
-- Phase 0에서 식별한 Top 5 도시명으로 검색
SELECT DISTINCT city_en FROM mrtdata.business.mkt_dashboard_raw_data
WHERE biz_type = 'T&A' AND (LOWER(city_en) LIKE '%{CITY_1}%' OR LOWER(city_en) LIKE '%{CITY_2}%' ...) LIMIT 20
```

**Step 3: 확인된 필터로 도시 × 채널별 광고비/GMV/ROAS 추출**

> **광고비 분석은 반드시 도시 레벨로 수행한다.** 국가 전체 합산은 도시별 차이를 가린다.

```sql
SELECT city_en, channel,
  CASE WHEN date BETWEEN '{Q1_PRIOR_START}' AND '{Q1_PRIOR_END}' THEN 'prior'
       WHEN date BETWEEN '{Q1_CURRENT_START}' AND '{Q1_CURRENT_END}' THEN 'current' END AS period,
  SUM(cost) AS cost, SUM(gmv) AS gmv, SUM(con_margin) AS con_margin,
  SAFE_DIVIDE(SUM(gmv), NULLIF(SUM(cost), 0)) AS roas
FROM mrtdata.business.mkt_dashboard_raw_data
WHERE biz_type = 'T&A'
  AND (country_en = '{CONFIRMED_COUNTRY_EN}' OR city_en IN ({CONFIRMED_CITY_ENS}))
  AND date BETWEEN '{Q1_PRIOR_START}' AND '{Q1_CURRENT_END}'
GROUP BY 1, 2, 3 HAVING SUM(cost) > 0 ORDER BY city_en, cost DESC
```

**Fallback 순서 요약**:
1. `country_en` 필터로 시도 (도시별 결과가 `city_en`에 포함됨)
2. country_en 결과 없으면 → `city_en` IN (Top 5 도시) 필터로 재시도
3. 둘 다 없으면 → "해당 국가의 광고비 데이터가 `mkt_dashboard_raw_data`에 없습니다. 마케팅팀에 데이터 적재를 요청하세요."로 안내. T&A 전체 기준 ROAS를 참고 수치로 제공.

> **주의**: `city_en` 값의 대소문자가 불규칙할 수 있다 (예: `lisbon` vs `Porto`). `LOWER()` 비교로 매칭하되, 결과 표시는 원본 값을 사용한다.

채널별 진단 프레임 적용:
```
             광고비 ↓              광고비 유지/↑
            ──────────           ─────────────────
UV ↓      → A. 투자 부족           B. 효율 악화
UV 유지    →  (해당 없음)          C. 전환 품질 문제
```

#### 3-2. 체크아웃 퍼널 이탈

```sql
-- 상세→체크아웃→결제 각 단계별 전환율
SELECT
  CASE WHEN c.BASIS_DATE BETWEEN '{CURRENT_START}' AND '{CURRENT_END}' THEN 'current'
       WHEN c.BASIS_DATE BETWEEN '{PRIOR_START}' AND '{PRIOR_END}' THEN 'prior' END AS period,
  COUNT(DISTINCT c.PID) AS detail_uv,
  SUM(c.CHECKOUT_FLAG) AS checkout_cnt,
  SUM(c.CHECKOUT_COMPLETE_FLAG) AS purchase_cnt,
  ROUND(SAFE_DIVIDE(SUM(c.CHECKOUT_FLAG), COUNT(DISTINCT c.PID)) * 100, 2) AS detail_to_checkout_pct,
  ROUND(SAFE_DIVIDE(SUM(c.CHECKOUT_COMPLETE_FLAG), NULLIF(SUM(c.CHECKOUT_FLAG), 0)) * 100, 2) AS checkout_to_purchase_pct
FROM mrtdata.edw_mart.MART_BIZ_LOG_PID_CONVERSION_D c
JOIN mrtdata.edw_mart.MART_PRODUCT_D p ON c.ITEM_ID = p.GID
WHERE c.OFFER_DETAIL_FLAG = 1 AND p.COUNTRY_NM = '{COUNTRY}' AND p.CITY_NM = '{TARGET_CITY}'
  AND p.STANDARD_CATEGORY_LV_1_CD IN ('TOUR','TICKET','ACTIVITY','CLASS','CONVENIENCE','SNAP')
  AND c.BASIS_DATE BETWEEN '{PRIOR_START}' AND '{CURRENT_END}'
GROUP BY 1 ORDER BY 1
```

"상세→체크아웃"이 문제인지 "체크아웃→결제"가 문제인지 특정.

#### 3-3. 취소 사유·시점 패턴

```sql
-- 예약→취소 간격 분포
SELECT CITY_NM,
  CASE WHEN RESVE_CANCEL_DAY_DIFF <= 0 THEN 'same_day'
       WHEN RESVE_CANCEL_DAY_DIFF <= 1 THEN '1d'
       WHEN RESVE_CANCEL_DAY_DIFF <= 3 THEN '2-3d'
       WHEN RESVE_CANCEL_DAY_DIFF <= 7 THEN '4-7d'
       WHEN RESVE_CANCEL_DAY_DIFF <= 14 THEN '8-14d'
       WHEN RESVE_CANCEL_DAY_DIFF <= 30 THEN '15-30d'
       ELSE '30d+' END AS cancel_timing,
  COUNT(DISTINCT ORDER_ID) AS order_cnt
FROM mrtdata.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE COUNTRY_NM = '{COUNTRY}' AND FPNA_DOMAIN_NM = 'TNA'
  AND CITY_NM IN ({CFR_PROBLEM_CITIES}) AND RECENT_STATUS = 'cancel'
  AND BASIS_DATE BETWEEN '{PEAK_START}' AND '{PEAK_END}'
GROUP BY 1, 2 ORDER BY 1, order_cnt DESC
```

취소 3가지 패턴 분류: (1) 즉시 취소, (2) 사전 취소, (3) 상품/파트너 문제.

#### 3-4. 쿠폰(CPD) ROI

```sql
-- 시즌 효과 제거를 위한 동기 비교 (H2 prior vs H2 current)
SELECT CITY_NM,
  CASE WHEN BASIS_DATE BETWEEN '{H2_PRIOR_START}' AND '{H2_PRIOR_END}' THEN 'prior_h2'
       WHEN BASIS_DATE BETWEEN '{H2_CURRENT_START}' AND '{H2_CURRENT_END}' THEN 'current_h2' END AS period,
  COUNT(DISTINCT ORDER_ID) AS order_cnt, SUM(GMV) AS gmv,
  SUM(CON_MARGIN) AS con_margin, SUM(CPD_AMOUNT) AS cpd
FROM mrtdata.edw_fpna.MART_FPNA_NONAIR_PROFIT_D
WHERE COUNTRY_NM = '{COUNTRY}' AND FPNA_DOMAIN_NM = 'TNA'
  AND CITY_NM IN ({TOP_5_CITIES})
  AND BASIS_DATE BETWEEN '{H2_PRIOR_START}' AND '{H2_CURRENT_END}'
GROUP BY 1, 2 ORDER BY 1, 2
```

CPD 1원당 증분 GMV/CM 산출. 도시별 효율 비교.

#### Phase 3 산출물

- 채널별 광고비/ROAS YoY + 진단(A/B/C)
- 퍼널 단계별 이탈 포인트 특정
- 취소 타이밍 패턴 + TOP 상품 취소 패턴
- 쿠폰 한계효과 (도시별)
- 마케팅팀 제안 액션 아이템

---

### 보고서 자동 발행 (Confluence)

모든 Phase가 완료되면 사용자의 Confluence 개인 스페이스에 종합 보고서를 자동 발행한다.

**제목 형식**: `[분석] {국가명} T&A 공헌이익 종합 분석 ({YYYY.MM})`

**보고서 구조**:
1. **한눈에 보는 결과** — 문제 4가지의 원인/수치/방향 1줄 요약표
2. **Phase 1 현황 진단** — 전체 YoY, 시즌 패턴, 퍼널 6단계 도시별 진단
3. **Phase 2 드릴다운** — 문제별 카테고리/상품 분해
4. **Phase 3 심층 분석** — 광고비 효율, 퍼널 이탈, 취소 패턴, 쿠폰 ROI
5. **종합 문제 구조도** — 전체 퍼널을 하나의 트리로 연결
6. **Next Step** — 전략 수립 방향

각 섹션 앞에 **"왜 이 분석이 필요한가?"**를 설명하고, 전문용어에는 괄호로 설명을 붙인다.

---

## 공통 규칙

- BQ wrapper: `./.claude/hooks/run-bq-readonly.sh bq query --use_legacy_sql=false --location=asia-northeast3 --format=prettyjson --max_rows=200 "쿼리"`
- 금액 단위: 1억 이상은 `억` 단위(소수 2자리), 1만 이상은 `만` 단위
- 비율: 소수 1자리 %
- 증감 표시: ▲(증가), ▼(감소)
- 모든 비교는 YoY 기준. 시즌 효과를 제거하기 위해 동일 시즌끼리 비교.
- 배치 데이터에서 오늘 날짜 제외. 당일 데이터는 미적재/불완전.
- BIZ_LOG 대용량 주의: 필요시 기간 분할 실행.
- 확정 건 기준: `RECENT_STATUS IN ('confirm', 'finish')`. `finish`는 확정 이후 lifecycle이므로 반드시 포함.
- `analyst` subagent(opus)에 위임하여 병렬 실행. main agent는 오케스트레이션 + 검수 + 최종 응답.
- 사용자는 사업개발/전략팀. 마케팅팀은 별도 조직. 광고비/유입 관련 산출물은 "마케팅팀에 제안할 수 있는 형태"로 정리.

## NOT for

- 항공, 숙박, 보험 등 T&A 외 도메인 분석
- dbt 모델 개발/수정
- 실시간 모니터링 (배치 데이터 기반 분석)
