# HFMT Architecture v1

> **문서 목적:** HFMT(HDFS File System Measurement Tool)의 모듈 구조(정적 뷰)와 실행 흐름(동적 뷰), 두 뷰를 잇는 규칙, 그리고 core 포트 인터페이스 계약을 확정한다.
>>
> **상태:** 확정 (v1.1)
> **최종 수정:** 2026-07-24

---

## 1. 아키텍처 스타일

**포트와 어댑터 (헥사고날) + functional core, imperative shell.**

- 순수 로직(계산·판단)은 중심(core)에 두고, 외부 세계(HDFS, YARN, 파일시스템, 프로세스)와의 접점은 어댑터로 격리한다.
- 실험의 시간적 흐름(국면 1→2→3)은 모듈 경계가 아니라 **Orchestrator의 내부 대본**으로 표현한다.
  - 근거: 시간 기반 분할(temporal cohesion)은 같은 지식(캐시 제어, CSV, HDFS 통신)을 여러 모듈에 중복시킨다. 모듈은 "방법을 누가 아는가"를, 대본은 "언제 하는가"를 답한다.

### 지켜야 할 세 가지 규칙

1. **의존은 아래로만.** `hfmt-app → hfmt-hadoop → hfmt-core`. 역방향 의존 금지.
2. **인터페이스는 core에, 구현은 어댑터에.** Orchestrator 로직은 가짜(fake) 어댑터로 클러스터 없이 테스트 가능해야 한다.
3. **세 모듈 모두 측정 대상 시스템의 아티팩트 import 금지.** 측정 독립성은 CI의 **허용 의존성 화이트리스트**(JDK, hadoop-client 계열, 테스트 라이브러리)로 강제한다 (아래 7절). 화이트리스트 방식은 특정 대상 차단보다 강한 검사다.

---

## 2. 정적 뷰 — 모듈 구조

```
hfmt-app     (조립과 지휘)          Orchestrator · ResultWriter · Provenance
   ↓ 의존
hfmt-hadoop  (바깥세상 어댑터)       JobRunnerImpl · TierSamplerImpl · EnvControl
   ↓ 의존
hfmt-core    (순수 코어, Hadoop·대상 시스템 의존 없음)
                                    ExperimentPlan · CostMath · Ports · SchemaRecord
```

### hfmt-core — 순수 코어

Hadoop·DSC 의존성 없음. 모든 테스트가 클러스터 없는 CI에서 실행 가능.

| 컴포넌트 | 책임 |
|---|---|
| `ExperimentPlan` | 실험 설정 → 실행 행렬(정책 × 워크로드 × 조건 × 반복) 생성. 실행 순서 결정 |
| `CostMath` | `List<TierSnapshot>` → byte-seconds 적분. 순수 함수 |
| `SamplingValidity` | 샘플 커버리지 등 측정 유효성 판정. 순수 함수 (판정 기준은 논문 방법론에 서술되는 결정이므로 core에 둔다) |
| Ports (`JobRunner`, `TierSampler`) | 어댑터가 구현할 인터페이스 (4절) |
| `SchemaRecord` | Schema v1의 Java 표현. CSV 한 줄 = 레코드 하나 |

### hfmt-hadoop — 어댑터

core의 포트를 구현. 외부 세계와의 접점만 담당하며, 판단하지 않는다.

| 컴포넌트 | 책임 |
|---|---|
| `JobRunnerImpl` | `spark-submit` 래핑, 하네스 벽시계로 제출·종료 시각 기록 |
| `TierSamplerImpl` | 백그라운드 주기 폴링, 원시 샘플 사이드카(`samples-*.csv`) 기록 |
| `EnvControl` | 사전 점검(preflight), 캐시 초기화, 정책별 초기 블록 배치, 클러스터 상태 검증. **preflight의 대상 시스템 헬스체크는 설정 파일에서 주입받은 명령을 실행해 exit 0 여부만 확인** — 그 명령이 무엇을 검사하는지 HFMT는 모른다 (대상 지식은 코드가 아니라 설정에 산다) |

