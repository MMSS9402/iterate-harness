---
description: 멀티에이전트 한 사이클 실행. 테스트가 게이트 — test-author 의 테스트가 전부 통과할 때까지 돌고(=iteration), 통과 후 맨 끝에 사람 시각확인 1회. bash 드라이버 없이 메인 Claude 가 구동. 언어/프레임워크 무관 코어 — 프로젝트 특화는 .claude/iterate.config.md 어댑터가 채운다.
argument-hint: [task ID 또는 작업 설명] [N — G 회차 상한, 선택(기본 10)] [deep — 완전 심사 모드, 선택(기본 lean)]
---

당신(메인 Claude)이 **드라이버**다. `${CLAUDE_PLUGIN_ROOT}/skills/iterate-protocol/SKILL.md`(SSOT)에 따라 아래 작업의 한 iteration 사이클을 구동한다. **핵심: iteration 은 "리뷰어가 되돌렸나"가 아니라 "독립 테스트가 전부 통과했나(게이트 G)"로 돈다.** 테스트 게이트는 당신이 직접 Bash 로 실행해 최종 판정한다.

**작업**: $ARGUMENTS

## 0) Step 0 — 어댑터 로드 + 카드 해석

### 0a) 프로젝트 어댑터 로드 (필수·먼저)
- **먼저 SSOT 를 Read**: `${CLAUDE_PLUGIN_ROOT}/skills/iterate-protocol/SKILL.md`.
- **그다음 `.claude/iterate.config.md`(프로젝트 어댑터)를 Read.** 없으면 **여기서 중단**하고 다음을 안내한다:
  > "이 레포에 `.claude/iterate.config.md` 어댑터가 없습니다. `/iterate-harness:iterate-init` 를 먼저 실행해 어댑터를 생성하세요(또는 `${CLAUDE_PLUGIN_ROOT}/examples/iterate.config.md` 를 `.claude/iterate.config.md` 로 수동 복사해 프로젝트 값을 채운 뒤) — 그다음 다시 `/iterate` 하세요."
- 어댑터에서 **모든 게이트 값**을 파싱해 이후 전 단계에서 사용한다(필드당 첫 코드펜스/불릿을 값으로 — **선택 필드는 코드펜스 부재·빈 값·`(없음)` 모두 미설정으로 취급**): `PROJECT`·`TEST_CMD`·`LINT_CMD`·`BUILD_GEN_CMD`(선택)·`E2E_CMD`(선택)·`TEST_SCOPE_RULES`·`GUARDS`·`FILE_LINE_LIMIT`·`FROZEN_DIR`·`ARTIFACTS_DIR`·`PLAN_PATH`(선택)·`DESIGN_SSOT`(선택)·`PROJECT_INVARIANTS`·`TEST_FILE_GLOB`·`TEST_SUPPORT_GLOBS`·`MUTATION_EXCLUDES`·`HUMAN_GATE`(선택)·`EXPLORE_QA`(선택). 아래 bash 의 `$TEST_CMD`·`$FROZEN`·`$ARTIFACTS` 등은 이 값이다.
- **어댑터 검증(파싱 직후·당신 직접)**: (1) 어느 필드든 값 코드펜스가 **2개 이상**이면 예시 템플릿을 통째 복사한 것으로 간주 — 중단하고 "필드당 자기 스택 블록 하나만 남기라" 안내. (2) `TEST_FILE_GLOB` 을 find 로 열거해 **매칭 0건**이면 게이트 진입 전 중단(글롭이 틀리면 동결·GUARD 가 빈 집합으로 공허 통과한다). (3) Step 0a 에서 파싱한 각 필드의 **값**(필드당 첫 코드펜스/불릿 내용, GUARDS 는 각 GUARD 커맨드 줄)에 `TODO:` 가 하나라도 있으면 게이트 진입 전 중단 — 해당 필드명 목록을 보여주고 "값 확정 후 다시 `/iterate` 하거나 `/iterate-harness:iterate-init` 재실행" 안내(미확정 커맨드가 게이트에서 exit 127 을 내면 테스트 실패로 오귀속된다). ★ 검사 범위는 **파싱된 필드 값으로 한정 — 파일 전체 grep 금지**(GUARDS 가 소스의 TODO 주석을 스캔하는 가드일 수 있고, PROJECT_INVARIANTS 산문의 TODO 언급이 정당할 수 있어 오탐한다). (4) `GUARDS` 에 `FILE_LINE_LIMIT` 를 참조하는 줄-상한 가드가 없으면 게이트 진입 전 중단·어댑터 보강 요청(줄-상한 판정은 어댑터 GUARDS 로 일원화 — §2 참조).
- 어댑터의 `ROLE_*` 섹션(선택 — `ROLE_ARCHITECT`·`ROLE_DESIGNER`·`ROLE_TEST_AUTHOR`·`ROLE_TEST_AUDITOR`·`ROLE_IMPLEMENTER`·`ROLE_REVIEWER`·`ROLE_EXPLORER`)은 **드라이버가 파싱하지 않는다** — 각 역할 에이전트가 자기 섹션을 직접 Read 해 추가 지침으로 따른다(SSOT 불변식과 충돌 시 SSOT 우선 — 게이트 약화 불가).
- **경로 준비**: `$ARTIFACTS_DIR` 없으면 `mkdir -p`. `$ARTIFACTS_DIR` 가 gitignore 안 됐으면 사용자에게 1줄 안내(산출물이 커밋에 섞이면 안 됨). `$FROZEN_DIR` 도 `mkdir -p`.
- **잔존 트리거 소거(per-카드 초기화)**: `rm -f "$ARTIFACTS_DIR"/{failures,spec_gap,test_dispute}.md` — escape 파일은 **존재 자체가 트리거**라, 이전 카드 잔존분이 남아 있으면 오발동한다(verifier_raw/reviewer_raw truncate 와 같은 원리). **`violations.md` 는 rm 대상에서 제외** — 카드 간 누적이 목적인 감사 기록이다.
- **세션 모델 자기점검(1줄)**: 세션 모델이 opus 계열이 아니면 "GATE1 판정·mutation 변형 설계(특히 INVALID 재설계)·동형 확장 프롬프트 품질은 드라이버=세션 모델을 따른다 — opus 세션 권장" 을 경고하고 계속 여부를 사용자에게 1줄 확인한다(게이트 변경 아님 — 진행 자체는 허용).

