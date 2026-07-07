---
description: 대화형 어댑터 부트스트랩 — 대상 레포의 스택을 감지하고 실측으로 값을 채워 .claude/iterate.config.md 를 생성한다. /iterate 실행 전 레포당 1회. 실측 불가 필드는 TODO + 사용자 질문으로 남긴다(지어내지 않음).
argument-hint: [대상 레포 경로 — 선택(기본: 현재 디렉터리)]
---

당신(메인 Claude)이 **어댑터 부트스트래퍼**다. 대상 레포(인자 경로 또는 현재 디렉터리)에 `/iterate` 가 요구하는 프로젝트 어댑터 `.claude/iterate.config.md` 를 **실측 기반**으로 생성한다. 원칙: **레포에서 확인한 값만 쓴다 — 확인 못 한 값은 지어내지 말고 `TODO:` 표기 + 사용자 1줄 질문.** 코어는 언어/프레임워크 무관이다 — 스택 특화는 전부 이 어댑터에 담긴다.

**대상 레포**: $ARGUMENTS (비어 있으면 현재 디렉터리)

## 0) 템플릿 로드 + 기존 어댑터 확인 (필수·먼저)

- `${CLAUDE_PLUGIN_ROOT}/examples/iterate.config.md` 를 Read — 필드 스키마·파싱 계약·스택별 예시 3종(A · Flutter / B · Go / C · React-TS Web)의 정본.
- **대상 레포에 `.claude/iterate.config.md` 가 이미 있으면 절대 덮어쓰지 않는다.** 기존 파일을 Read 해 아래 필수 필드 목록과 대조 → 누락 필드만 나열하고 "이 필드들을 보강할까요?" 1줄 확인 후, **누락 필드 블록만 append/Edit** 한다(기존 값 무수정). 누락 0이면 "어댑터 완비"만 보고하고 §4 자가 검증으로 직행.

**필수 필드(전부 생성)**: `PROJECT` · `TEST_CMD` · `LINT_CMD` · `TEST_SCOPE_RULES` · `GUARDS` · `FILE_LINE_LIMIT` · `FROZEN_DIR` · `ARTIFACTS_DIR` · `PROJECT_INVARIANTS` · `TEST_FILE_GLOB` · `TEST_SUPPORT_GLOBS` · `MUTATION_EXCLUDES`
(README §5-1 필수 12종과 동일. `TEST_SUPPORT_GLOBS` 는 헤딩 자체는 필수 — 지원 파일 패턴이 없으면 값을 `(없음)` 으로.)
**선택 필드(해당 시만)**: `BUILD_GEN_CMD` · `E2E_CMD` · `PLAN_PATH` · `DESIGN_SSOT` · `HUMAN_GATE` · `EXPLORE_QA` · `ROLE_*`

## 1) 스택 감지

레포 루트에서 마커 파일을 확인한다:

| 마커 | 스택 | 골격 |
|---|---|---|
| `pubspec.yaml` | Flutter | 예시 A |
| `go.mod` | Go | 예시 B |
| `package.json` | Web(React/TS 등) | 예시 C |

- **감지 실패(마커 0개) 또는 복수 감지(모노레포 등)**: 사용자에게 1줄 질문 — "이 레포의 주 스택/대상 서브디렉터리는?" 답을 받아 골격을 정한다.
  - ★ 함정: `pubspec.yaml` 과 `package.json` 이 공존하는 Flutter 레포가 흔하다(웹 빌드 부산물) — 루트 마커 우선순위는 pubspec.yaml > go.mod > package.json 로 두되, 애매하면 반드시 물어라.
- **3예시 밖 스택**(Python·Rust·JVM 등): 가장 가까운 골격을 고르고(백엔드류→B, 프런트류→C), **§2 실측을 동일하게 전부 수행**한다 — Makefile·CI 워크플로·설정 파일(pyproject.toml·build.gradle 류)에서 확인된 커맨드/패턴은 확정값으로 채우고, **실측이 실제로 실패한 값만** `TODO:` 로 남긴다(추정 금지는 유지 — 확실치 않은 커맨드를 지어내면 게이트가 공허 통과/오작동한다). FILE_LINE_LIMIT(공통 800)·FROZEN_DIR(§2 공식 산출)·ARTIFACTS_DIR(기본 `.iterate/`)는 스택 무관이므로 TODO 금지, 줄-상한 GUARD 는 골격 커맨드에서 확장자·스캔 경로만 치환한다. TODO 필드는 §3 생성 후 사용자에게 묶어서 질문.

