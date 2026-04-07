# Date Histogram 집계 — 시간 버킷·타임존·캘린더 인식 간격

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 에포크 타임스탬프를 기반으로 시간 버킷 경계를 계산하는 방법은?
- `fixed_interval`과 `calendar_interval`의 근본적인 차이는 무엇인가?
- 타임존 설정이 버킷 경계에 미치는 영향은 무엇인가?
- 서머타임(DST) 전환 시 버킷이 어떻게 처리되는가?
- 월별, 분기별 집계에서 `calendar_interval`이 반드시 필요한 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

시계열 데이터 분석에서 "일별 주문 수"나 "월별 매출"은 가장 흔한 요구사항이다. 타임존을 잘못 설정하면 한국 사용자의 "1월 1일" 데이터가 UTC 기준으로 "12월 31일"과 "1월 1일"에 걸쳐 분산된다. `calendar_interval`을 `fixed_interval`로 설정하면 2월이 28~29일이라는 사실을 무시해 버킷 경계가 어긋난다. 이런 오류는 데이터를 직접 세어보지 않으면 찾기 어렵다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 타임존 설정 없이 UTC로 집계
  요청:
    { "date_histogram": {
      "field": "created_at",
      "calendar_interval": "day"
    }}

  문제 (한국 서비스):
    한국 시간 2024-01-15 00:00 KST
    = 2024-01-14 15:00 UTC

    UTC 기준 "2024-01-14" 버킷에 포함
    → 한국 사용자 입장에서 "1월 15일" 데이터가 "1월 14일" 버킷에 잡힘

  올바른 설정:
    "time_zone": "Asia/Seoul"
    → 버킷 경계를 한국 시간 00:00으로 설정

실수 2: 월별 집계에 fixed_interval 사용
  설정:
    "fixed_interval": "30d"  // 30일 고정

  문제:
    1월: 31일 → 30일 고정 → 1/31 데이터가 다음 버킷에
    2월: 28일 → 30일 고정 → 2/29 ~ 3/1 데이터가 같은 버킷에
    → "월별" 버킷이 아닌 "30일 단위" 버킷

  올바른 설정:
    "calendar_interval": "month"
    → 캘린더 기준 달 경계 인식 (28/29/30/31일 자동 처리)

실수 3: 빈 버킷이 없어야 하는데 나타나는 경우
  상황:
    "일별 주문" 집계 → 주문 없는 날도 버킷으로 나타남

  기본 동작:
    ES는 기본적으로 빈 버킷도 생성
    → 차트에서 불연속 데이터처럼 보이지 않음 (실제로는 0)

  min_doc_count 설정:
    "min_doc_count": 1 → 문서 없는 버킷 제외
    "min_doc_count": 0 → 빈 버킷도 포함 (기본값: 조건에 따라 다름)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Date Histogram 설계 원칙:

  항상 time_zone 명시:
    "time_zone": "Asia/Seoul"      // IANA 타임존 ID 사용
    "time_zone": "+09:00"          // 고정 오프셋 (DST 미적용)
    → 애플리케이션 서버와 동일한 타임존

  캘린더 인식 간격 (calendar_interval):
    day, week, month, quarter, year
    → 캘린더 경계 인식 (월별 일수 차이, 윤년 등)
    → 달력 기반 집계에 필수

  고정 간격 (fixed_interval):
    "1d", "1h", "30m", "1ms" (밀리초 단위)
    → 항상 동일한 밀리초 간격
    → 정확한 시간 단위 분석에 사용 (로그 분석, 실시간 모니터링)

  extended_bounds로 날짜 범위 강제:
    "extended_bounds": {
      "min": "2024-01-01", "max": "2024-12-31"
    }
    → 문서 없는 날도 버킷 생성 (차트 연속성 유지)