### 0b) 카드 해석
- **인자 파싱**: `$ARGUMENTS` 에서 따옴표 토큰 = ID 또는 작업 설명, **맨 뒤 정수 = N(G 회차 상한, 기본 10)**, **`deep` 토큰 = deep 모드**(없으면 **lean 기본** — SSOT §실행 모드: G2 델타 심사·reviewer 1패스·explorer 큐·designer 캐시. 기계 게이트는 모드 무관 동일). 모드를 회차 보고 첫 줄에 명시한다.
- **`PLAN_PATH` 설정 시**: ID 면 원장에서 카드 블록만 읽는다 — **파일이 크면 통째 Read 금지**: Grep 으로 카드 헤더(`### .* <ID>`) 줄번호를 찾아 그 offset 부터 카드 블록만 Read(드라이버 컨텍스트 보호). 못 찾으면 알리고 중단.
- **`PLAN_PATH` 미설정 시(직접 작업 문자열 모드)**: 인자 문자열 자체가 작업이다(원장 조회·완료기록 갱신 없음 — §7 스킵, §8 보고만).
- **카드 플래그 판정**: `design`(새 시각 표면)·`human-visual`(사람 눈 필요)·`e2e`(헤드리스 미렌더 표면)·`explore`(사용자 흐름이 얽힌 화면·상태 많은 표면 — 명세 밖 탐색 QA 가치). 카드에 명시 없으면 작업 성격으로 판정. 모호하면 사용자 1줄 확인. **`design`↔`DESIGN_SSOT`·`human-visual`↔`HUMAN_GATE`·`e2e`↔`E2E_CMD`·`explore`↔`EXPLORE_QA` — 플래그인데 대응 필드 미설정 → 어댑터 보강을 요청하고 중단.** e2e 플래그면 추가로 `E2E_CMD` 가 실행하는 테스트 경로(어댑터 주석·커맨드 인자로 확인)가 동결 열거(`TEST_FILE_GLOB` ∪ `TEST_SUPPORT_GLOBS` 의 find 절)에 덮이는지 확인 — 안 덮이면 implementer 의 e2e 테스트 변조를 동결 체크섬·가드가 못 잡으므로 어댑터 보강 요청 후 중단.

## 1) 기계 루프 (사람 없음)
각 Phase 시작·종료를 한 줄로 보고.

**★ 역할 에이전트는 플러그인 네임스페이스**: 이 문서의 `Agent(architect)` 표기는 축약이다 — 실제 `subagent_type` 은 `iterate-harness:architect`·`iterate-harness:designer`·`iterate-harness:test-author`·`iterate-harness:test-auditor`·`iterate-harness:implementer`·`iterate-harness:reviewer`·`iterate-harness:explorer` 다(플러그인 에이전트는 항상 `플러그인명:이름` 으로 노출된다). `Explore` 만 빌트인 그대로다.

**★ 모든 `Agent()` 호출은 포그라운드**: 매번 `run_in_background: false` 를 명시한다(Agent 도구 기본값은 백그라운드다 — 명시 없으면 사용자가 원치 않는 백그라운드로 뜬다). 완료된 에이전트를 `SendMessage` 로 재개하지 마라 — 그 경로는 항상 백그라운드고 포그라운드 옵션이 없다. 반려/바운스(test-author 보강, reviewer 재확인 등)도 매번 **새 `Agent` 호출**로 띄우고, 필요한 맥락은 프롬프트로 전달한다(에이전트가 디스크의 현재 테스트/코드를 직접 Read). 검사자(test-auditor·reviewer·explorer) 산출물을 바운스로 전달할 때는 **발견 항목**(Escapes/갭/결함의 변형 설명·재현 경로)만 넘기고, Verdict·Covered·체크리스트 등 판정 서사와 피검사자의 수정 설명은 전달하지 않는다(spec_gap/test_dispute 보고 전달은 escape 규정대로). `Workflow` 등 백그라운드 도구도 **이 사이클 안에선** 금지(예외는 `/iterate-qa` 배치 한정 — SSOT §에이전트 독립성).

**★ 전후 다이제스트 가드 공통(위반 기록)**: 아래 4곳 가드(§1 test-author 소스 루트·§1 implementer 지원파일·§4 reviewer·§4b explorer)에서 호출 '전' 다이제스트 계산 시 내부 커맨드 원시 출력을 `tee "$ARTIFACTS_DIR/.guard_pre.txt"` 로도 저장한다(gitignore 된 ARTIFACTS_DIR 라 다이제스트에서 자연 제외 — happy path 비용은 tee 1회). 불일치 시 내부 커맨드를 재실행해 `.guard_pre.txt` 와 diff 한 것(위반분만 분리)을 상한 200줄로 `$ARTIFACTS_DIR/violations.md` 에 `[회차 N·역할 R·검출 지점]` 헤더로 append 한 뒤 원복 지시한다. violations.md 는 파일로만 쌓고 당신 응답엔 1줄 요약만.

- **0 Explore**: `Agent(Explore)` — 재사용 후보·기존 패턴 탐색. 산출 보존. (Explore 는 빌트인이라 **세션 모델·effort 를 상속**한다 — 호출 시 `model: "sonnet"` 을 명시하고(맵핑은 게이트 심사가 아니다 — opus 불요) 프롬프트에 카드 관련 모듈로 범위를 좁혀 명시.)
- **A architect**: `Agent(architect)` ← 작업 + Explore 결과. 설계+AC+**카드 플래그** 보존. architect Card flags 가 0b 잠정 판정과 다르면 architect 판정으로 갱신한다 — 단 (i) **플래그 제거 방향**(0b 또는 원장에서 판정된 design/human-visual/e2e/explore 를 끄는 쪽)이면 사용자 1줄 확인 필수(원장 명시 플래그는 architect 가 제거 불가), (ii) **플래그 추가 방향**이면 0b 의 어댑터 필드 검사(design↔DESIGN_SSOT·human-visual↔HUMAN_GATE·e2e↔E2E_CMD·explore↔EXPLORE_QA)를 그 시점에 재실행 — 미설정이면 어댑터 보강 요청 후 중단(0b 와 동일 규칙). 산출은 당신이 `$ARTIFACTS_DIR/plan.md` 로 Write 한다(직전 회차분이 있으면 먼저 `plan.prev.md` 로 이동 — 재실행 시 diff 기계 판정용, §2 참조).
  - **GATE1(당신 직접)**: 설계에 코드 본문 없음 + estLines 전부 ≤ `$FILE_LINE_LIMIT` 확인. 위반→architect 재호출. plan.md Open questions 에 `[blocking]` 항목이 있으면 진행 중단, 사용자에게 각 1줄 확인 → 답을 architect 재호출 프롬프트로 전달해 AC 에 반영시킨 뒤 진행(non-blocking 은 기록만 하고 통과).
  - **spec.md 발췌(GATE1 통과 직후·당신 직접)**: plan.md 에서 **Card flags·Interface·Acceptance Criteria·Visual 섹션만 발췌**해 `$ARTIFACTS_DIR/spec.md` 로 Write 한다(출제자·검사자 전달용 — Files/Reuse/Rationale/Notes 비포함). plan.md 를 새로 Write 한 회차마다 재발췌한다(Notes-only diff 회차는 AC·Interface 불변이므로 기존 spec.md 유효).
