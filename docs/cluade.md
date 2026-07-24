# HFMT Claude Code 지시서 — H0~H5

> **문서 목적:** HFMT 저장소를 H0부터 H5까지 단계적으로 구축하기 위한 Claude Code 실행 지시서.
> 각 스텝은 **CI 초록불 상태로 끝나야 하며**, Definition of Done을 전부 만족하기 전에는 다음 스텝으로 넘어가지 않는다.
>
> **참조 문서 (스텝 시작 전 반드시 읽을 것):**
> - `docs/schema-v1.md` — 결과 CSV 계약 (v1.1: `target_commit` 반영본)
> - `docs/architecture-v1.md` — 모듈 구조·포트 계약 (v1.1: 범용화 반영본)
>
> **최종 수정:** 2026-07-24

---

## 전역 규칙 (모든 스텝에 적용)

1. **범용성:** HFMT는 특정 티어링 구현에 종속되지 않는다. 측정 대상 시스템(DSC 포함)의 코드·아티팩트를 import 하거나 그 존재를 가정하는 코드를 작성하지 않는다. 대상별 차이는 설정 값으로만 표현한다.
2. **의존 방향:** `hfmt-app → hfmt-hadoop → hfmt-core`만 허용. 역방향 의존이 필요해 보이면 작업을 멈추고 사람에게 보고한다.
3. **포트 계약 준수:** 인터페이스·record·예외의 시그니처는 architecture 문서 4절과 **자구 일치**해야 한다. 개선안이 떠올라도 임의로 바꾸지 않고 제안만 남긴다.
4. **Docker 등 시스템 수준 설치는 팀의 명시적 승인 없이 진행하지 않는다.**
5. **각 스텝 완료 시:** `./mvnw verify` 로컬 통과 확인 → 커밋 → push → GitHub Actions 초록불 확인. 커밋 메시지는 `[H{n}] 요약` 형식.
6. 스텝 내 작업 중 이 지시서와 참조 문서가 충돌하면 **참조 문서가 우선**이며, 충돌 사실을 보고한다.

---

## H0 — 멀티모듈 골격 + CI

**목표:** 빈 3모듈이 빌드·CI를 통과하는 건강한 기반.

**작업:**
1. Maven Wrapper 추가 (`mvnw` 실행 권한을 git index에 포함: `git update-index --chmod=+x mvnw`)
2. 루트 pom + 3개 모듈 생성: `hfmt-core`, `hfmt-hadoop`, `hfmt-app`
   - `hfmt-core`: 의존성 없음 (JDK 11 + 테스트 라이브러리만)
   - `hfmt-hadoop`: `hfmt-core` + `hadoop-hdfs-client` (Hadoop 3.4.1 계열)
   - `hfmt-app`: `hfmt-core` + `hfmt-hadoop`
3. GitHub Actions 워크플로우: JDK 11, `./mvnw verify`
4. 의존성 화이트리스트 검사 추가 (maven-enforcer-plugin `bannedDependencies` 또는 동등 수단): 허용 목록 = JDK, `org.apache.hadoop:*`, 테스트 라이브러리(JUnit, Mockito, AssertJ). 그 외 발견 시 빌드 실패
5. 각 모듈에 더미 클래스 + 더미 테스트 1개씩 (CI가 테스트 실행까지 하는지 확인용)

**Definition of Done:**
- [ ] 로컬 `./mvnw verify` 통과
- [ ] GitHub Actions 초록불
- [ ] 화이트리스트 위반을 일부러 하나 넣었을 때 빌드가 **실제로 실패하는 것을 확인**하고, 되돌린 커밋 이력이 남아 있음
- [ ] fresh clone 후 `./mvnw verify`가 추가 설정 없이 통과 (mvnw 권한 문제 재발 방지 확인)

**금지:** 이 단계에서 실제 기능 코드 작성 금지. Hadoop 클러스터 접속 시도 금지.

---

## H1 — core: 계약의 코드화

**목표:** Schema와 포트 계약을 컴파일되는 Java 코드로 옮긴다. 로직 없음.

**작업 (전부 `hfmt-core`):**
1. `SchemaRecord`: schema 문서 v1.1의 14컬럼을 담는 record. 컬럼명·타입·순서가 문서와 일치
2. `SchemaRecord`의 CSV 직렬화/역직렬화 (헤더 포함, UTF-8). null 정책 반영: `status != completed`이면 그룹 3 컬럼은 빈 값 허용
3. 포트: `JobRunner`, `TierSampler` — architecture 문서 4절과 자구 일치
4. record: `JobExecution`, `TierSnapshot`, `SamplingResult`, `WorkloadSpec`, `RunId`, `ExitStatus`(enum)
5. 예외: `JobSubmissionException`, `SamplingException`(`runId`, `snapshotsCollected`, `lastSuccessAt`, cause 포함)

