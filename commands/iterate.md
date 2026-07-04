---
description: 멀티에이전트 한 사이클 실행. 테스트가 게이트 — test-author 의 테스트가 전부 통과할 때까지 돌고(=iteration), 통과 후 맨 끝에 사람 시각확인 1회. bash 드라이버 없이 메인 Claude 가 구동. 언어/프레임워크 무관 코어 — 프로젝트 특화는 .claude/iterate.config.md 어댑터가 채운다.
argument-hint: [task ID 또는 작업 설명] [N — G 회차 상한, 선택(기본 10)]
---

당신(메인 Claude)이 **드라이버**다. `${CLAUDE_PLUGIN_ROOT}/skills/iterate-protocol/SKILL.md`(SSOT)에 따라 아래 작업의 한 iteration 사이클을 구동한다. **핵심: iteration 은 "리뷰어가 되돌렸나"가 아니라 "독립 테스트가 전부 통과했나(게이트 G)"로 돈다.** 테스트 게이트는 당신이 직접 Bash 로 실행해 최종 판정한다.

**작업**: $ARGUMENTS

## 0) Step 0 — 어댑터 로드 + 카드 해석

### 0a) 프로젝트 어댑터 로드 (필수·먼저)
- **먼저 SSOT 를 Read**: `${CLAUDE_PLUGIN_ROOT}/skills/iterate-protocol/SKILL.md`.
- **그다음 `.claude/iterate.config.md`(프로젝트 어댑터)를 Read.** 없으면 **여기서 중단**하고 다음을 안내한다:
  > "이 레포에 `.claude/iterate.config.md` 어댑터가 없습니다. `${CLAUDE_PLUGIN_ROOT}/examples/iterate.config.md` 를 `.claude/iterate.config.md` 로 복사하고 프로젝트 값(TEST_CMD·소스 루트·GUARDS 등)을 채운 뒤 다시 `/iterate` 하세요."
- 어댑터에서 **모든 게이트 값**을 파싱해 이후 전 단계에서 사용한다(필드당 첫 코드펜스/불릿을 값으로): `PROJECT`·`TEST_CMD`·`LINT_CMD`·`BUILD_GEN_CMD`(선택)·`E2E_CMD`(선택)·`TEST_SCOPE_RULES`·`GUARDS`·`FILE_LINE_LIMIT`·`FROZEN_DIR`·`ARTIFACTS_DIR`·`PLAN_PATH`(선택)·`DESIGN_SSOT`(선택)·`PROJECT_INVARIANTS`·`TEST_FILE_GLOB`·`MUTATION_EXCLUDES`·`HUMAN_GATE`(선택)·`EXPLORE_QA`(선택). 아래 bash 의 `$TEST_CMD`·`$FROZEN`·`$ARTIFACTS` 등은 이 값이다.
- 어댑터의 `ROLE_*` 섹션(선택 — `ROLE_ARCHITECT`·`ROLE_DESIGNER`·`ROLE_TEST_AUTHOR`·`ROLE_TEST_AUDITOR`·`ROLE_IMPLEMENTER`·`ROLE_REVIEWER`·`ROLE_EXPLORER`)은 **드라이버가 파싱하지 않는다** — 각 역할 에이전트가 자기 섹션을 직접 Read 해 추가 지침으로 따른다(SSOT 불변식과 충돌 시 SSOT 우선 — 게이트 약화 불가).
- **경로 준비**: `$ARTIFACTS_DIR` 없으면 `mkdir -p`. `$ARTIFACTS_DIR` 가 gitignore 안 됐으면 사용자에게 1줄 안내(산출물이 커밋에 섞이면 안 됨). `$FROZEN_DIR` 도 `mkdir -p`.

### 0b) 카드 해석
- **인자 파싱**: `$ARGUMENTS` 에서 따옴표 토큰 = ID 또는 작업 설명, **맨 뒤 정수 = N(G 회차 상한, 기본 10)**.
- **`PLAN_PATH` 설정 시**: ID 면 원장에서 카드 블록만 읽는다 — **파일이 크면 통째 Read 금지**: Grep 으로 카드 헤더(`### .* <ID>`) 줄번호를 찾아 그 offset 부터 카드 블록만 Read(드라이버 컨텍스트 보호). 못 찾으면 알리고 중단.
- **`PLAN_PATH` 미설정 시(직접 작업 문자열 모드)**: 인자 문자열 자체가 작업이다(원장 조회·완료기록 갱신 없음 — §7 스킵, §8 보고만).
- **카드 플래그 판정**: `design`(새 시각 표면)·`human-visual`(사람 눈 필요)·`e2e`(헤드리스 미렌더 표면)·`explore`(사용자 흐름이 얽힌 화면·상태 많은 표면 — 명세 밖 탐색 QA 가치). 카드에 명시 없으면 작업 성격으로 판정. 모호하면 사용자 1줄 확인. **`design` 인데 `DESIGN_SSOT` 미설정, `e2e` 인데 `E2E_CMD` 미설정, 또는 `explore` 인데 `EXPLORE_QA` 미설정 → 어댑터 보강을 요청하고 중단.**

