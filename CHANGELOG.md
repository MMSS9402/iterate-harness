# Changelog

이 프로젝트의 주요 변경 사항을 기록한다. 형식은 [Keep a Changelog](https://keepachangelog.com/ko/1.1.0/)를 따르고, 버전은 [유의적 버전](https://semver.org/lang/ko/)(0.y.z 개발 단계)을 따른다.

> **릴리스 관행**: version 범프 커밋에는 `git tag v<버전>` 을 함께 남기고, 커밋 메시지의 버전 표기가 `plugin.json` 의 `version` 과 일치하는지 확인한다(불일치 전례: v0.4.0 커밋, 아래 각주).

각 버전에는 **어댑터 영향** 소절을 둔다 — 대상 레포의 `.claude/iterate.config.md` 에 필드 추가/변경이 필요한지가 사용자 업그레이드 비용의 전부이기 때문이다.

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