```

---

## 🔬 내부 동작 원리

### 1. 버킷 경계 계산 — 에포크 기반

```
Date Histogram 버킷 경계 계산 방식:

  date 필드 내부 저장:
    "2024-01-15T09:30:00+09:00"
    → epoch milliseconds: 1705279800000

  UTC 기준 일별 버킷 경계 계산:
    bucket_key = floor(epoch_ms / interval_ms) × interval_ms

    1일 = 86,400,000 ms
    epoch = 1705279800000

    bucket_key = floor(1705279800000 / 86400000) × 86400000
               = floor(19736.11...) × 86400000
               = 19736 × 86400000
               = 1705305600000  ← 2024-01-15T00:00:00 UTC
               = 2024-01-15T09:00:00 KST

    → UTC 기준으로 "2024-01-15" 버킷에 배치

  타임존 적용 시 (Asia/Seoul, UTC+9):
    실제 버킷 경계:
      2024-01-15T00:00:00 KST = 2024-01-14T15:00:00 UTC
      → epoch_ms = 1705237200000

    타임존 적용 후 버킷 경계:
      2024-01-14T15:00:00 UTC ~ 2024-01-15T14:59:59.999 UTC
      → KST로 2024-01-15 00:00:00 ~ 23:59:59

    2024-01-15T09:30:00 KST → UTC 기준으로 위 범위 내 → "2024-01-15" 버킷 ✅

  타임존 없이 (UTC 기준):
    2024-01-15T09:30:00 KST = 2024-01-15T00:30:00 UTC
    → UTC 기준 "2024-01-15" 버킷에 배치 ✅ (우연히 같은 날)

    하지만 2024-01-15T00:30:00 KST = 2024-01-14T15:30:00 UTC
    → UTC 기준 "2024-01-14" 버킷에 배치 ❌ (KST로는 1월 15일인데!)
```

### 2. calendar_interval vs fixed_interval

```
fixed_interval (고정 밀리초):
  "1d" = 86,400,000 ms 고정
  "1h" = 3,600,000 ms 고정
  "30m" = 1,800,000 ms 고정

  특징:
    항상 동일한 간격 → 수학적으로 일관
    달력 의미 없음 → 2월 28일 다음 버킷이 3월 29일이 될 수 있음
    서머타임 영향 없음 (밀리초 기반)

  사용 사례:
    로그 분석 (매 10분마다 에러 수)
    실시간 모니터링 (매 1시간 TPS)
    정확한 시간 간격 측정

calendar_interval (캘린더 인식):
  day:     달력의 하루 (서머타임 → 23h 또는 25h 가능)
  week:    달력의 일주일 (7일)
  month:   달력의 한 달 (28/29/30/31일)
  quarter: 달력의 분기 (3개월)
  year:    달력의 1년 (365/366일)

  특징:
    달력 경계를 인식 → 1월/2월/3월 각각의 일수 자동 처리
    서머타임 인식 → 전환 날에 버킷 크기가 달라질 수 있음
    인간 친화적 시간 단위

  사용 사례:
    월별 매출 분석
    분기별 실적 비교
    연도별 성장률 계산

비교 예시:
  2024-02 데이터 집계:

  fixed_interval "30d":
    2024-01-01 + 30d = 2024-01-31
    2024-01-31 + 30d = 2024-03-01
    → 2월 전체가 하나의 버킷에 들어가지 않음

  calendar_interval "month":
    2024-01-01 ~ 2024-01-31: 1월 버킷
    2024-02-01 ~ 2024-02-29: 2월 버킷 (2024년은 윤년)
    2024-03-01 ~ 2024-03-31: 3월 버킷
    → 달력 기준으로 정확히 월별 집계
```

### 3. 서머타임(DST) 처리

```
서머타임 예시: 미국 동부 시간 (America/New_York)
  2024-03-10: 서머타임 시작 (2시 → 3시로 1시간 전진)
              이 날은 23시간!

  calendar_interval "day" + time_zone: "America/New_York":
    2024-03-09 버킷: 24시간 (86,400,000 ms)
    2024-03-10 버킷: 23시간 (82,800,000 ms) ← 서머타임으로 1시간 짧음
    2024-03-11 버킷: 24시간

  calendar_interval이 서머타임을 인식:
    "day"는 캘린더의 하루 → 그 날이 23시간이면 23시간 버킷
    
  fixed_interval "1d"는 항상 86,400,000 ms:
    서머타임 전환일도 24시간 버킷
    → 일부 시간이 잘못된 버킷에 배치될 수 있음

  한국 (Asia/Seoul): 서머타임 없음
    → 한국 서비스는 DST 문제 없음
    → 고정 오프셋 "+09:00" 또는 "Asia/Seoul" 모두 동일 결과

  DST 있는 타임존의 권장:
    IANA ID 사용: "America/New_York" (DST 자동 처리)
    고정 오프셋 피하기: "+05:00"는 DST 전환 인식 못함