## 1) 기계 루프 (사람 없음)
각 Phase 시작·종료를 한 줄로 보고.

**★ 역할 에이전트는 플러그인 네임스페이스**: 이 문서의 `Agent(architect)` 표기는 축약이다 — 실제 `subagent_type` 은 `iterate-harness:architect`·`iterate-harness:designer`·`iterate-harness:test-author`·`iterate-harness:test-auditor`·`iterate-harness:implementer`·`iterate-harness:reviewer`·`iterate-harness:explorer` 다(플러그인 에이전트는 항상 `플러그인명:이름` 으로 노출된다). `Explore` 만 빌트인 그대로다.

**★ 모든 `Agent()` 호출은 포그라운드**: 매번 `run_in_background: false` 를 명시한다(Agent 도구 기본값은 백그라운드다 — 명시 없으면 사용자가 원치 않는 백그라운드로 뜬다). 완료된 에이전트를 `SendMessage` 로 재개하지 마라 — 그 경로는 항상 백그라운드고 포그라운드 옵션이 없다. 반려/바운스(test-author 보강, reviewer 재확인 등)도 매번 **새 `Agent` 호출**로 띄우고, 필요한 맥락은 프롬프트로 전달한다(에이전트가 디스크의 현재 테스트/코드를 직접 Read). `Workflow` 등 백그라운드 도구도 이 하네스에선 금지.

- **0 Explore**: `Agent(Explore)` — 재사용 후보·기존 패턴 탐색. 산출 보존.
- **A architect**: `Agent(architect)` ← 작업 + Explore 결과. 설계+AC+**카드 플래그** 보존.
  - **GATE1(당신 직접)**: 설계에 코드 본문 없음 + estLines 전부 ≤ `$FILE_LINE_LIMIT` 확인. 위반→architect 재호출.
- **designer (design 플래그면)**: `Agent(designer)` ← 작업 + architect 설계 + `$DESIGN_SSOT`. 시각 스펙 산출. **구현 전**에 둬 구조를 한 번에 맞춘다. (design 아니면 생략.) 디자이너 Open Question(새 토큰 등)은 당신이 사용자에 확인.
- **T test-author**: `Agent(test-author)` ← AC + (design면) 시각 스펙. **모든 카드.** AC 를 독립 블랙박스 테스트로(구현 안 봄·명세에서). 부수효과 되읽기·분기 양쪽·불변식 구체값(불변식 11).
  - **소스 루트 불가침 가드(당신 직접)**: 호출 전후로 `git status --porcelain <소스 루트> | shasum` 을 비교 — 달라졌으면 test-author 가 소스 루트를 건드린 것(도구상 Write/Edit 가 소스에도 미침) → 원복 지시 후 재호출.
- **G2 test-auditor (동결 전·독립)**: `Agent(test-auditor)` ← AC + (design면) 시각 스펙 + test-author 가 쓴 테스트 파일(**구현 안 봄**). "틀린 구현이 빠져나갈 변형"을 적대 검사. **escapes ≥1 → `Agent(test-author)` 재호출**(그 변형 잡는 케이스 추가, 최대 2회) → 재검사. **반려 2회 소진 후에도 escapes ≥1 → 중단·사용자 개입**(AC 가 테스트 불가능하게 쓰였을 가능성 — architect 재계획 여부를 사용자와 결정). **escapes 0 → 동결**:
  ```bash
  FROZEN="$FROZEN_DIR"   # ★ 레포 밖 — $ARTIFACTS_DIR 는 implementer(전체 도구) 쓰기 범위라 스냅샷 변조 가능. 반드시 레포 밖에.
  mkdir -p "$FROZEN"
  # 테스트 파일 '목록' 스냅샷 — 몰래 추가/삭제 검출. TEST_FILE_GLOB 을 find 로 열거(zsh ** 확장 함정 회피).
  #   예: TEST_FILE_GLOB='test/**/*_test.dart' → find test -name '*_test.dart'  |  '**/*_test.go' → find . -path ./vendor -prune -o -name '*_test.go' -print
  find <TEST_FILE_GLOB 루트> -name '<name 패턴>' | sort > "$FROZEN/<카드>.filelist"
  # 내용 체크섬 — ★ zsh 단어분리 함정: `shasum $(cat filelist)` 는 zsh 기본 no-split 로 파일명이 한 덩어리가 됨.
  #   반드시 xargs 파이프로(NUL 안전): 아래처럼.
  tr '\n' '\0' < "$FROZEN/<카드>.filelist" | xargs -0 shasum > "$FROZEN/<카드>.sha256"
  ```
  ★ test-author 와 **반드시 별개 호출**(출제자≠검사자 — P1 격리 확장).
