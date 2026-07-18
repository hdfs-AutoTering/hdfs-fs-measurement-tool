# Measurement 결과 Schema v1 

> hdfs-fs-measurement tool이 출력하는 결과 CSV 파일의 스키마를 설명한다. 이 스키마는 **측정 도구(Java)와 분석 레이어(Python) 사이의 API**로 역할한다. 
>
> **상태:** 확정 (v1)
> **최종 수정:** 2026-07-18
>
---
## 1. 설계 원칙 

1. **측정 실험만 수행**
   평균·중앙값·오차 막대 등 데이터 가공 및 통계는 전부 분석 레이어(Python)에서 수행된다. 측정 도구는 통계를 내지 않는다.

2. **스키마 확장, 실험 시도 횟수 축소**
   RQ2, RQ3 확장 실험에 필요한 칼럼을 모두 포함하여 스키마를 구성하여, 실험 비용을 절약한다. 

## 2. 스키마 정의 

컬럼은 4개 그룹으로 구성된다: **식별자 → 실험 조건 → 측정 → 출처 → 유효성**

### 그룹 1: 식별자 (identifier) 

| 컬럼 | 타입 | 의미 및 컴포넌트 |
|---|---|---|
| `policy_id` | string | 측정 대상 티어링 정책. B0/B1/B2(베이스라인), P1/P2(제안 정책) |
| `workload_id` | string | 클라이언트 Spark 워크로드 식별자. 워크로드 정의(입력 크기, 잡 종류) 파일 생성 이후 업로드 예정 (예시: terasort-100g) |
| `concurrent_jobs` | int | **[확장안]** 동시 실행 잡 수. 동시성에 따라 최적 정책이 달라질 수 있다는 가설(확장 실험)을 위함. **(default:1)** |
| `run_index` | int | 동일 조건 내 반복 회차 (1부터 시작) |

### 그룹 2: 실험 조건 (Treatment) - RQ3 독립 변수
| 컬럼 | 타입 | 설명 |
|---|---|---|
| `staleness_ms` | long | 정책에 공급하는 클러스터 관측(observation)에 고의로 부여한 지연. **(default:0)** |

### 그룹 3: 측정 (Measurement) - 결과 그래프의 두 축

| 컬럼 | 타입 | 예시 | 설명 |
|---|---|---|---|
| `submitted_at` | ISO 8601 | `2026-07-18T14:02:11.532+09:00` | 하네스가 잡을 제출한 시각 (하네스 측 벽시계 기준) |
| `job_finished_at` | ISO 8601 | `2026-07-18T14:09:13.104+09:00` | 잡 프로세스 종료 시각 (하네스 측 벽시계 기준) |
| `ssd_byte_seconds` | double | `8.2e14` | 실험 구간 동안 SSD 티어에 상주한 데이터의 시간 적분 (∑ bytes × seconds) |
| `hdd_byte_seconds` | double | `3.1e15` | 실험 구간 동안 HDD 티어에 상주한 데이터의 시간 적분 |

> **참고:** `submitted_at`, `job_finished_at` 은 x 축의 jct_seconds, `ssd_byte_seconds`, `hdd_byte_seconds` 는 y축의 비용 (byte_seconds * 사용 단가) 으로 사용됩니다. 

**파생 지표 (분석 레이어에서 계산, CSV에는 저장하지 않음):**

- `jct_seconds = job_finished_at − submitted_at`
- `cost = ssd_byte_seconds × 단가(SSD) + hdd_byte_seconds × 단가(HDD)`

> **추가 문서:** `ssd_byte_seconds`, `hdd_byte_seconds` 의 원시 시계열 타임스탬프 파일은 /paper/data/samples 경로에 samples-{policy}-{workload}-{run}.csv 형식 파일로 저장됩니다. **(시각, SSD 상주 바이트, HDD 상주 바이트)** 값을 기록합니다. 



### 그룹 4: 출처 (Provenance) — 조건 트래킹

| 컬럼 | 타입 | 예시 | 설명 |
|---|---|---|---|
| `dsc_commit` | string | `a3f9c21` | 측정 **대상**(DSC 티어링 구현)의 git commit. 대상 버전 추적용 |
| `hfmt_commit` | string | `7be0d44` | 측정 **도구**(HFMT 자신)의 git commit. 도구 버그로 인한 이상값 추적용 |
| `config_hash` | string | `sha256:9f2c…` | 실험 설정 파일 전체의 해시. 동일 조건 여부를 한 값으로 판별 |

### 그룹 5: 유효성 (Validity) — 이 줄을 믿어도 되나

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `status` | enum | `completed` / `failed` / `aborted`. 분석 레이어는 `completed`만 사용 |
| `is_warmup` | bool | **[확장안]** default: `false` / `true`. 워밍업 여부. 워밍업은 **버리지 않고 수집**한다 — 분석에서 제외하되, 워밍업 효과 자체가 추후 분석 대상이 될 수 있다 |

---

## 3. 파일 규약

- **포맷:** UTF-8, 헤더 포함 CSV, 필드 구분자 `,`
- **저장 위치:** `paper/data/results/` 
- **파일명:** `results-{실험명}-{YYYYMMDD}.csv` (예: `results-rq1-baseline-20260801.csv`)
- **null 정책:** `status != completed`인 줄은 측정 컬럼(그룹 3)이 비어 있을 수 있다. 그 외 그룹은 항상 채운다

---

## 4. 버전 이력

| 버전 | 날짜 | 변경 내용 |
|---|---|---|
| v1 | 2026-07-18 | 최초 확정. 식별자·조건·측정·출처·유효성 5그룹, 14컬럼 |