**Definition of Done:**
- [ ] CSV 왕복(직렬화 → 역직렬화 → 동등성) 테스트 통과
- [ ] `aborted` 상태 레코드의 null 정책 테스트 통과
- [ ] 포트·record·예외 시그니처가 architecture 4절과 diff 없이 일치 (수동 대조 결과를 커밋 메시지에 명기)
- [ ] `hfmt-core`에 Hadoop 의존성이 없음 (H0 화이트리스트가 이미 강제하지만, pom 재확인)

**금지:** 적분·검증 등 로직 구현 금지 (H2의 일). 인터페이스 시그니처 임의 변경 금지.

---

## H2 — core: 순수 로직

**목표:** 클러스터 없이 완전 검증 가능한 계산·판단 로직.

**작업 (전부 `hfmt-core`):**
1. `CostMath.integrate(List<TierSnapshot>) → (ssdByteSeconds, hddByteSeconds)`
   - 샘플 간격 불균등 허용 (각 스냅샷의 `observedAt` 기반 구간 계산)
   - 적분 방식(예: 구간별 좌측값 × 구간 길이)을 javadoc에 명시 — 논문 방법론 서술의 근거가 됨
2. `SamplingValidity.check(List<TierSnapshot>, Duration jobDuration) → 판정`
   - 판정 기준(최소 샘플 수, 최대 공백 허용치 등)은 상수가 아니라 파라미터로 받는다 (기준값은 설정에서 옴)
3. `ExperimentPlan`: 설정 → 실행 행렬(정책 × 워크로드 × 조건 × 반복) 생성. 실행 순서 결정 포함

**Definition of Done:**
- [ ] `CostMath` 테스트: 균등 간격, 불균등 간격, 샘플 1개, 빈 리스트(예외 또는 명시적 결과) 케이스 포함
- [ ] `SamplingValidity` 테스트: 정상/공백 과다/샘플 부족 케이스
- [ ] `ExperimentPlan` 테스트: 행렬 크기 = 정책 수 × 워크로드 수 × 조건 수 × 반복 수 확인
- [ ] 전 테스트가 클러스터 없는 CI에서 통과

**금지:** 파일시스템·네트워크 접근 코드 금지. HDFS 관련 타입 사용 금지.

---

## H3 — hadoop: 어댑터 구현

**목표:** 포트의 실제 구현. 판단하지 않고, 정직하게 접촉·보고만 한다.

**작업 (전부 `hfmt-hadoop`):**
1. `JobRunnerImpl`: `spark-submit` 프로세스 래핑. 하네스 벽시계(`java.time.Clock` 생성자 주입, 기본 `Clock.systemDefaultZone()`)로 `submittedAt`/`finishedAt` 기록. 프로세스 exit code → `ExitStatus` 매핑. 제출 자체 실패는 `JobSubmissionException`
2. `TierSamplerImpl`: 별도 스레드에서 주기 폴링(간격은 설정값). HDFS 공개 API로 티어별 상주 바이트 수집. 매 스냅샷을 원시 사이드카(`samples-{policy}-{workload}-{run}.csv`)에 즉시 append. 폴링 1회 실패는 견디고 계속, **연속 N회 실패(N은 설정값) 시 사망 판정** → `stop()`에서 `SamplingException`
3. `EnvControl`: preflight(설정에서 주입받은 헬스체크 명령 실행, exit 0 확인), 캐시 초기화 명령 실행, 정책별 초기 블록 배치(표준 storage policy 명령), 클러스터 상태 검증

**Definition of Done:**
- [ ] `JobRunnerImpl` 단위 테스트: 가짜 프로세스(짧은 셸 명령)로 시각 기록·exit 매핑·제출 실패 케이스 검증
- [ ] `TierSamplerImpl` 단위 테스트: HDFS 접근부를 인터페이스로 분리해 가짜로 대체, 사망 판정(연속 N회)·사이드카 기록·불균등 간격 스냅샷 검증
- [ ] `EnvControl` 단위 테스트: 헬스체크 명령의 성공/실패 분기
- [ ] 클러스터 필요 테스트는 `@Tag("cluster")` 등으로 분리, CI에서는 제외됨을 확인