```

### 4. extended_bounds와 hard_bounds

```
extended_bounds (버킷 범위 확장):
  쿼리 결과에 문서가 없는 날도 버킷 생성

  용도: 차트 연속성 유지
        "1월 5일 주문 없음" → 버킷 없이 4일에서 6일로 점프 (불연속)
        → extended_bounds로 5일 버킷(0개) 강제 생성

  { "date_histogram": {
    "field": "created_at",
    "calendar_interval": "day",
    "time_zone": "Asia/Seoul",
    "extended_bounds": {
      "min": "2024-01-01",
      "max": "2024-01-31"
    },
    "min_doc_count": 0     // 빈 버킷 포함 (필수!)
  }}

hard_bounds (버킷 범위 제한):
  특정 날짜 범위 밖의 버킷 생성 방지

  { "date_histogram": {
    "field": "created_at",
    "calendar_interval": "month",
    "hard_bounds": {
      "min": "2024-01-01",
      "max": "2024-12-31"
    }
  }}
  → 2023년 이전, 2025년 이후 데이터가 있어도 버킷 미생성

  extended_bounds vs hard_bounds:
    extended: 범위를 확장 (최소~최대 사이 빈 버킷 추가)
    hard:     범위를 제한 (경계 밖 버킷 제거)
    함께 사용 가능: 특정 기간의 일별 버킷을 모두 생성

min_doc_count:
  기본값 1 (쿼리 범위가 없으면) 또는 0 (extended_bounds 있으면)
  0: 문서 없는 버킷도 반환
  1: 문서 있는 버킷만 반환
```

---

## 💻 실전 실험

```bash
# 기본 날짜별 집계
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "daily_orders": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "day",
        "time_zone": "Asia/Seoul",
        "format": "yyyy-MM-dd"
      },
      "aggs": {
        "daily_revenue": { "sum": { "field": "amount" } }
      }
    }
  }
}

# calendar_interval vs fixed_interval 비교
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "calendar_month": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month",
        "time_zone": "Asia/Seoul",
        "format": "yyyy-MM"
      }
    },
    "fixed_30d": {
      "date_histogram": {
        "field": "created_at",
        "fixed_interval": "30d",
        "time_zone": "Asia/Seoul",
        "format": "yyyy-MM-dd"
      }
    }
  }
}
# 두 집계 버킷 경계 차이 확인 (특히 2월)

# extended_bounds로 빈 날짜 포함
GET /orders/_search
{
  "size": 0,
  "query": {
    "range": {
      "created_at": { "gte": "2024-01-01", "lte": "2024-01-31" }
    }
  },
  "aggs": {
    "daily_with_empty": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "day",
        "time_zone": "Asia/Seoul",
        "format": "yyyy-MM-dd",
        "min_doc_count": 0,
        "extended_bounds": {
          "min": "2024-01-01",
          "max": "2024-01-31"
        }
      }
    }
  }
}
# 주문 없는 날도 버킷으로 나타남 (doc_count: 0)

# 중첩 집계: 월별 + 카테고리별
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "monthly": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month",
        "time_zone": "Asia/Seoul",
        "format": "yyyy-MM"
      },
      "aggs": {
        "by_category": {
          "terms": { "field": "category" },
          "aggs": {
            "revenue": { "sum": { "field": "amount" } }
          }
        }
      }
    }
  }
}

# 타임존 차이 실험 (UTC vs Asia/Seoul)
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "utc_daily": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "day",
        "format": "yyyy-MM-dd HH:mm:ss"
      }
    },
    "seoul_daily": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "day",
        "time_zone": "Asia/Seoul",
        "format": "yyyy-MM-dd HH:mm:ss"
      }
    }
  }
}
# key_as_string 값 비교: UTC는 15:00으로 시작, Seoul은 00:00으로 시작
```

---

## 📊 calendar_interval vs fixed_interval 비교

| 항목 | `calendar_interval` | `fixed_interval` |
|------|-------------------|----------------|
| 기준 | 달력 경계 (가변 일수) | 고정 밀리초 |
| 월별 지원 | month, quarter, year | 지원 불가 (30d ≠ 1개월) |
| 서머타임 처리 | 인식 (버킷 크기 가변) | 인식 못함 (항상 고정) |
| 사용 사례 | 달력 기반 비즈니스 분석 | 정확한 시간 간격 분석 |
| 2월 처리 | 28/29일 자동 | 30d → 3월 포함 |

---

## ⚖️ 트레이드오프

```
calendar_interval "month":
  이득: 달력 의미 보존, 월별 분석의 표준
  비용: 서버에서 각 달의 일수를 계산해야 함 (약간의 추가 비용)