- **designer (design 플래그면)**: **★캐시 판정 먼저(당신 직접·SSOT §카드 플래그 design)**: `$ARTIFACTS_DIR/design_cache.md` 에서 대상 화면군의 visual_spec 항목과 그 산출 시점 `DESIGN_SSOT` 다이제스트를 읽고, `shasum <DESIGN_SSOT 경로>` 현재값과 대조 — **일치 + 이번 카드가 기존 화면군의 확장(신규 시각 표면 아님)이면 designer 스킵·기존 `visual_spec.md` 재사용**(보고 1줄). 캐시 미스(신규 표면·SSOT 변경·스펙 부재)면 `Agent(designer)` ← 작업 + architect 설계 + `$DESIGN_SSOT`. 시각 스펙 산출 — 당신이 `$ARTIFACTS_DIR/visual_spec.md` 로 Write 하고 `design_cache.md` 에 `[화면군] visual_spec 경로 · DESIGN_SSOT shasum` 1줄 append. **구현 전**에 둬 구조를 한 번에 맞춘다. (design 아니면 생략.) 디자이너 Open Question(새 토큰 등)은 당신이 사용자에 확인. 스펙만 갱신하고 싶으면 사이클 밖 `/iterate-harness:iterate-design` 을 안내.
- **T test-author**: `Agent(test-author)` ← `$ARTIFACTS_DIR/spec.md` 경로(드라이버 발췌본 — **plan.md 원문 경로는 주지 않는다**) + (design면) `visual_spec.md` 경로. **모든 카드.** AC 를 독립 블랙박스 테스트로(구현 안 봄·명세에서). 부수효과 되읽기·분기 양쪽·불변식 구체값(불변식 11). (e2e 카드) E2E 시나리오 테스트도 test-author 산출물이다 — AC 에서 도출하며, 동결 글롭 안 경로에만 작성.
  - **★증분 재호출(v0.9 — 환원·반려·보강 회차)**: 프롬프트에 **대상 파일 목록 + 대상 AC(신규/변경분만)** 를 명시하고 "대상 밖 동결 테스트 Read 금지(확인은 Glob·Grep)"를 포함한다 — 스코프 없는 재호출은 전체 재열람이 되어 최초 작성과 같은 비용이 든다. 같은 라운드의 결함(mutation 생존·reviewer 갭·explorer 결함)은 **한 번의 호출로 배치**한다(결함 1건당 별도 호출 금지 — SSOT §게이트 실패 0).
  - **소스 루트 불가침 가드(당신 직접)**: 호출 전후로 `{ git status --porcelain <소스 루트>; git diff HEAD -- <소스 루트>; git ls-files -o --exclude-standard -z <소스 루트> | xargs -0 shasum; } | shasum` 을 비교 — 달라졌으면 test-author 가 소스 루트를 건드린 것(도구상 Write/Edit 가 소스에도 미침) → 원복 지시 후 재호출. (★ `git status --porcelain | shasum` 단독은 상태 전용이라 이미-dirty/untracked 파일의 **내용** 변조를 못 잡는다 — diff+untracked 내용 해시까지 포함. `--exclude-standard` 로 gitignore 된 `$ARTIFACTS_DIR` 는 자연 제외.)