## 2) 레포 실측으로 값 확정

골격의 예시 값을 그대로 두지 말고, **레포에서 직접 확인**해 교체한다:

- **TEST_CMD / LINT_CMD**: `package.json` 의 `scripts`(test·lint·typecheck), `Makefile` 타깃, `pubspec.yaml`·CI 설정(`.github/workflows/*`)을 grep — 레포가 실제 쓰는 커맨드를 채택한다(예: 레포가 `jest` 면 vitest 예시를 그대로 두지 마라). 채택 전 TEST_CMD 가 **테스트 경로를 trailing 인자로 받고, watch 모드 없이 1회 실행 후 종료하는 형태**인지 확인하라 — Makefile 타깃은 trailing 인자를 별도 타깃으로 해석해 반드시 깨지므로(`make test <경로>` → 'No rule to make target') 기저 러너로 풀어 쓴다(예: `make test` → `go test -race`). npm/pnpm scripts 는 positional 인자가 포워딩되긴 하나, 스크립트가 watch 모드 기본(`vitest`(run 없음)·`react-scripts test` 류 — 게이트가 무한 대기)이거나 경로/플래그를 하드코딩했으면 역시 기저 러너로 풀어 쓴다(예: `npm test` → `npx vitest run`). 템플릿 예시가 전부 기저 러너인 이유가 이 계약이다.
- **소스 루트**: `lib/`·`internal/`·`src/`·`app/` 중 실존 + 코드가 실제 사는 디렉터리를 `ls`/`find` 로 확인해 `PROJECT` 에 기재.
- **TEST_FILE_GLOB**: 기존 테스트 파일의 **실제 명명 패턴**을 grep/find 로 확인(`*_test.dart`? `*.test.ts`? `*.spec.ts`? co-located? 별도 `test/` 트리?). 기존 테스트가 0건이면 스택 관례 패턴을 쓰되 그 사실을 명시.
- **TEST_SUPPORT_GLOBS**: 테스트가 import 하는 헬퍼·픽스처·Fake 의 위치를 실측(`test/helpers/`·`testutil`·`__fixtures__`·`test-utils` 류 grep). 이 패턴은 **동결 스냅샷 범위**에 들어간다 — test-author 가 쓸 수 있는 모든 파일이 대상. 없으면 `(없음)`.
- **MUTATION_EXCLUDES**: 생성 파일 패턴 실측(`*.g.dart`·`*.pb.go`·`*.gen.ts` 류가 레포에 실존하는지 find).
- **FROZEN_DIR**: `$HOME/.iterate-harness/<repo-slug>/frozen_tests` — `repo-slug` 는 `basename "$(git rev-parse --show-toplevel)" | tr -c 'A-Za-z0-9._-' '-'` 로 산출(★ 반드시 레포 밖 — implementer 쓰기 범위 차단).
- **GUARDS / PROJECT_INVARIANTS / TEST_SCOPE_RULES**: 골격 예시에서 출발하되, 스캔 경로를 실존 디렉터리로 교체하고 레포에 안 맞는 가드(예: UI 없는 레포의 색-리터럴 가드)는 제거. 3예시 밖 스택은 골격 불변식(Go 의 `%w` 랩핑 등)을 그대로 두지 말 것: PROJECT_INVARIANTS 는 대상 레포의 CLAUDE.md·CONTRIBUTING·린터/포매터 설정에서 후보 규칙 3~5개를 추출해 "이것을 불변식으로 채택할까요? (추가/삭제 자유)" 선택지 질문으로 확정 — 후보 소스가 없으면 해당 스택 관례 불변식 3개를 '레포에서 미확인' 라벨과 함께 후보로 제시한다. TEST_SCOPE_RULES 는 실측한 테스트 배치(co-located/미러 트리)로 미러 규칙 초안을 직접 쓰고 횡단 계층 디렉터리 목록 1개 질문만 남긴다. 빈칸 주관식 질문은 최후 수단.
- **선택 필드**: 실측 근거가 있을 때만 채운다 — 코드젠 설정 실존→`BUILD_GEN_CMD`, playwright/integration_test 실존→`E2E_CMD`, 디자인 문서 실존→`DESIGN_SSOT`. 근거 없으면 `(없음)`.
- **실측 불가 필드**: 값을 지어내지 말고 `TODO: <무엇을 확인해야 하는지>` 로 쓰고, 생성 후 사용자에게 묶음 질문 1회로 해소한다.