### hfmt-app — 조립과 지휘

| 컴포넌트 | 책임 |
|---|---|
| `Orchestrator` | 실험 대본(3절) 실행. 재시도·재개(resume) 로직 |
| `ResultWriter` | 실행 1회 종료 즉시 append + flush. 부분 실패 시에도 완료분 보존. 마무리 국면의 무결성 검증 |
| `Provenance` | 실험 시작 시 git commit(target/hfmt)·config hash 1회 수집. 대상 커밋은 설정에 지정된 경로에서 읽는다 |

---

## 3. 동적 뷰 — Orchestrator의 실험 대본

```java
void runExperiment(ExperimentPlan plan) {
    prepare(plan);       // 국면 1
    for (Run run : plan.runs()) {
        executeOne(run); // 국면 2 (재시도 포함)
    }
    finalize();          // 국면 3
}
```

| 국면 | 하는 일 | 호출되는 컴포넌트 |
|---|---|---|
| 1. 준비 | 사전 점검(preflight), 커밋·해시 수집, 실행 행렬 생성 | `EnvControl.verifyPreconditions()` · `Provenance` · `ExperimentPlan` |
| 2. 실행 (×N) | 매 회차: 환경 초기화 → 잡 실행 ∥ 티어 샘플링 → 즉시 기록. 샘플러 실패 시 `aborted` 기록 후 재시도 | `EnvControl` · `JobRunner` + `TierSampler`(병행) · `ResultWriter` |
| 3. 마무리 | CSV 무결성 검증, 오류 보고서 생성 | `ResultWriter` · 오류 요약 |

`EnvControl`이 국면 1과 2에서 재사용되는 것에 주목 — 캐시·환경 지식은 한 곳에만 살고, 호출 시점만 Orchestrator가 정한다.

실행 국면의 병행 조립 (비동기 관리는 Orchestrator의 책임, 포트는 동기):

```java
sampler.start(runId);
JobExecution exec = jobRunner.run(workload);   // 블로킹
SamplingResult samples = sampler.stop();
```

---

## 4. 포트 인터페이스 계약 (core)

### JobRunner

```java
public interface JobRunner {
    /** 워크로드를 제출하고 종료까지 블로킹. 실행 국면에서 TierSampler와 병행 호출됨. */
    JobExecution run(WorkloadSpec workload) throws JobSubmissionException;
}

/** Schema v1 그룹 3의 시간 측정 원재료 + 유효성 판단 재료 */
public record JobExecution(
    Instant submittedAt,
    Instant finishedAt,
    ExitStatus status        // COMPLETED / FAILED — CSV status로 변환
) {}
```

설계 근거:
- 반환 타입은 Schema v1 그룹 3에서 역산했다.
- 시계는 인터페이스에 노출하지 않는다. 테스트용 `java.time.Clock`은 구현 클래스의 생성자 주입으로 해결한다.
- 비동기는 계약이 아니라 호출자(Orchestrator)의 결정이다.

### TierSampler

```java
public interface TierSampler {
    /** 백그라운드 주기 폴링 시작. 원시 샘플 사이드카 기록도 여기서 시작. */
    void start(RunId runId) throws SamplingException;

    /** 폴링 중지, 수집된 샘플 반환. 샘플러가 복구 불가능하게 죽어 있었다면 예외. */
    SamplingResult stop() throws SamplingException;
}

public record TierSnapshot(Instant observedAt, long ssdBytes, long hddBytes) {}

public record SamplingResult(
    List<TierSnapshot> snapshots,   // CostMath.integrate()의 입력
    Path rawSidecarFile             // samples-{policy}-{workload}-{run}.csv 위치
) {}
```