- **B implementer**: `Agent(implementer)` ← 설계 + (design면) 시각 스펙 + **동결 테스트 경로** + `failures.md`(있으면). 동결 테스트를 통과시키는 구현. test/ 불가침.
  - `spec_gap.md` 생기면 → architect 재진입. `test_dispute.md` 생기면 → test-author 가 케이스 수정(최대 2회) → **G2 재심사(escapes 0)→재동결** 후 implementer 재개(테스트 변경은 경로 불문 이 관문을 거친다 — SSOT).

## 2) 게이트 G (★ 당신이 직접 Bash — iteration 의 단위)
implementer 산출 후 당신이 직접 실행해 측정한다(모든 커맨드는 어댑터 값):
```bash
ART="$ARTIFACTS_DIR"; FROZEN="$FROZEN_DIR"
: > "$ART/verifier_raw.txt"   # 매 게이트 시작 truncate — 마커 '정확히 1회' 보장(reviewer 는 reviewer_raw.txt 에 별도 기록)
# ── 동결 무결성(implementer 가 test 안 건드렸나) — (1) 파일 목록 동일 (2) 내용 체크섬 동일. 둘 다 G2 escapes-0 시점 스냅샷 대비.
#   ★ 재동결은 매 G2 escapes-0 직후에만(SSOT) → 그 사이(implementer 단계) test 변경만 변조로 본다.
#   ★ zsh 단어분리: 체크섬은 xargs 파이프로(변수 확장 금지).
find <TEST_FILE_GLOB 루트> -name '<name 패턴>' | sort | diff - "$FROZEN/<카드>.filelist" >/dev/null 2>&1; echo "FROZEN_LIST=$?" >> "$ART/verifier_raw.txt"
tr '\n' '\0' < "$FROZEN/<카드>.filelist" | xargs -0 shasum | diff - "$FROZEN/<카드>.sha256" >/dev/null 2>&1; echo "FROZEN_OK=$?" >> "$ART/verifier_raw.txt"
# ── 테스트(영향-스코프 — SSOT §테스트 스코프 / 어댑터 TEST_SCOPE_RULES, 경로 명시)
$TEST_CMD <영향 경로들> > /tmp/g_test.txt 2>&1; echo "TEST_EXIT=$?" >> "$ART/verifier_raw.txt"
$LINT_CMD > /tmp/g_lint.txt 2>&1; echo "ANALYZE_EXIT=$?" >> "$ART/verifier_raw.txt"
# ── 줄-상한 GUARD(모든 소스/테스트 ≤ FILE_LINE_LIMIT, MUTATION_EXCLUDES 제외). 비어야 통과.
find <소스 루트> <TEST_FILE_GLOB 루트> -type f <MUTATION_EXCLUDES 를 ! -name 로> -exec wc -l {} + \
  | awk -v L="$FILE_LINE_LIMIT" '$2!="total" && $1>L {print}'
# ── 어댑터 GUARDS 루프: 각 GUARD 의 커맨드를 실행하고 통과조건(대개 "출력 비어야 함" 또는 "exit 0")을 확인.
#   위반(출력 있음/exit≠0)이면 그 GUARD 이름과 출력을 위반으로 기록. 예시는 어댑터 GUARDS 참조.
#   각 GUARD: <커맨드> → 통과조건 위반이면 GUARD_VIOLATION=<이름> 기록.
```
- **e2e 플래그면** §3 E2E 도 게이트에 포함.
- **게이트는 잴 뿐 — 실패면 파이프라인 전체 재실행.** M개 fail / 위반 / mutation 생존이면 `$ARTIFACTS_DIR/failures.md`(실패 테스트·메시지·생존 변형) 적고 **처음(architect)부터 다시 돌린다**: `Agent(architect)`(실패 맥락 포함) 재계획 → (design면)`Agent(designer)` → `Agent(test-author)`(테스트 추가/수정/유지) → **`Agent(test-auditor)` 재심사(escapes 0)→재동결** → `Agent(implementer)` → **게이트**. ★ 재실행에서도 **G2·재동결 생략 금지**(끼워맞추기 백스톱). 콕 집어 한 에이전트만 부르지 말 것 — 재계획은 architect 가 하고 아래가 그대로 흐른다.
  - **입력 안 바뀐 단계는 산출 그대로**(불필요 재생성·테스트 churn 금지): architect 계획 동일 → designer/test-author 동일 산출. Explore 맵은 코드베이스 그대로면 재사용. 단순 구현 버그면 architect 가 계획 유지 + 노트로 버그 위치 짚고 implementer 재구현.
  - → 한 바퀴 = 다음 회차. 0 fail 까지, 최대 N회.