- **G2 test-auditor (동결 전·독립)**: `Agent(test-auditor)` ← `$ARTIFACTS_DIR/spec.md` 경로(발췌본 — plan.md 원문 비전달) + (design면) `visual_spec.md` 경로 + test-author 가 쓴 테스트 파일(**구현 안 봄**). "틀린 구현이 빠져나갈 변형"을 적대 검사.
  - **★델타 심사(lean·재심사 한정 — SSOT §GATE2)**: 카드 **최초 G2 는 전체 심사**. 재심사(환원·반려 보강·mutation 생존 추가)면 당신이 직전 동결 `.sha256` 과 현재 파일을 `shasum` 대조해 **변경·신규 테스트 파일 목록**을 산출하고, test-auditor 프롬프트에 그 목록 + "심사 스코프 = 이 파일들 전체 + 해당 AC + 기존 커버리지와의 상호작용(목록·Grep 으로만) + 직전 Escapes 회귀 부록. **체크섬 불변 파일의 직전 escapes-0 판정은 유효 — 재판정 불요·Read 금지(v0.9)**" 를 명시한다. deep 모드는 매번 전체 재심사(기존 규정 그대로).
  - **형식검사(당신 직접)**: 산출물에 `## Mutant checklist` 섹션 부재, 또는 Escapes/Covered 항목의 checklist ID 참조·'후행 발견' 표기 누락 시 test-auditor 재호출(GATE1·마커 형식검사와 동일 관행).
  - **`## AC defects` ≥1 → test-author 반려가 아니라 architect 재계획으로 라우팅**(designer→test-author→G2 전체 재흐름, 반려 상한 2회에 합산 — 소진 시 기존대로 중단·사용자 개입).
  - 판정 수신 직후 당신이 `$ARTIFACTS_DIR/g2_verdicts.md` 에 `## [<카드 ID/작업 슬러그> · 회차 N · G2 라운드 R]` 헤더로 산출물에서 **`## Mutation targets` 를 제외한** 전문(Mutant checklist/Escapes/Covered/AC defects/Out-of-scope/Verdict)을 append 한다(반려 재검사 포함 매 호출). Mutation targets 는 ARTIFACTS_DIR 에 쓰지 않고 동결 다이제스트와 같은 방식으로 **당신의 응답 텍스트(대화 컨텍스트)에만 기록**한다 — implementer 는 대화 컨텍스트를 읽지 못한다(§2b 기밀 규칙).
  - **escapes ≥1 → `Agent(test-author)` 재호출**(그 변형 잡는 케이스 추가, 최대 2회) → 재검사(입력은 첫 호출과 동일: spec.md 경로 + 현재 테스트 파일 **전체**. 직전 Verdict·Covered·'추가/보강할 케이스' 제안·test-author 의 수정 설명은 전달 금지 — 재검사는 fix 확인이 아니라 **전체 재심사**다. 단 직전 Escapes 의 변형 설명(각 항목의 AC#·빠져나갈 구체 변형 두 필드만)은 "전체 재심사가 우선, 이 목록은 회귀 확인용 부록" 프레이밍으로 덧붙일 수 있다 — 알려진 구멍이 부실 보강으로 동결되는 것 차단). **반려 2회 소진 후에도 escapes ≥1 → 중단·사용자 개입**(AC 가 테스트 불가능하게 쓰였을 가능성 — architect 재계획 여부를 사용자와 결정). **escapes 0 → 동결**:
  ```bash
  FROZEN="$FROZEN_DIR"   # ★ 레포 밖 — $ARTIFACTS_DIR 는 implementer(전체 도구) 쓰기 범위라 스냅샷 변조 가능. 반드시 레포 밖에.
  mkdir -p "$FROZEN"
  # 테스트+지원 파일 '목록' 스냅샷 — 몰래 추가/삭제 검출. TEST_FILE_GLOB ∪ TEST_SUPPORT_GLOBS(헬퍼·픽스처) 합집합을 find 로 열거(zsh ** 확장 함정 회피).
  #   예: TEST_FILE_GLOB='test/**/*_test.dart' → find test -name '*_test.dart'  |  '**/*_test.go' → find . -path ./vendor -prune -o -name '*_test.go' -print
  #   TEST_SUPPORT_GLOBS 의 각 글롭도 같은 방식으로 find 절 하나씩 — '(없음)' 이면 그 절은 생략.
  { find <TEST_FILE_GLOB find 절>; find <TEST_SUPPORT_GLOBS find 절들>; } | sort -u > "$FROZEN/<카드>.filelist"
  # 내용 체크섬 — ★ zsh 단어분리 함정: `shasum $(cat filelist)` 는 zsh 기본 no-split 로 파일명이 한 덩어리가 됨.
  #   반드시 xargs 파이프로(NUL 안전): 아래처럼.
  tr '\n' '\0' < "$FROZEN/<카드>.filelist" | xargs -0 shasum > "$FROZEN/<카드>.sha256"
  # 동결 다이제스트 자체를 고정 — ★ 디스크는 $FROZEN 포함 전부 implementer(전체 도구) 가 쓸 수 있어 파일 백스톱만으로는 부족.
  shasum "$FROZEN/<카드>.filelist" "$FROZEN/<카드>.sha256"
  ```
  ★ 마지막 `shasum` 두 줄 출력을 **당신의 응답 텍스트(대화 컨텍스트)에 그대로 기록**한다 — implementer 는 대화 컨텍스트를 변조할 수 없다. §2 첫 검사에서 같은 커맨드를 재실행해 이 기록값과 대조한다. 정당한 재동결은 항상 이 G2 블록 재실행이므로 기록값도 그때 자동 갱신 — 거짓 양성 없음. 같은 shasum 두 줄을 `iteration_log.md` append 에도 포함한다 — **감사 사본이며 저신뢰**(ARTIFACTS_DIR 는 implementer 쓰기 범위), 정본은 여전히 대화 컨텍스트 기록.
  ★ test-author 와 **반드시 별개 호출**(출제자≠검사자 — P1 격리 확장).
- **B implementer**: `Agent(implementer)` ← `$ARTIFACTS_DIR/plan.md` 경로(설계 인라인 전달 대신) + (design면) `visual_spec.md` 경로 + **동결 테스트 경로** + `failures.md`(있으면). 동결 테스트를 통과시키는 구현. test/ 불가침.
  - **지원 파일 불가침 가드(당신 직접 — 동결 체크섬과 이중 방어)**: test-author 가드와 동형으로, 호출 전후 `{ git status --porcelain <TEST_SUPPORT_GLOBS 경로>; git diff HEAD -- <TEST_SUPPORT_GLOBS 경로>; git ls-files -o --exclude-standard -z <TEST_SUPPORT_GLOBS 경로> | xargs -0 shasum; } | shasum` 을 비교 — 달라졌으면 implementer 가 헬퍼·픽스처를 건드린 것 → 원복 지시.
  - `spec_gap.md` 생기면 → architect 재진입. `test_dispute.md` 생기면 → test-author 가 케이스 수정(최대 2회) → **G2 재심사(escapes 0)→재동결** 후 implementer 재개(테스트 변경은 경로 불문 이 관문을 거친다 — SSOT).
  - **★ escape 파일 소비 규칙**: `spec_gap.md`/`test_dispute.md` 는 **라우팅 직후 `rm`** 한다 — 파일 존재 자체가 트리거라 소비 후 남기면 다음 판정에서 재발동한다. 후속 에이전트에 필요한 맥락은 하네스 관행대로 재호출 프롬프트로 전달.

### 드라이버 컨텍스트 위생 (§1 공통)
- **매 회차 종료 시** `$ARTIFACTS_DIR/iteration_log.md` 에 1~3줄 append: 회차 번호·fail 추이·현재 단계·**N 상한·플래그(+판정 주체)**·동결 스냅샷 파일명·**동결 다이제스트 사본(shasum 두 줄, 저신뢰)**·`mutation: 시도 M·kill K·INVALID x·생존 S`.
- **escape 라우팅 직후 append 의무**: spec_gap/test_dispute `rm` 시점·AC defects→architect 라우팅 시점에 `escape: test_dispute #k` 식 1줄을 `iteration_log.md` 에 append 한다(반려 카운터의 디스크 앵커 — 컴팩션 후 재수화용).
- **컴팩션/세션 재개 후**: SSOT 와 어댑터를 **다시 Read** 해 경로·마커 규약을 복구하고, `iteration_log.md`·`failures.md`(있으면)를 Read 해 회차 상태를 재수화한 뒤 이어 간다. §2 (0) 대조에 쓸 G2 다이제스트 기록이 컨텍스트에 남아 있는지 먼저 확인한다 — 소실됐으면: (a) iteration_log 의 저신뢰 사본과 디스크(`.filelist`/`.sha256`)를 대조하고, 고신뢰 앵커 소실 사실을 사용자에게 1줄 고지한다(**무언 생략 금지**) — 불일치면 변조 의심, 즉시 중단·사용자 보고. (b) 일치해도 사본은 저신뢰이므로 현재 테스트 파일에 대해 **G2 재심사(escapes 0)→재동결**을 강제해 다이제스트를 새로 컨텍스트에 고정한 뒤 이어 간다(체크섬 앵커 소실은 의미 백스톱 G2+Mutation 으로 엄격성 재수립 — 비용은 G2 1회). 추가로 (c) `g2_verdicts.md` 헤더(G2 라운드 R)와 iteration_log 의 escape 줄로 반려 카운터를 재수화한다. (d) 에이전트 호출 중 컴팩션으로 가드 '전' 다이제스트가 소실됐으면: 내부 커맨드를 재실행해 `.guard_pre.txt` 와 diff 하되, 사본이 저신뢰(피검사 에이전트 쓰기 범위)임을 사용자에게 1줄 고지한다(**무언 생략 금지** — G2 앵커 소실과 동일 원칙). implementer 지원파일 가드는 §2 동결 체크섬이 이중 방어하므로 G 판정으로 대체 확인한다.

## 2) 게이트 G (★ 당신이 직접 Bash — iteration 의 단위)
implementer 산출 후 당신이 직접 실행해 측정한다(모든 커맨드는 어댑터 값):
```bash
ART="$ARTIFACTS_DIR"; FROZEN="$FROZEN_DIR"
: > "$ART/verifier_raw.txt"   # 매 게이트 시작 truncate — 마커 '정확히 1회' 보장(reviewer 는 reviewer_raw.txt 에 별도 기록)
# ── (0·최우선) 스냅샷 자체 무결성 — 디스크($FROZEN 포함)는 implementer 쓰기 가능이라, G2 동결 직후 '대화 컨텍스트'에 기록한 다이제스트와 대조.
shasum "$FROZEN/<카드>.filelist" "$FROZEN/<카드>.sha256"
# ↑ 출력이 G2 시점 컨텍스트 기록값과 불일치 → 스냅샷 변조 — 현재 .filelist/.sha256 내용과 G2 기록 다이제스트를
#   나란히 violations.md 에 append([회차 N·역할 R·검출 지점] 헤더)한 뒤 즉시 중단·사용자 보고(아래 검사 무의미). 일치 시에만 계속.
# ── 동결 무결성(implementer 가 test 안 건드렸나) — (1) 파일 목록 동일 (2) 내용 체크섬 동일. 둘 다 G2 escapes-0 시점 스냅샷 대비.
#   ★ 재동결은 매 G2 escapes-0 직후에만(SSOT) → 그 사이(implementer 단계) test 변경만 변조로 본다.
#   ★ zsh 단어분리: 체크섬은 xargs 파이프로(변수 확장 금지).
#   ★ 열거는 G2 동결과 동일하게 TEST_FILE_GLOB ∪ TEST_SUPPORT_GLOBS 합집합('(없음)' 인 절은 생략).
{ find <TEST_FILE_GLOB find 절>; find <TEST_SUPPORT_GLOBS find 절들>; } | sort -u | diff - "$FROZEN/<카드>.filelist" >/dev/null 2>&1; echo "FROZEN_LIST=$?" >> "$ART/verifier_raw.txt"
tr '\n' '\0' < "$FROZEN/<카드>.filelist" | xargs -0 shasum | diff - "$FROZEN/<카드>.sha256" >/dev/null 2>&1; echo "FROZEN_OK=$?" >> "$ART/verifier_raw.txt"
# ── 테스트(영향-스코프 — SSOT §테스트 스코프 / 어댑터 TEST_SCOPE_RULES, 경로 명시)
#   ★ 어댑터 HUMAN_GATE 골든 규약의 태그/글롭에 매칭되는 골든 테스트는 기계 게이트·mutation 실행에서 제외(예: --exclude-tags golden) — 기준은 사람 승인 후에만 생성(§6).
$TEST_CMD <영향 경로들> > /tmp/g_test.txt 2>&1; echo "TEST_EXIT=$?" >> "$ART/verifier_raw.txt"
#   ★ TEST_EXIT!=0 이면 /tmp/g_test.txt 의 tail 만 Read(failures.md 스키마의 '원문 tail' 확보용 — 전문 Read 금지).
$LINT_CMD > /tmp/g_lint.txt 2>&1; echo "ANALYZE_EXIT=$?" >> "$ART/verifier_raw.txt"
# ── 어댑터 GUARDS 루프(줄-상한 GUARD 포함 — Step 0a (4)에서 존재를 검증했다): 각 GUARD 의 커맨드를 실행하고 통과조건(대개 "출력 비어야 함" 또는 "exit 0")을 확인.
#   위반(출력 있음/exit≠0)이면 그 GUARD 이름과 출력을 위반으로 기록. 예시는 어댑터 GUARDS 참조.
#   각 GUARD: <커맨드> → 통과조건 위반이면 GUARD_VIOLATION=<이름> 기록.
```
- **e2e 플래그면** §3 E2E 도 게이트에 포함.
- **게이트는 잴 뿐 — 실패면 파이프라인 전체 재실행.** M개 fail / 위반 / mutation 생존이면 `$ARTIFACTS_DIR/failures.md`(실패 테스트·메시지·생존 변형 — 형식은 SSOT §게이트 실패의 failures.md 스키마를 따른다, 드라이버 진단·수정 제안 기입 금지) 적고 **처음(architect)부터 다시 돌린다**: `Agent(architect)`(실패 맥락은 failures.md 경로 전달로 한정 — 드라이버 자체 진단·요약을 프롬프트에 싣지 않는다) 재계획 → (design면)`Agent(designer)` → `Agent(test-author)`(테스트 추가/수정/유지) → **`Agent(test-auditor)` 재심사(escapes 0)→재동결** → `Agent(implementer)` → **게이트**. ★ 재실행에서도 **G2·재동결 생략 금지**(끼워맞추기 백스톱). 콕 집어 한 에이전트만 부르지 말 것 — 재계획은 architect 가 하고 아래가 그대로 흐른다. **`failures.md` 는 재실행의 implementer 호출이 끝난 직후 `rm`**(존재가 트리거 — 소비 후 남기면 재발동).
  - **입력 안 바뀐 단계는 산출 그대로**(불필요 재생성·테스트 churn 금지) — **재사용 여부는 기계 판정**: 재실행 architect 의 새 `plan.md` 를 `plan.prev.md` 와 `diff` 한다. 차이가 없거나 `### Notes` 섹션 안에만 국한되면(계획 유지 + 단순 구현 버그 노트) designer/test-author/G2 를 **스킵하고 직전 동결 스냅샷 유지** → implementer 재구현. 단, 이번 재실행의 failures.md 가 **mutation 생존·reviewer 갭·explorer 결함 환원**을 담고 있으면 plan diff 가 Notes-only 여도 **test-author→G2→재동결은 반드시 흘린다**(스킵 허용은 designer 뿐). Notes 밖이 **1줄이라도** 다르면 계획 변경 — 전체 흐름(designer→test-author→G2→재동결)을 다시 탄다. Explore 맵은 코드베이스 그대로면 재사용.
  - → 한 바퀴 = 다음 회차. 0 fail 까지, 최대 N회.
- **동결 불일치(FROZEN_LIST!=0 또는 FROZEN_OK!=0, implementer 단계)** → implementer 가 test 변조(수정·추가·삭제 모두) → 즉시 미통과 — 해당 `diff … >/dev/null` 를 리다이렉트 없이 재실행해 그 출력을 `violations.md` 에 같은 형식(`[회차 N·역할 R·검출 지점]` 헤더, 상한 200줄)으로 append 한 뒤 "test/ 복원·소스로 통과" 지시. (재동결은 매 G2 통과 직후만 갱신되므로, test-author 의 정당 추가는 G2 후 스냅샷에 반영돼 불일치가 아니다 — implementer 구간 고정 스냅샷 대비만 본다.)

## 2b) Mutation 게이트 (test 0 fail 후 — 끼워맞추기 차단, 당신 직접)
정답지(동결 테스트)를 보고 구현한 게 *진짜 맞는지* 모델 판단 없이 실증한다 — **변경/생성된 impl 에 미묘한 변형을 심어 영향-스코프 테스트가 잡는지** 본다.
- 대상: 이 카드가 바꾼 **손으로 쓴** impl 파일만(전 코드베이스 아님·**`MUTATION_EXCLUDES` 제외**). 어댑터 HUMAN_GATE 골든 규약의 태그/글롭에 매칭되는 골든 테스트는 mutation 실행에서도 제외한다(§2 와 동일 — 승인 baseline 없인 판정 불능). **최종 G2 산출물의 `## Mutation targets`(당신의 대화 컨텍스트 기록 — §1 규칙대로 g2_verdicts.md 에는 없다)를 우선순위 메뉴로 소비**해 당신이 변경 impl 파일 위 실제 위치로 매핑한다(전 항목 주입 의무 아님 — 예산은 기존대로 파일당 ~2~4개, 부족분은 당신이 보충). 이 목록은 드라이버만 보유하며 implementer 에게 전달하지 않는다(g2_verdicts.md 아카이브에서도 제외).
- 변형 예: `<`↔`<=`·off-by-one·가드 1줄 제거·보존 필드 `→ null`·상태변이 no-op·분기 조건 반전·필터 술어 제거.
- 절차(★ **기본 = 격리 사본** — 메인 작업트리는 미커밋 WIP 를 담고 있어, 변형 주입~원복 사이에 세션 크래시/컨텍스트 압축이 끼면 사용자 코드가 오염된 채 남는다):
  ```bash
  MUT="<세션 scratchpad>/mutation_copy"
  rsync -a --delete --exclude .git --exclude build --exclude .dart_tool --exclude node_modules ./ "$MUT"/
  (cd "$MUT" && ${BUILD_GEN_CMD:+eval "$BUILD_GEN_CMD"} )   # 경로 종속/생성물 복원(있으면). 없으면 생략.
  ```
  사본 안에서, 변형당 **Edit 1회 + 단일 Bash 1회**로 처리한다(§2 테스트 출력 규율의 §2b 미러링 — 변형 N개×전체 테스트 로그가 드라이버 컨텍스트로 유입되는 것 차단). **★주입은 패턴 기반(v0.9)**: 줄번호 지정 sed/치환 금지 — `grep -n` 으로 대상 코드 원문·실위치를 실측한 뒤 패턴 매칭 Edit 로 주입하고, 주입 직후 diff 로 의도 위치인지 확인한다(줄번호 어긋난 주입 = 빈 변형 → 가짜 생존/무효 판정 실측 2회):
  ```bash
  mkdir -p "$ARTIFACTS_DIR/mutlogs"   # 테스트 출력은 파일로만 — 트랜스크립트에는 exit 마커 한 줄
  (cd "$MUT" && $TEST_CMD <미러 test 경로>) > "$ARTIFACTS_DIR/mutlogs/<변형ID>.txt" 2>&1; echo "MUT_<변형ID>_EXIT=$?"; cp <원본 파일> <사본 경로>
  ```
  판정: `MUT_<변형ID>_EXIT=0` → **생존**(기계 판정 — 로그 안 읽음) / `≠0` → 해당 `mutlogs/<변형ID>.txt` 의 **tail 만 Read** 해 kill / INVALID 분류(아래 INVALID 규칙 그대로 — 폐기 후 타입-보존 변형 재주입). 집계는 변형당 한 줄(`파일:변형종류=판정`)만 남긴다. **메인 작업트리는 어떤 경우에도 수정하지 않는다.** (주의: `git worktree add … HEAD` 는 미커밋 구현이 빠져 무의미 — 쓰지 말 것.) rsync 가 불가한 환경에서만 예외적으로 메인 트리 정확-바이트 save/restore 를 쓰되, 주입 전 `cp <파일> <scratchpad>/mut_backup/` 영구 백업을 먼저 남긴다(크래시 복구 지점). `git checkout/stash/clean` 금지.
- **★ INVALID 변형 처리(중요)**: 변형이 **컴파일/로드 실패**를 내면 그건 **kill 이 아니라 무효(INVALID)** 다 — 컴파일 단계에서 전부 실패해 "잡힌 것처럼" 보여도 테스트 엄격성을 증명하지 못한다. INVALID 는 **세지 말고 폐기**하고, **컴파일은 되지만 동작이 달라지는 변형**으로 다시 주입한다(타입-보존: 가드 제거·경계 이동·no-op 화·분기 반전 등). kill 은 *컴파일되는 변형을 테스트 단정이 fail 시킬 때*만 인정.
- `MUT_SURVIVED=<n>` 마커를 `$ARTIFACTS_DIR/verifier_raw.txt` 에 기록 — n 은 **exit-0 변형 개수**(INVALID 는 분모에서 제외). 집계 한 줄 `mutation: 시도 M·kill K·INVALID x·생존 S` 를 `iteration_log.md` 회차 append 에 포함한다(집계는 이미 여기서 계산 — 추가 연산 0).
- **생존 ≥1 → G 미통과**: 생존 변형을 `failures.md` 에 적고 → architect 부터 파이프라인 재실행(test-author 가 그 변형 잡는 케이스 추가) [iteration +1]. **생존 0 이어야** reviewer 로.

## 3) E2E (e2e 플래그일 때, 당신 직접 — 게이트 일부)
```bash
ART="$ARTIFACTS_DIR"
# E2E_CMD 는 어댑터가 준 실행(시뮬레이터/브라우저/헤드리스 드라이버). ★ timeout 래퍼 필수(hang 결정적 차단).
${TIMEOUT:-$(command -v timeout||command -v gtimeout)} 300 $E2E_CMD > /tmp/g_e2e.txt 2>&1; echo "E2E_EXIT=$?" >> "$ART/verifier_raw.txt"
```
- **E2E 출력은 파일로만** — `E2E_EXIT!=0` 일 때만 `/tmp/g_e2e.txt` 의 tail 을 Read 해 failures.md 원문 발췌에 쓴다(전문 Read 금지 — §2 g_test.txt 와 동일 규율).
- `E2E_EXIT=124`=hang, `!=0`=실패 → G 미통과(회차 계속).
- **timeout 바이너리 부재 시**: `timeout`/`gtimeout` 둘 다 없으면(macOS 기본 상태) **무-timeout 실행 금지** — 사용자에게 `brew install coreutils`(gtimeout) 설치를 요청하고 E2E 를 중단한다(hang 차단 장치 없이 돌리지 않는다).
- **환경 미기동 = 통과 아님**: 실행 환경(시뮬레이터/브라우저)이 없으면 당신이 직접 기동 시도(어댑터 `HUMAN_GATE`/`E2E_CMD` 주석의 기동법). 그래도 불가면 **자동 green 선언 금지·중단하고 사용자에게 환경 기동을 요청**한다(e2e 플래그 카드에서 E2E 생략 green 은 human-visual 플래그가 없으면 무보완 구멍).

## 4) reviewer (게이트 0 fail 후 — 적대적 갭 사냥)
- 호출 전 `: > "$ARTIFACTS_DIR/reviewer_raw.txt"` truncate(이전 카드/회차 마커 잔존 방지) + `{ git status --porcelain .; git diff HEAD -- .; git ls-files -o --exclude-standard -z . | xargs -0 shasum; } | shasum` 기록. (★ 상태 전용 `git status|shasum` 은 이미-dirty/untracked 파일의 **내용** 변조를 못 잡는다 — reviewer 시점 트리는 매 회차 dirty 이므로 diff+untracked 내용 해시까지 포함.)
- **게이트가 0 fail 이 된 뒤** `Agent(reviewer)` 호출 ← `$ARTIFACTS_DIR/plan.md` 경로(**"Context·Card flags·Files·Reuse·Interface·AC·Visual 섹션만 읽어라 — Rationale·Notes 노출 금지"** 를 프롬프트에 명시, SSOT reviewer 예외 규정에 따른 섹션 제한 — test-author 의 spec.md 제한과 달리 reviewer 는 plan.md 에서 Rationale·Notes 만 차단, Card flags 는 사람 게이트 몫 분류에 필요해 허용) + 변경 + "테스트가 *놓친* 결함을 사냥하라(도장 금지)". reviewer 는 accept/reject 가 아니라 **갭 목록**을 낸다. 산출물에 `## Expected` 섹션이 없으면 재호출. 판정 수신 직후 당신이 `$ARTIFACTS_DIR/reviewer_verdicts.md` 에 같은 헤더 형식(`## [<카드> · 회차 N]`)으로 Gaps/Checklist ✗ 항목/Verdict + 환원 라우팅 결과(어느 갭이 test-author 케이스로 갔는지) 1줄을 append 한다.
- 호출 후 같은 커맨드로 재비교 — 달라졌으면 reviewer 가 파일을 수정한 것(Bash 보유라 도구로 못 막음 — 수정 금지는 지시문) → 원복 지시. (reviewer_raw.txt 기록은 예외라 다이제스트 대상에서 제외: gitignore 된 `$ARTIFACTS_DIR` 는 `git status`/`--exclude-standard` 모두 안 세므로 자연 제외된다.)
- **갭 ≥1** → 종류별로: 데모/제품을 깨는 결함 → 그 케이스를 test-author 에 추가시킴 → **G2 재심사(escapes 0)→재동결** → **게이트 재진입**(fail→fix 루프) / 중복·죽은코드 → implementer 가 정리 → **게이트 재진입**. 비차단 잠복은 Notes 로만 기록. (테스트 변경은 경로 불문 G2→재동결을 거친다 — 이 관문 없이 스냅샷 갱신 금지.)
  - **★수렴 규칙(lean 기본 — SSOT §실행 모드)**: reviewer **본심사는 카드당 1회**다. 본심사의 갭이 전부 환원되어 게이트가 다시 green 이면 **갭 해소로 판정하고 reviewer 를 재호출하지 않는다**(신규 갭 재사냥은 G2 델타 심사+mutation 이 백스톱). **deep 모드에서만** 갭 0 이 나올 때까지 재사냥을 반복한다. 단 lean 이어도 환원 과정에서 **구현이 새 표면을 추가**했으면(테스트 보강만이 아니라 소스 신규 파일/공개 인터페이스 변경) 그 표면에 한정한 재검토 1회를 허용한다.
  - **★ 동형 확장(일괄 시정)**: 갭을 재계획으로 넘기기 전에 당신이 architect 프롬프트에 **"이 갭과 동형인 결함 표면(같은 원인·다른 소비처)을 전수 나열해 한 회차에 묶어라"** 를 명시한다. reviewer 가 **같은 원인 계열 태그**로 묶은 항목(Notes 포함)을 그대로 architect 재검토 대상에 포함시킨다 — 원인 동형성 판정은 reviewer 산출물의 태그를 따르고 당신이 내용을 재분류하지 않는다. 갭 항목에 태그가 누락되면 내용 판단 없이 reviewer 에 형식 보완(태그 부착 재출력)만 재요청한다(architect 프롬프트의 '동형 표면 전수 나열' 지시는 유지 — 태그 누락 백스톱). 같은 원인의 표면을 회차마다 하나씩 발견해 재파이프라인을 반복하는 것이 최대 낭비 지점이다(실측: 동형 갭 분할 발견 1회 = 재파이프라인 1회 ≈ 1시간).
- **갭 해소**(lean: 본심사 갭 전부 환원·게이트 green / deep: 재사냥 갭 0) → (explore 플래그면 §4b 로 / 아니면) 자동 green 후보. (간결화는 별도 polish 단계가 아니라 reviewer 가 잡으면 implementer 가 고치는 일반 수정.)

## 4b) explorer (explore 플래그 카드만)
- **★lean(기본) = 큐 등록만(SSOT §카드 플래그 explore)**: reviewer 갭 해소 후, `$ARTIFACTS_DIR/qa_queue.md` 에 `## [<카드 ID> · <날짜>]` 헤더로 **카드 요약·AC 1줄·변경 파일 목록·탐색 초점(EXPLORE_QA 초점 중 해당분)·시드/기동 힌트**를 append 하고 §5 로 진행한다(카드 상태 = `green (QA 대기)` — §7·§8 에 명기). 실제 탐색은 사용자가 `/iterate-harness:iterate-qa` 로 배치 실행. **인라인 탐색은 deep 모드 또는 카드에 `explore-inline` 명시 시에만** 아래 절차로:
- 전제: 어댑터 `EXPLORE_QA` 설정(미설정이면 Step 0b 에서 이미 차단). 호출 전 `{ git status --porcelain .; git diff HEAD -- .; git ls-files -o --exclude-standard -z . | xargs -0 shasum; } | shasum` 기록(★ 상태 전용 shasum 은 dirty/untracked 내용 변조 미탐 — reviewer 백스톱과 동일 커맨드) + `mkdir -p "$ARTIFACTS_DIR/explore"`.
- `Agent(explorer)` 호출 ← 카드/AC 요약 + 변경 파일 목록 + 어댑터 `EXPLORE_QA`(기동법·조작 도구·초점) + **시간상자**(기본 탐색 흐름 5~8개). explorer 는 명세 밖(인접 흐름·빈/대량 데이터·중단·역행·연타·경계 밖 입력)을 실물로 구동하고, 일회성 탐색 스크립트·증거는 `$ARTIFACTS_DIR/explore/` 에만 쓴다(동결 glob 밖 — test/·소스 루트 금지).
- 호출 후 같은 커맨드로 재비교 — 달라졌으면 explorer 가 제품/테스트를 수정한 것 → 원복 지시(ARTIFACTS_DIR 는 gitignore 라 자연 제외).
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
            AND 어댑터 GUARDS 위반 0 (줄-상한 GUARD 포함)
            AND MUT_SURVIVED==0 (Mutation 게이트 생존 0, INVALID 제외)
            AND reviewer 갭 해소 (lean: 본심사 1회 갭 전부 환원·게이트 green / deep: 재사냥 갭 0)
            AND (explore 없음 OR lean: qa_queue 등록 완료 OR deep·explore-inline: explorer 결함 0)
```
자동 green 까지 루프, **MAX_ITERS = N(기본 10) 하드 상한**. 초과·같은 fail 2회 연속 동일원인 미해결 시 중단·사용자 개입.

## 6) 사람 시각확인 게이트 (human-visual 플래그면 — 맨 끝·1회)
- **자동 green 확정 후에만**, 당신이 어댑터 `HUMAN_GATE` 방법(시뮬레이터/브라우저 스크린샷·CLI 출력 등)으로 직접 한 번 띄워(셀프검증) → 사용자에게 "이걸 확인해 달라"고 캡처/출력과 함께 제시.
- **골든/스냅샷 기준**(어댑터 HUMAN_GATE 골든 규약 있을 때): 기준 산출물은 **사람 승인 후에만** 생성한다(드라이버 임의 갱신 금지 — 잘못된 기준 박제 차단). 기계 게이트 스코프에서 제외(식별 수단은 골든 규약의 태그/글롭 — §2 참조).
- **사람 OK →** 드라이버가 HUMAN_GATE 골든 규약 커맨드(`--update-goldens` 등)로 기준(baseline)을 생성하고 §7 완료기록에 골든 생성 여부를 기재한다(신규 test/** 골든 파일은 다음 카드 동결부터 포함).
- 사용자가 "투박/틀림" → 그 지적을 **새 라운드**로(가능하면 test-author 새 케이스나 architect 재계획으로 환원). 매 회차마다 사람을 부르지 마라.
- human-visual 없는 순수 로직 카드 → 자동 green = done.

## 7) 완료기록 갱신 (`PLAN_PATH` 설정 시만)
`$PLAN_PATH` 카드 완료기록: 상태·**G 회차(테스트 N개·1회차 M fail→…→0 fail 추이)**·`mutation M개 중 K kill(중간 회차 생존 발견 총 S건)`·날짜·플래그·**모드(lean/deep)**·특이사항(violations.md 발생 건은 `[회차·역할·요지]` 요약 필수 기재).
- **human-visual 카드**: 자동 green = **'verification passed'**, `실기 시각확인 날짜: –`. 사람 OK 후 채워야 **'done'**.
- **explore 큐 카드(lean)**: 자동 green = **'green (QA 대기)'** — `/iterate-qa` 배치가 결함 0 확인 시 done 으로 갱신(결함 발견 시 해당 카드 `/iterate` 재실행 안내).
- 그 외: 자동 green = done.
- `PLAN_PATH` 미설정(직접 작업 문자열 모드)이면 원장 갱신 생략 — §8 종료 보고만.

## 8) 종료 보고
```
## Iteration 완료
**카드/작업**: <ID 또는 설명> · **모드**: lean|deep · **플래그**: [design/human-visual/e2e/explore] (판정: 원장 명시/architect/드라이버 0b/사용자 확인) · **회차**: X (테스트 N개, 1회차 M fail → 0 fail)
**변경 파일**: …
**게이트**: 테스트 N/N · LINT 무경고 · GUARDS(줄상한 포함) OK · E2E(해당 시) K/K · mutation M개 중 K kill(생존 발견 총 S건) · reviewer 갭 해소(발견 G건→환원) · explore(해당 시) 결함 0 | QA 큐 등록
**explorer 발견→테스트화**: <있었으면 무엇> | 없음 | QA 큐 등록(/iterate-qa 대기) | (explore 아님)
**reviewer 발견→테스트화**: <있었으면 무엇> | 없음
**완료 상태**: done | verification passed (실기 시각확인 대기)
**특이사항**: 1~2줄(잠복/후속 Note 포함 · violations.md 발생 건은 [회차·역할·요지] 요약 필수)
```
transient(Agent 크래시)는 해당 역할 1~2회 재호출. spec_gap/test_dispute/MAX_ITERS/동일-fail-2회/guard 위반(violations.md 기록 건) 중단·발생은 사유와 함께 사용자에게 보고.