## 3) 어댑터 생성 + gitignore

- `.claude/iterate.config.md` 를 Write — **필드당 코드펜스 블록 정확히 1개**(★ 함정: 템플릿은 예시가 스택당 여럿이라, 여러 블록을 남기면 드라이버가 "첫 코드펜스"만 읽어 엉뚱한 스택 값이 잡힌다. 예시 라벨(**예시 A·B·C**) 줄도 남기지 마라 — 단일-블록 어댑터가 계약이다).
- 파일 서두에 생성 메타 2줄: 생성일·감지 스택·"TODO 필드 N개(사용자 확인 필요)".
- **ARTIFACTS_DIR 을 `.gitignore` 에 추가**: `.gitignore` 에 해당 경로(기본 `.iterate/`)가 없으면 append(산출물이 커밋에 섞이면 안 됨). FROZEN_DIR 는 레포 밖이라 불필요.

## 4) 자가 검증 (생성 직후 당신이 직접)

1. **필수 필드 헤딩 존재**: §0 필수 목록 전부에 대해 `grep -c '^## <FIELD>'` — 누락 시 즉시 보완.
2. **필드당 코드펜스 정확히 1개**: 각 필수 필드 헤딩~다음 헤딩 사이의 ``` 쌍이 정확히 1개인지 확인 — 2개 이상이면 템플릿 통째 복사 상태(드라이버 Step 0a 가 중단시키는 조건)이므로 즉시 단일-블록으로 수정.
3. **TEST_FILE_GLOB 실행 검증**: 글롭을 find 로 변환해 실행(템플릿 각 예시의 find 변환 주석 참조) → **매칭 수를 보고**. 0건이면 경고 — 동결·GUARD 가 공허 통과하게 되므로 패턴 오탈/테스트 부재 중 무엇인지 사용자에게 확인.
4. **GUARD 스캔 경로 실존**: 각 GUARD 커맨드가 참조하는 디렉터리를 `ls`/`test -d` 로 확인 — 부재 경로는 가드가 영원히 공허 통과하므로 실존 경로로 교체하거나 가드 제거.
5. TODO 필드가 남았으면 목록으로 정리해 사용자에게 질문(어댑터는 생성됐지만 `/iterate` 전 TODO 해소 필요라고 명시).
6. **TEST_CMD 스모크(실행 검증)**: TEST_FILE_GLOB 매칭이 1건 이상이면 그중 1개 파일을 인자로 `$TEST_CMD <파일>` 을 timeout 으로 감싸 1회 실행 — exit 코드가 테스트 결과(0 또는 테스트 실패)면 통과. 커맨드 자체 오류(127·'No rule to make target'·'Missing script' 류)나 timeout(watch 모드 의심)이면 TEST_CMD 를 §2 조건대로 교정 후 재검증. TEST_CMD 에 '사전조건:' 줄이 있으면 사용자 확인 후 실행. 매칭 0건이면 스모크 생략(3번 항목 경고로 갈음).

## 5) 종료 보고

```
## 어댑터 생성 완료
**경로**: .claude/iterate.config.md (신규 | 누락 N필드 보강)
**감지 스택**: <A · Flutter | B · Go | C · Web | 기타(<골격> 기반)>
**실측 확정**: TEST_CMD=<...> · LINT_CMD=<...> · TEST_FILE_GLOB 매칭 <N>건 · TEST_SUPPORT_GLOBS=<... | (없음)>
**TODO(사용자 확인 필요)**: <필드 목록 | 없음> — 드라이버는 필드 값에 TODO 가 남아 있으면 Step 0a 에서 거부한다.
**gitignore**: ARTIFACTS_DIR 추가됨 | 이미 있음
다음: TODO 해소 후 `/iterate-harness:iterate "<카드>"` 실행.
```