- **동결 불일치(FROZEN_LIST!=0 또는 FROZEN_OK!=0, implementer 단계)** → implementer 가 test 변조(수정·추가·삭제 모두) → 즉시 미통과, "test/ 복원·소스로 통과" 지시. (재동결은 매 G2 통과 직후만 갱신되므로, test-author 의 정당 추가는 G2 후 스냅샷에 반영돼 불일치가 아니다 — implementer 구간 고정 스냅샷 대비만 본다.)

## 2b) Mutation 게이트 (test 0 fail 후 — 끼워맞추기 차단, 당신 직접)
정답지(동결 테스트)를 보고 구현한 게 *진짜 맞는지* 모델 판단 없이 실증한다 — **변경/생성된 impl 에 미묘한 변형을 심어 영향-스코프 테스트가 잡는지** 본다.
- 대상: 이 카드가 바꾼 **손으로 쓴** impl 파일만(전 코드베이스 아님·**`MUTATION_EXCLUDES` 제외**). G2 test-auditor 가 지목한 고가치 지점 우선. 파일당 ~2~4개.
- 변형 예: `<`↔`<=`·off-by-one·가드 1줄 제거·보존 필드 `→ null`·상태변이 no-op·분기 조건 반전·필터 술어 제거.
- 절차(★ **기본 = 격리 사본** — 메인 작업트리는 미커밋 WIP 를 담고 있어, 변형 주입~원복 사이에 세션 크래시/컨텍스트 압축이 끼면 사용자 코드가 오염된 채 남는다):
  ```bash
  MUT="<세션 scratchpad>/mutation_copy"
  rsync -a --delete --exclude .git --exclude build --exclude .dart_tool --exclude node_modules ./ "$MUT"/
  (cd "$MUT" && ${BUILD_GEN_CMD:+eval "$BUILD_GEN_CMD"} )   # 경로 종속/생성물 복원(있으면). 없으면 생략.
  ```
  사본 안에서: 변형 Edit → 사본의 해당 미러 test 실행(`$TEST_CMD`) → fail=kill / green=생존 → `cp <원본 파일> <사본 경로>` 로 파일 리셋 → 다음 변형. **메인 작업트리는 어떤 경우에도 수정하지 않는다.** (주의: `git worktree add … HEAD` 는 미커밋 구현이 빠져 무의미 — 쓰지 말 것.) rsync 가 불가한 환경에서만 예외적으로 메인 트리 정확-바이트 save/restore 를 쓰되, 주입 전 `cp <파일> <scratchpad>/mut_backup/` 영구 백업을 먼저 남긴다(크래시 복구 지점). `git checkout/stash/clean` 금지.
- **★ INVALID 변형 처리(중요)**: 변형이 **컴파일/로드 실패**를 내면 그건 **kill 이 아니라 무효(INVALID)** 다 — 컴파일 단계에서 전부 실패해 "잡힌 것처럼" 보여도 테스트 엄격성을 증명하지 못한다. INVALID 는 **세지 말고 폐기**하고, **컴파일은 되지만 동작이 달라지는 변형**으로 다시 주입한다(타입-보존: 가드 제거·경계 이동·no-op 화·분기 반전 등). kill 은 *컴파일되는 변형을 테스트 단정이 fail 시킬 때*만 인정.
- `MUT_SURVIVED=<n>` 마커를 `$ARTIFACTS_DIR/verifier_raw.txt` 에 기록(INVALID 는 분모에서 제외).
- **생존 ≥1 → G 미통과**: 생존 변형을 `failures.md` 에 적고 → architect 부터 파이프라인 재실행(test-author 가 그 변형 잡는 케이스 추가) [iteration +1]. **생존 0 이어야** reviewer 로.