fixed_interval "30d":
  이득: 고정 간격 → 버킷 크기 일정
  비용: "월별" 의미와 맞지 않음 → 비즈니스 오해 가능

time_zone 설정:
  이득: 사용자 관점의 정확한 시간 버킷
  비용: 타임존 변환 연산 추가

extended_bounds:
  이득: 차트 연속성, 빈 기간 가시화
  비용: 빈 버킷도 메모리에 생성 → 긴 기간의 day 단위는 많은 버킷

min_doc_count:
  0: 완전한 날짜 범위 표현 (차트에 적합)
  1: 데이터 있는 날짜만 (집계/분석에 적합)
```

---

## 📌 핵심 정리

```
버킷 경계 계산:
  epoch_ms 기반 → time_zone 적용 → 달력 경계 계산
  타임존 없으면 UTC 기준 → 한국 서비스에서 날짜 어긋남 발생

calendar_interval:
  달력 경계 인식 (28/29/30/31일, 서머타임)
  월/분기/연도 집계에 필수
  "month", "quarter", "year", "week", "day"

fixed_interval:
  고정 밀리초 간격 (서머타임 인식 못함)
  정확한 시간 단위 분석에 적합
  "1d", "1h", "30m", "1ms"

타임존 설정:
  항상 IANA ID로 명시 ("Asia/Seoul", "America/New_York")
  고정 오프셋("+09:00")은 DST 없는 타임존에서만 안전

extended_bounds + min_doc_count: 0:
  빈 날짜 버킷 강제 생성
  차트 연속성 유지에 필수
```

---

## 🤔 생각해볼 문제

**Q1.** "매주 월요일 시작" 기준으로 주별 집계를 하려면 어떻게 설정하는가?

<details>
<summary>해설 보기</summary>

`calendar_interval: "week"`의 기본 시작일은 월요일(ISO 8601 기준)이다. 타임존만 올바르게 설정하면 월요일 00:00을 기준으로 버킷이 생성된다. 일요일 시작을 원한다면 `offset` 파라미터를 사용해 버킷 경계를 이동할 수 있다. `"offset": "-1d"`를 사용하면 일요일 시작으로 조정된다. 또는 `calendar_interval: "week"`에 `"week_of_year_policy": "SUN_START"` 같은 설정은 없으므로, 일요일 시작이 필요하면 날짜를 애플리케이션 레벨에서 변환하거나 `offset`을 활용해야 한다.

</details>

**Q2.** `extended_bounds` 없이 특정 날짜 범위 쿼리와 `date_histogram`을 함께 사용하면 빈 날짜가 나타나는가?

<details>
<summary>해설 보기</summary>

기본적으로 데이터가 없는 날짜 버킷은 생성되지 않는다. `min_doc_count`의 기본값은 1이므로 문서가 없는 버킷은 제외된다. 하지만 `min_doc_count: 0`으로 설정하면 데이터 있는 첫 날과 마지막 날 사이의 빈 버킷은 생성된다. 그러나 쿼리 범위의 시작일이나 종료일에 데이터가 없으면 그 경계 버킷은 생성되지 않는다. `extended_bounds`는 이 한계를 넘어 쿼리 결과와 무관하게 지정된 날짜 범위 전체의 버킷을 강제로 생성한다.

</details>

**Q3.** "분기별 매출"을 집계할 때 `calendar_interval: "quarter"`와 `fixed_interval: "90d"`의 차이가 실제로 나타나는 구체적인 상황은?

<details>
<summary>해설 보기</summary>

2024년 1분기(Q1)는 1월 1일부터 3월 31일까지 91일이다. `fixed_interval: "90d"`를 사용하면 1월 1일 + 90일 = 3월 31일이 버킷 경계가 되므로 3월 31일 하루가 다음 버킷(Q2)에 포함된다. 또한 2024년 Q2는 4월 1일부터 6월 30일까지 91일인데, 90일 버킷은 4월 1일 + 90일 = 6월 29일에 버킷 경계가 생겨 6월 29~30일이 Q3 버킷에 섞인다. 반면 `calendar_interval: "quarter"`는 3월 31일, 6월 30일, 9월 30일, 12월 31일을 정확한 경계로 인식해 비즈니스 의미의 분기와 정확히 일치한다.

</details>

---

<div align="center">

**[⬅️ 이전: Terms Aggregation 내부](./02-terms-aggregation.md)** | **[홈으로 🏠](../README.md)** | **[다음: 집계와 메모리 ➡️](./04-aggregation-memory.md)**

</div>