설계 근거:
- `SamplingResult`는 적분값이 아니라 스냅샷 목록을 반환한다. 적분은 core(`CostMath`)의 책임 ("상류의 값" 원칙의 인터페이스 적용).
- `TierSnapshot.observedAt` 덕분에 폴링 간격이 불균등해도 적분이 정확하다.

### 예외 계약

| 상황 | 표현 방식 | 근거 |
|---|---|---|
| 잡 제출 실패 (클러스터 문제) | `JobSubmissionException` throw | 흐름 중단, 재시도 가치 있음 |
| 잡 실행 중 실패 (워크로드/정책 문제) | `ExitStatus.FAILED` **반환** | 실험의 결과이므로 기록 대상. 정상 흐름에서 처리 |
| 샘플러 사망 (연속 폴링 실패) | `SamplingException` throw | **측정의 실패** — 해당 실행은 측정으로서 무효, 재시도 외 대안 없음 |

구분 기준: *정상 흐름에서 기록·처리되는 결과면 반환값, 흐름을 중단시키고 재시도에 맡길 일이면 예외.*

추가 규칙:
- **예외는 화물을 싣는다.** `SamplingException`은 `runId`, `snapshotsCollected`, `lastSuccessAt`, cause를 포함한다 — 마무리 국면의 오류 보고서 재료.
- **예외 ≠ 기록 생략.** Orchestrator는 샘플러 예외를 잡으면 `status = aborted`로 CSV 한 줄을 남긴 뒤 재시도한다 (Schema v1 null 정책 적용).
- **일시 실패와 사망의 구분은 어댑터의 규칙.** 폴링 1회 타임아웃은 견디고 계속한다. 연속 N회 실패 시 사망 판정 (N은 설정값).
- **유효성 판단은 core.** "샘플이 듬성듬성한데 유효한가"는 `SamplingValidity`(순수 함수)가 판정한다. 어댑터는 정직하게 모은 것을 보고할 뿐, 판단하지 않는다.

---

## 5. 모듈 물리 구조

Maven 멀티모듈 3개 (`hfmt-core`, `hfmt-hadoop`, `hfmt-app`). DSC에서 멀티모듈 운영 경험이 있으므로 처음부터 분리한다.

- `hfmt-core`: 의존성 없음 (JDK만). 테스트는 전부 순수 단위 테스트
- `hfmt-hadoop`: `hfmt-core` + `hadoop-hdfs-client` 등
- `hfmt-app`: `hfmt-core` + `hfmt-hadoop`

---

## 6. 분석 레이어와의 경계

- 하네스(Java)의 출력 = Schema v1 CSV + 원시 샘플 사이드카. 여기까지가 이 저장소의 책임.
- 통계(평균·중앙값·오차 막대)·단가 환산·그래프 생성은 분석 레이어(Python)의 책임.
- 두 세계의 계약은 Schema v1 문서이며, 스키마 변경은 반드시 버전 증가를 동반한다.

---

## 7. CI 강제 사항

- core 테스트는 클러스터 없이 GitHub Actions에서 전부 실행된다.
- 의존성 검사: 허용 화이트리스트(JDK, hadoop-client 계열, 테스트 라이브러리) 외 아티팩트 발견 시 빌드 실패. 측정 대상 시스템의 아티팩트는 정의상 화이트리스트에 들어갈 수 없다.
- (선행 프로젝트에서의 학습 반영) `mvnw` 실행 권한 확인, 리소스 경로 검증을 초기 워크플로우에 포함.

---

## 8. 버전 이력

| 버전 | 날짜 | 변경 내용 |
|---|---|---|
| v1 | 2026-07-19 | 최초 확정. 3계층 모듈, 3국면 대본, 포트 2종 + 예외 계약 |
| v1.1 | 2026-07-24 | 범용화: 대상 시스템 비종속 선언, 의존성 검사를 화이트리스트로 전환, preflight 헬스체크를 설정 주입으로 변경 |