## 3) E2E (e2e 플래그일 때, 당신 직접 — 게이트 일부)
```bash
ART="$ARTIFACTS_DIR"
# E2E_CMD 는 어댑터가 준 실행(시뮬레이터/브라우저/헤드리스 드라이버). ★ timeout 래퍼 필수(hang 결정적 차단).
${TIMEOUT:-$(command -v timeout||command -v gtimeout)} 300 $E2E_CMD; echo "E2E_EXIT=$?" >> "$ART/verifier_raw.txt"
```
- `E2E_EXIT=124`=hang, `!=0`=실패 → G 미통과(회차 계속).
- **timeout 바이너리 부재 시**: `timeout`/`gtimeout` 둘 다 없으면(macOS 기본 상태) **무-timeout 실행 금지** — 사용자에게 `brew install coreutils`(gtimeout) 설치를 요청하고 E2E 를 중단한다(hang 차단 장치 없이 돌리지 않는다).
- **환경 미기동 = 통과 아님**: 실행 환경(시뮬레이터/브라우저)이 없으면 당신이 직접 기동 시도(어댑터 `HUMAN_GATE`/`E2E_CMD` 주석의 기동법). 그래도 불가면 **자동 green 선언 금지·중단하고 사용자에게 환경 기동을 요청**한다(e2e 플래그 카드에서 E2E 생략 green 은 human-visual 플래그가 없으면 무보완 구멍).

## 4) reviewer (게이트 0 fail 후 — 적대적 갭 사냥)
- 호출 전 `: > "$ARTIFACTS_DIR/reviewer_raw.txt"` truncate(이전 카드/회차 마커 잔존 방지) + `git status --porcelain | shasum` 기록.
- **게이트가 0 fail 이 된 뒤** `Agent(reviewer)` 호출 ← 설계 + 변경 + "테스트가 *놓친* 결함을 사냥하라(도장 금지)". reviewer 는 accept/reject 가 아니라 **갭 목록**을 낸다.
- 호출 후 `git status --porcelain | shasum` 재비교 — 달라졌으면 reviewer 가 파일을 수정한 것(Bash 보유라 도구로 못 막음 — 수정 금지는 지시문) → 원복 지시. (reviewer_raw.txt 기록은 예외라 shasum 대상에서 제외: `git status` 는 gitignore 된 `$ARTIFACTS_DIR` 를 안 세므로 자연 제외된다.)
- **갭 ≥1** → 종류별로: 데모/제품을 깨는 결함 → 그 케이스를 test-author 에 추가시킴 → **G2 재심사(escapes 0)→재동결** → **게이트 재진입**(fail→fix 루프) / 중복·죽은코드 → implementer 가 정리 → **게이트 재진입**. 비차단 잠복은 Notes 로만 기록. (테스트 변경은 경로 불문 G2→재동결을 거친다 — 이 관문 없이 스냅샷 갱신 금지.)
- **갭 0** → (explore 플래그면 §4b 로 / 아니면) 자동 green 후보. (간결화는 별도 polish 단계가 아니라 reviewer 가 잡으면 implementer 가 고치는 일반 수정.)

## 4b) explorer (explore 플래그 카드만 — reviewer 갭 0 후·자동 green 확정 직전)
- 전제: 어댑터 `EXPLORE_QA` 설정(미설정이면 Step 0b 에서 이미 차단). 호출 전 `git status --porcelain | shasum` 기록 + `mkdir -p "$ARTIFACTS_DIR/explore"`.
- `Agent(explorer)` 호출 ← 카드/AC 요약 + 변경 파일 목록 + 어댑터 `EXPLORE_QA`(기동법·조작 도구·초점) + **시간상자**(기본 탐색 흐름 5~8개). explorer 는 명세 밖(인접 흐름·빈/대량 데이터·중단·역행·연타·경계 밖 입력)을 실물로 구동하고, 일회성 탐색 스크립트·증거는 `$ARTIFACTS_DIR/explore/` 에만 쓴다(동결 glob 밖 — test/·소스 루트 금지).
- 호출 후 shasum 재비교 — 달라졌으면 explorer 가 제품/테스트를 수정한 것 → 원복 지시(ARTIFACTS_DIR 는 gitignore 라 자연 제외).
- **실행 환경 미기동 = 통과 아님**: explorer 가 기동 불가를 보고하면 당신이 기동을 시도하고, 그래도 불가면 **탐색 생략 green 선언 금지·중단하고 사용자에게 환경 기동 요청**(e2e 와 동일 원칙).
- **결함 ≥1** → 기계로 환원 가능한 결함은 재현 경로를 test-author 에 케이스로 추가시킴 → **G2 재심사(escapes 0)→재동결** → **게이트 재진입**(fail→fix 루프, 회차 계속). 환원 불가(순수 미감·실기기 한정)는 human-visual 게이트 몫으로 Notes 분리.
- **결함 0** → 자동 green 후보.
- 재실행 회차에서: 직전 explore 이후 변경이 탐색 표면과 무관(순수 내부 로직 수정)하면 직전 탐색 결과를 재사용할 수 있다(churn 금지) — 표면이 바뀌었으면 재탐색.

