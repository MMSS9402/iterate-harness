# Changelog

이 프로젝트의 주요 변경 사항을 기록한다. 형식은 [Keep a Changelog](https://keepachangelog.com/ko/1.1.0/)를 따르고, 버전은 [유의적 버전](https://semver.org/lang/ko/)(0.y.z 개발 단계)을 따른다.

> **릴리스 관행**: version 범프 커밋에는 `git tag v<버전>` 을 함께 남기고, 커밋 메시지의 버전 표기가 `plugin.json` 의 `version` 과 일치하는지 확인한다(불일치 전례: v0.4.0 커밋, 아래 각주).

각 버전에는 **어댑터 영향** 소절을 둔다 — 대상 레포의 `.claude/iterate.config.md` 에 필드 추가/변경이 필요한지가 사용자 업그레이드 비용의 전부이기 때문이다.

## [0.8.0] - 2026-07-20

토큰·시간 최적화 릴리스 — 실측(반복 회차가 비용의 대부분·explorer 회당 10만+ 토큰·designer 스킵 판정 비용) 기반으로 **기본 사이클을 lean 으로 경량화**하고 QA·디자인을 별도 배치 명령으로 분리했다. 기계 백스톱(동결 체크섬·mutation·GUARDS·LINT)은 모드 무관 동일 — 조절 대상은 LLM 심사의 반복 횟수뿐이다.

### Added
- **실행 모드 lean(기본)/deep**: `/iterate <카드> [N] [deep]`. lean = G2 델타 심사·reviewer 본심사 1회·explorer 큐·designer 캐시. deep = 기존 완전 심사(전체 G2 재심사·reviewer 재사냥·explorer 인라인). SSOT §실행 모드 신설.
- **`/iterate-qa`(신규 명령)**: lean 이 `qa_queue.md` 에 등록한 explore 카드들을 환경 1회 기동으로 묶어 배치 탐색. 카드×초점 fan-out 에 **Workflow 병렬 허용**(하네스 유일 예외 — 격리 규칙은 fan-out 에이전트에도 동일 적용, 게이트 판정 권한은 불이동). 카드 간 교차 상호작용 항목 포함(단일 카드 탐색이 원리상 못 보는 것). 결함은 발견·기록까지 — 테스트 환원은 해당 카드 `/iterate` 재실행이 담당.
- **`/iterate-design`(신규 명령)**: designer 를 사이클 밖에서 화면군 단위 1회 실행 → `visual_spec` 산출·`design_cache.md` 등록(DESIGN_SSOT shasum 이 캐시 키). 디자인 확정 프로젝트의 사이클 비용 상각.
- **designer 스펙 캐시**: design 카드라도 (a) 화면군 visual_spec 존재 (b) DESIGN_SSOT 다이제스트 불변 (c) 기존 화면군 확장이면 designer 스킵·스펙 재사용(드라이버 기계 판정).

### Changed
- **G2 델타 심사(lean·재심사 한정)**: 카드 최초 G2 는 전체 심사 유지. 재심사는 변경·신규 테스트 파일 + 해당 AC + 기존 커버리지와의 상호작용(매트릭스) + 직전 Escapes 회귀 부록으로 한정 — 체크섬 불변 파일의 직전 escapes-0 판정은 유효 재사용. deep 은 매번 전체.
- **reviewer 수렴 규칙(lean)**: 본심사 카드당 1회 — 갭이 전부 환원되어 게이트 green 이면 해소 판정(재사냥 없음·G2 델타+mutation 백스톱). 환원 과정에서 구현이 새 표면을 추가했으면 그 표면 한정 재검토 1회 허용. deep 은 갭 0 까지 반복.
- **explorer 분리(lean)**: 사이클 인라인 → 자동 green 시 `qa_queue.md` 등록(상태 `green (QA 대기)`), `/iterate-qa` 가 배치 소비. deep 또는 카드 `explore-inline` 명시 시 기존 인라인 유지.
- **Workflow 금지 규정 개정**: 사이클 안 금지는 유지(의존 체인·게이트 직렬 책임), `/iterate-qa` 배치 fan-out 만 허용.
- **green-bar 갱신**: reviewer 항목 = 모드별 해소 조건, explore 항목 = lean 큐 등록/deep 결함 0. explore 큐 카드의 자동 green 은 `green (QA 대기)` — 배치가 결함 0 확인 시 done.
- **Explore(빌트인) lean 완화**: 새 모듈 영역을 만질 때만 — 기존 영역 재작업이면 드라이버 Grep 2~3회로 대체 가능.

### 어댑터 영향
- **없음(필드 추가 불요)** — 기존 `EXPLORE_QA`·`DESIGN_SSOT` 를 그대로 쓴다. `qa_queue.md`·`design_cache.md`·`qa_findings.md` 는 기존 `ARTIFACTS_DIR` 안에 생긴다(이미 gitignore). 기존 카드 원장의 `explore` 플래그 의미가 "인라인 탐색"→"큐 등록(기본)"으로 바뀌므로, 인라인을 유지하고 싶은 카드에만 `explore-inline` 표기 추가.

## [0.7.0] - 2026-07-07

3차 감사(드라이-런 시뮬레이션·누적 편집 회귀) 반영 릴리스 — 드라이버가 절차서대로 굴렀을 때 실제로 막히는 지점을 수리했다.

### Fixed (P1 — 실행 결함)
- **하드코딩 줄-상한 find 삭제 → 어댑터 GUARDS 일원화**: §2 게이트의 무필터 find(소스 확장자·vendor/.git/ARTIFACTS_DIR prune 없음)가 Go 어댑터에서 go.sum·vendor·.git 팩파일로 거짓 위반을 확정 생성 → architect 가 못 고치는 원인으로 카드가 영구 종결되던 결함. 줄-상한 판정은 어댑터 GUARDS 의 줄-상한 가드로 일원화, Step 0a 가 가드 부재 시 게이트 진입 전 중단.
- **골든 규약의 기계 실행 가능화**: 골든 테스트의 기계 식별(태그/글롭)·기계 게이트/mutation 제외·사람 승인 후 baseline 생성 스텝 신설 — design+human-visual 카드가 baseline 부재로 데드락되던 결함 해소.
- **e2e 테스트 동결 범위 확인**: e2e 플래그 카드에서 E2E_CMD 가 실행하는 테스트 경로가 동결 열거(TEST_FILE_GLOB ∪ TEST_SUPPORT_GLOBS)에 덮이는지 0b 에서 확인 — 밖이면 implementer 의 e2e 테스트 변조를 동결 체크섬·가드가 못 잡는 구멍이었다.

### Changed
- **reviewer 입력 규정 모순 교정**: "test-author 와 동일한 SSOT 입력 제한" 오기를 교정 — test-author·test-auditor 는 spec.md, reviewer 는 plan.md 를 받되 Rationale·Notes 열람만 차단(섹션 제한 예외). SSOT·README·CHANGELOG 동기화.
- **g2_verdicts.md 에서 Mutation targets 제외**: 판정 원문 append 가 '드라이버만 보유' 기밀 규칙과 충돌하던 것을 교정 — Mutation targets 는 ARTIFACTS_DIR 에 쓰지 않고 드라이버 대화 컨텍스트에만 기록.
- **human-visual↔HUMAN_GATE 조기 차단**: design/e2e/explore 와 동형으로 0b 카드 플래그 판정·architect 플래그 추가 재검사 양쪽에 HUMAN_GATE 미설정 차단 추가.
- **E2E/테스트 출력 리다이렉트 규율**: E2E 원시 출력을 파일로만 받고 실패 시 tail 만 Read — 대량 출력의 드라이버 트랜스크립트 유입 차단(게이트 테스트 출력도 동일 규율).
- **컴팩션 재수화 확장**: G2 다이제스트 외에 가드 '전' 다이제스트·escape 반려 카운터·N 상한까지 복구 — iteration_log 에 N 상한·플래그·escape 라우팅 1줄 append 의무화.
- **iterate-init 실효성 보강**: 래퍼 커맨드 채택 조건(trailing 인자 계약·watch 모드 금지 — Makefile 타깃/npm scripts 함정) + TEST_CMD 스모크 실행 검증(자가 검증 6번 — exit 127/'No rule to make target'/timeout 구분) + TODO 잔존 시 Step 0a 거부 계약 명시 + 3예시 밖 스택은 '전부 TODO' 대신 §2 실측 수행·실측 실패 값만 TODO + PROJECT_INVARIANTS/TEST_SCOPE_RULES 질문 스크립트.

### 어댑터 영향
- **GUARDS 에 FILE_LINE_LIMIT 참조 줄-상한 가드 필수** — 없으면 드라이버가 Step 0a 에서 중단하고 어댑터 보강을 요청한다(기존 3예시 골격에는 이미 존재).
- **HUMAN_GATE 골든 규약을 쓰면 기계 식별 수단(태그 또는 글롭) 필수** — 예: Flutter `@Tags(['golden'])` + `--exclude-tags golden`.
- **E2E 테스트 경로는 동결 글롭 안에** — E2E_CMD 가 실행하는 테스트 파일 경로를 TEST_FILE_GLOB ∪ TEST_SUPPORT_GLOBS 안에 두라(예시 A 에 `integration_test/**` 추가됨).

## [0.6.0] - 2026-07-07

2026-07 감사(audit) 반영 릴리스 — 게이트 엄격성은 구조로, 비용은 라우팅으로.

### Added
- **`/iterate-harness:iterate-init`** (commands/iterate-init.md): 대화형 어댑터 부트스트랩 — 스택 감지(Flutter/Go/Web + 3예시 밖 스택은 TODO 골격) → 레포 실측으로 값 확정 → 단일-블록 어댑터 생성 + 자가 검증(필드 헤딩·코드펜스 1개·TEST_FILE_GLOB 매칭 수·GUARD 경로 실존). 기존 어댑터는 덮어쓰지 않고 누락 필드만 보강.
- **`TEST_SUPPORT_GLOBS` 어댑터 필드**: 동결 스냅샷 범위를 테스트 파일(TEST_FILE_GLOB)에서 테스트 지원 파일(헬퍼·픽스처·Fake)까지 확장 — test-author 가 쓸 수 있는 모든 것이 동결 대상.
- **README 퀵스타트(5분)** + **§6c 모델 라우팅과 비용**: 역할×모델 표, 게이트 실패 = 전체 파이프라인 재실행이라는 비용 구조, 로컬 오버라이드 경로, 강등 가이드.
- **plan.md 영속화 + 재사용 기계 판정**: architect 산출을 `$ARTIFACTS_DIR/plan.md` 로 영속화(직전분은 plan.prev.md), 재실행 시 diff 가 `### Notes` 밖에서 비어 있으면 designer/test-author/G2 스킵 + 직전 동결 유지 — 자가보고가 아니라 diff 가 유일한 판정자.
- 드라이버 컨텍스트 위생: 회차별 `iteration_log.md` append + 컴팩션/세션 재개 후 재수화 절차.

### Changed
- **모델·effort 라우팅**: implementer 는 `sonnet` 으로 강등(동결 테스트+체크섬+mutation+GUARDS+reviewer 가 품질을 기계 강제하는 유일한 역할), 검사자·출제자·설계자(architect·test-author·test-auditor·reviewer)와 explorer 는 `opus` + effort `xhigh` 고정, designer 는 opus/high — 게이트 심사 깊이가 세션 effort 에 좌우되지 않게 frontmatter 로 고정.
- **백스톱 다이제스트 강화**: `git status --porcelain | shasum`(상태 전용 — dirty/untracked 파일의 내용 변조를 못 잡음)을 status+diff+untracked 내용 체크섬 결합 다이제스트로 교체. 동결 스냅샷 자체의 다이제스트를 드라이버 대화 컨텍스트에 고정해 디스크 변조를 검출.
- **mutation 로그 위생**: 변형당 테스트 출력을 `$ARTIFACTS_DIR/mutlogs/` 로 리다이렉트, 트랜스크립트에는 exit 마커 한 줄만 — 변형 N개 × 전체 테스트 로그의 드라이버 컨텍스트 유입 차단. kill/INVALID 분류는 exit≠0 인 로그의 tail 만 Read.
- reviewer 의 verdict.json 자가채점 잔재 제거, polish 재확인 섹션 제거 등 구세대 잔재 정리.

### 독립성 보강 (2차 감사)
- **발췌본 전달**: 출제자(test-author)·G2 검사자(test-auditor)에게 plan.md 원문 대신 드라이버 발췌본 `spec.md`(Card flags·Interface·AC·Visual)를 전달 — architect 의 Rationale(구현 접근)·Notes(실패 프레이밍) 노출 차단. (reviewer 는 예외 — plan.md 를 받되 Rationale·Notes 열람만 차단하는 섹션 제한. 0.7.0 에서 문구 모순 교정.)
- **검사자 커밋먼트 규격**: test-auditor 는 테스트 열람 전 명세에서 `Mutant checklist` 를 먼저 고정, reviewer 는 diff/테스트 열람 전 `Expected` 를 먼저 고정 — 드라이버가 산출물 형식으로 검증.
- **G2 확장**: AC 형식 감사(0단계 — 불변식 6/8 위반형·검증불가 AC 지목, ≥1 이면 architect 재계획 라우팅) + `Mutation targets` 산출(escapes 0 이어도 필수 — mutation 게이트의 우선순위 메뉴, 드라이버만 보유).
- **바운스 재호출 전달물 규격**: 검사자 산출물의 발견 항목만 전달하고 판정 서사(Verdict·Covered·제안 케이스)와 피검사자의 수정 설명은 전달 금지 — G2 재검사는 fix 확인이 아니라 전체 재심사.
- **Notes-only 스킵 예외**: mutation 생존·reviewer 갭·explorer 결함 환원(테스트 얕음) 시 plan diff 가 Notes-only 여도 test-author→G2→재동결은 반드시 흘린다(스킵 허용은 designer 뿐).
- **동결 다이제스트 컴팩션 폴백**: 컨텍스트 앵커 소실 시 iteration_log 의 저신뢰 사본·디스크 대조 + 사용자 고지 후 G2 재심사→재동결로 엄격성 재수립.
- **감사 추적**: 검사자 판정 원문 영속화(`g2_verdicts.md`·`reviewer_verdicts.md`) + 독립성 위반 인시던트 기록(`violations.md`, 카드 간 누적) + mutation-kill-rate 수치를 영속 기록 3곳(iteration_log·완료기록·종료 보고)에 기재.
- 어댑터 영향: 없음(어댑터 필드 변경 없음).

### 어댑터 영향
- **필드 신설(조치 필요)**: `TEST_SUPPORT_GLOBS` — 기존 어댑터에 헤딩을 추가해야 한다. 테스트 헬퍼·픽스처·Fake 패턴을 선언하고, 없으면 `(없음)`. 이 패턴 밖에 test-author 가 지원 파일을 만들면 드라이버가 반려한다.
- 그 외 필드 변경 없음. 선택 필드 sentinel 규칙(코드펜스 부재·빈 값·`(없음)` = 미설정)이 명문화됐다.

## [0.5.0] - 2026-07-06

커밋: `08d546a`

- **동형 갭 일괄 시정**: reviewer 갭 재실행 시 architect 가 같은 원인·다른 소비처의 결함 표면을 전수 나열해 한 회차에 묶는다(T2.8a 실측 — 동형 갭 분할 발견 1회 = 재파이프라인 ≈1시간 낭비 차단). Notes(비차단) 분류도 같은 원인 계열이면 재검토 대상.
- **SPEC GATE 제거**(사용자 결정 2026-07-06): 구현 전 사람 스펙확인 단계 폐지 — human-visual 게이트와 확인이 이중.

### 어댑터 영향
- 없음(프로토콜·드라이버 문서 변경만).

## [0.4.0] - 2026-07-05

커밋: `a2635b5` [^v040]

- **SPEC GATE 도입**: new-UX 카드(신규 화면/플로우·인터랙션 모델 변경·데이터 노출 정책 변경)는 테스트·구현 전 사람 스펙확인 1회 — T2.6b·T2.9c 실측에서 구현 후 스펙이 뒤집혀 라운드 전체가 폐기되는 낭비 근거. (0.5.0 에서 제거됨.)

### 어댑터 영향
- 없음(SSOT·드라이버에 단계 삽입만).

[^v040]: 커밋 메시지의 "(v0.2.0)" 표기는 **오기**다 — 해당 커밋의 plugin.json diff 는 0.3.0→0.4.0 범프로, 실제 버전은 0.4.0 이다(git log -p 실측). 버전 이력 자체는 0.1.0→0.5.0 완전 단조.

## [0.3.0] - 2026-07-04

커밋: `978803c`

- **explorer 역할(7번째) + `explore` 카드 플래그**: reviewer 갭 0 후·자동 green 확정 직전에 실물을 구동해 명세(AC) 밖 — 인접 흐름·이상 상태·중단/역행·연타·경계 밖 입력 — 을 시간상자 탐색. 결함은 test-author 케이스로 환원(G2 재심사→재동결 경유). green-bar 에 "explore 없음 OR 결함 0" AND 조건 추가.
- README 에 플러그인 업데이트 절차 문서화.

### 어댑터 영향
- **선택 필드 신설**: `EXPLORE_QA`(기동·조작·초점 3줄) — `explore` 플래그 카드를 쓰려면 필요, 미설정이면 explore 플래그 사용 불가(기존 어댑터는 무변경 호환). `ROLE_EXPLORER` 파인튜닝 슬롯 추가(선택).

## [0.2.0] - 2026-07-04

커밋: `804e6c5` (커밋 메시지에 버전 라벨 없음 — plugin.json diff 0.1.0→0.2.0)

- **`ROLE_*` 역할별 파인튜닝 계층**: 어댑터에 선택 섹션을 두면 각 에이전트가 자기 섹션을 Read 해 추가 지침으로 따름 — SSOT 불변식·격리·게이트 구조가 항상 우선(게이트 약화 지시는 무시+충돌 보고).
- **예시 C · React/TS Web** 어댑터 예시 추가(vitest·tsc+eslint·Playwright·msw·jsdom seam).
- 게이트 무결성 보강: 테스트 변경은 경로 불문 G2 재심사→재동결 경유, E2E timeout 바이너리 부재 시 무-timeout 실행 금지.

### 어댑터 영향
- **선택 섹션 신설**: `ROLE_ARCHITECT`~`ROLE_REVIEWER`(전부 선택 — 기존 어댑터 무변경 호환). 템플릿에 Web 스택 예시가 추가됐을 뿐 필수 필드 변경 없음.

## [0.1.0] - 2026-07-04

커밋: `934d098`

- 최초 릴리스: 멀티에이전트 TDD 하네스 — 출제자≠검사자≠구현자 격리(P1), 테스트 동결(filelist+sha256), Mutation 게이트, 사람 시각확인 게이트. 언어 무관 코어(commands·agents·SSOT) + 프로젝트 어댑터(`.claude/iterate.config.md`) 분리.

### 어댑터 영향
- 어댑터 스키마 최초 도입 — `examples/iterate.config.md` 템플릿(Flutter·Go 예시) 기준으로 레포당 1회 작성.