**금지:** 적분·유효성 판정 등 판단 로직 작성 금지 (core 호출도 금지 — 어댑터는 수집만). CSV 결과 파일(Schema) 기록 금지 (app의 일). 특정 대상 시스템 이름이 코드·상수에 등장 금지 (설정 키 이름 포함).

---

## H4 — app: Orchestrator 대본

**목표:** 3국면 대본 + 실패 대응. **이 스텝의 핵심은 가짜 어댑터 시뮬레이션 테스트다.**

**작업 (전부 `hfmt-app`):**
1. `Orchestrator`: 3국면(준비 → 실행 루프 → 마무리) 구현
   - 실행 국면: 환경 초기화 → `sampler.start()` → `jobRunner.run()` 블로킹 → `sampler.stop()` → `CostMath` 적분 → `SamplingValidity` 판정 → 기록
   - `SamplingException` catch 시: `status=aborted` 레코드 기록 후 재시도 (최대 재시도 횟수는 설정값)
   - 재개(resume): 기존 결과 CSV를 읽어 완료된 (정책×워크로드×조건×반복) 조합은 건너뜀
2. `ResultWriter`: 실행 1회 종료 즉시 append + flush. 마무리 국면에서 무결성 검증(줄 수, 필수 컬럼)
3. `Provenance`: git commit(target/hfmt)·config hash 수집. 대상 커밋은 설정에 지정된 경로에서 읽음
4. 오류 보고서: 마무리 국면에서 `status != completed` 줄들의 요약 생성
5. CLI 진입점: 설정 파일 경로를 받아 실험 실행

**Definition of Done:**
- [ ] **시뮬레이션 테스트:** 가짜 `JobRunner`/`TierSampler`로 정책 2 × 반복 3의 전체 실험을 수 초 내 실행, CSV 6줄 생성 확인
- [ ] **실패 시나리오 테스트:** 가짜 샘플러가 특정 회차에 `SamplingException`을 던지도록 설정 → CSV에 `aborted` 1줄 + 재시도 성공 1줄이 올바른 순서로 남는지 확인
- [ ] **재개 테스트:** 3줄 기록된 CSV를 두고 재실행 → 완료분 3개 조합은 건너뛰고 나머지만 실행됨을 확인
- [ ] **중도 종료 테스트:** 실행 도중 프로세스가 죽어도(테스트에서 시뮬레이션) 이미 기록된 줄이 손실되지 않음
- [ ] 전 테스트가 클러스터 없는 CI에서 통과

**금지:** Hadoop 타입 직접 사용 금지 (포트를 통해서만). 통계 계산(평균 등) 금지 — 분석 레이어의 일.

---

## H5 — 통합 리허설: 첫 실데이터 점

**목표:** 실제 클러스터에서 첫 번째 대상 시스템(DSC — HFMT의 주인이 아니라 **첫 손님**)으로 B1 1회 실행.

**작업:**
1. DSC용 설정 파일 작성 (`config/dsc-b1.yaml` 등): 클러스터 주소, 헬스체크 명령, 정책 이름 B1, 워크로드 1개, 반복 1회
2. 실제 클러스터에서 CLI 실행
3. 산출물 검증 및 커밋: 결과 CSV 1줄(`paper/data/results/`), 사이드카 1개(`paper/data/samples/`)

**Definition of Done:**
- [ ] CSV 1줄이 schema v1.1의 14컬럼을 모두 채우고 `status=completed`
- [ ] `jct_seconds`(파생 계산)와 byte_seconds 값이 상식적 범위 (예: JCT > 0, ssd+hdd > 0)
- [ ] 사이드카의 스냅샷 수 ≈ 잡 실행 시간 / 폴링 간격 (±1 허용)
- [ ] Provenance 컬럼(target_commit, hfmt_commit, config_hash)이 실제 값으로 채워짐
- [ ] 산출물이 저장소에 커밋됨 (CI as source of truth)

**금지:** 이 단계에서 발견된 버그의 수정은 해당 스텝(H1~H4)의 테스트 보강과 함께 커밋 — 테스트 없는 핫픽스 금지.

---

## 스텝 진행 기록

| 스텝 | 상태 | 완료일 | 비고 |
|---|---|---|---|
| H0 | 대기 | | |
| H1 | 대기 | | |
| H2 | 대기 | | |
| H3 | 대기 | | |
| H4 | 대기 | | |
| H5 | 대기 | | |