## 5) 자동 green 판정 (당신의 AND-gate)
```
동결 전제 = G2 test-auditor escapes 0 (얕으면 동결 안 함 — test-author 반려)

자동 green = TEST_EXIT==0 (마커 줄시작 정확히 1회)
            AND FROZEN_LIST==0 AND FROZEN_OK==0 (동결 파일목록·체크섬 일치)
            AND ANALYZE_EXIT==0 (LINT 무경고)
            AND (e2e 없음 OR E2E_EXIT==0)
            AND 줄-상한 위반 0 AND 어댑터 GUARDS 위반 0
            AND MUT_SURVIVED==0 (Mutation 게이트 생존 0, INVALID 제외)
            AND reviewer 갭 0
            AND (explore 없음 OR explorer 결함 0)
```
자동 green 까지 루프, **MAX_ITERS = N(기본 10) 하드 상한**. 초과·같은 fail 2회 연속 동일원인 미해결 시 중단·사용자 개입.

## 6) 사람 시각확인 게이트 (human-visual 플래그면 — 맨 끝·1회)
- **자동 green 확정 후에만**, 당신이 어댑터 `HUMAN_GATE` 방법(시뮬레이터/브라우저 스크린샷·CLI 출력 등)으로 직접 한 번 띄워(셀프검증) → 사용자에게 "이걸 확인해 달라"고 캡처/출력과 함께 제시.
- **골든/스냅샷 기준**(어댑터 HUMAN_GATE 골든 규약 있을 때): 기준 산출물은 **사람 승인 후에만** 생성한다(드라이버 임의 갱신 금지 — 잘못된 기준 박제 차단). 기계 게이트 스코프에서 제외.
- 사용자가 "투박/틀림" → 그 지적을 **새 라운드**로(가능하면 test-author 새 케이스나 architect 재계획으로 환원). 매 회차마다 사람을 부르지 마라.
- human-visual 없는 순수 로직 카드 → 자동 green = done.

## 7) 완료기록 갱신 (`PLAN_PATH` 설정 시만)
`$PLAN_PATH` 카드 완료기록: 상태·**G 회차(테스트 N개·1회차 M fail→…→0 fail 추이)**·날짜·플래그·특이사항.
- **human-visual 카드**: 자동 green = **'verification passed'**, `실기 시각확인 날짜: –`. 사람 OK 후 채워야 **'done'**.
- 그 외: 자동 green = done.
- `PLAN_PATH` 미설정(직접 작업 문자열 모드)이면 원장 갱신 생략 — §8 종료 보고만.

## 8) 종료 보고
```
## Iteration 완료
**카드/작업**: <ID 또는 설명> · **플래그**: [design/human-visual/e2e/explore] · **회차**: X (테스트 N개, 1회차 M fail → 0 fail)
**변경 파일**: …
**게이트**: 테스트 N/N · LINT 무경고 · 줄상한·GUARDS OK · E2E(해당 시) K/K · mutation 생존 0 · reviewer 갭 0 · explore(해당 시) 결함 0
**explorer 발견→테스트화**: <있었으면 무엇> | 없음 | (explore 아님)
**reviewer 발견→테스트화**: <있었으면 무엇> | 없음
**완료 상태**: done | verification passed (실기 시각확인 대기)
**특이사항**: 1~2줄(잠복/후속 Note 포함)
```
transient(Agent 크래시)는 해당 역할 1~2회 재호출. spec_gap/test_dispute/MAX_ITERS/동일-fail-2회 중단은 사유와 함께 사용자에게 보고.
