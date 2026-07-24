# iterate — 멀티에이전트 TDD 하네스 (Claude Code 플러그인)

한 작업(카드)을 **독립 서브에이전트 7종 + 메인 Claude 드라이버**가 TDD 사이클로 완성하는 하네스다. 핵심 아이디어는 하나:

> **게이트는 리뷰어의 도장이 아니라 테스트다.** 독립적으로 작성·동결된 테스트가 전부 통과하고, 그 테스트가 mutation-proof 임이 *기계로* 실증될 때만 통과한다.

LLM 에게 구현을 맡길 때 가장 흔한 실패는 "자기가 짜고 자기가 통과시켰다고 주장"하는 자가채점이다. 이 하네스는 **출제자(테스트 작성)·검사자(엄격성 심사)·구현자를 서로 다른 에이전트로 분리**하고, 테스트를 **레포 밖에 동결**하고, green 이 된 뒤에도 **변형(mutation)을 심어 테스트가 정말 지키고 있는지**를 드라이버가 직접 실증한다.

플러그인은 **언어/프레임워크 무관 코어**(명령·에이전트·SSOT)와 대상 레포마다 채우는 **프로젝트 어댑터**(`.claude/iterate.config.md`)로 나뉜다. 코어가 철학·격리·게이트 구조를 규정하고, 어댑터가 "무엇을 실행하고 무엇을 금지하는가"(테스트/린트 커맨드·가드·프로젝트 불변식)를 채운다.

---

## 퀵스타트 (5분)

**① 설치** (Claude Code 안에서 — 상세는 §4):
```
/plugin marketplace add MMSS9402/iterate-harness
/plugin install iterate-harness@iterate-harness
```

**② 어댑터 생성** (대상 레포에서 레포당 1회):
```
/iterate-harness:iterate-init
```
스택을 감지해 필수 필드를 레포 실측값으로 채운 `.claude/iterate.config.md` 를 생성하고 `ARTIFACTS_DIR` gitignore 까지 처리한다. **폴백(수동)**: 플러그인의 `examples/iterate.config.md` 를 `.claude/iterate.config.md` 로 복사 — 이때 **필드당 자기 스택 예시 블록 하나만** 남겨라(템플릿 통째 복사는 드라이버가 중단시킨다). `ARTIFACTS_DIR`(기본 `.iterate/artifacts`)를 `.gitignore` 에 추가.

**③ 첫 카드 실행** (맨 뒤 정수 = 회차 상한 N):
```
/iterate-harness:iterate "로그인 실패 시 에러 카피 개선" 3
```

**④ 기대 출력**: 플래그 없는 카드는 Explore→architect→test-author→test-auditor(G2)→implementer→게이트→reviewer 6단계가 사람 개입 없이 돈다(design/explore 플래그 시 designer/explorer 추가). 끝나면 종료 보고가 나온다 — 축약하면:
```
## Iteration 완료
**카드/작업**: … · **회차**: X (테스트 N개, 1회차 M fail → 0 fail)
**게이트**: 테스트 N/N · LINT 무경고 · mutation 생존 0 · reviewer 갭 0
**완료 상태**: done | verification passed (실기 시각확인 대기)
```

**⑤ 막히면**: 어댑터가 없으면 드라이버가 Step 0 에서 중단하고 안내한다(§5-1). 그 밖의 중단 조건(회차 상한 N 초과·escape 소진 등)은 §7.

---

## 동작 원리

아래 §1(구조)·§2(사이클 흐름)·§3(회차와 N)이 하네스의 동작 원리다.

## 1. 구조

### 파일 구성

```
commands/iterate.md              # 드라이버 절차서 — 메인 Claude 가 이대로 오케스트레이션
commands/iterate-init.md         # 어댑터 부트스트랩 — 스택 감지·레포 실측으로 어댑터 생성+자가 검증
commands/iterate-qa.md           # 배치 탐색 QA — lean 이 큐에 미룬 explore 카드를 환경 1회 기동으로 묶어 탐색 (Workflow 병렬 허용, v0.8)
commands/iterate-design.md       # 디자인 스펙 단독 패스 — 화면군 단위 visual_spec 산출·캐시 등록 (v0.8)
agents/architect.md              # 설계자
agents/designer.md               # 시각 스펙 (design 카드 전용)
agents/test-author.md            # 테스트 출제자
agents/test-auditor.md           # 테스트 엄격성 검사자 (GATE2)
agents/implementer.md            # 구현자
agents/reviewer.md               # 적대적 갭 사냥꾼
agents/explorer.md               # 탐색 QA (explore 카드 전용 — 명세 밖 실물 탐색)
skills/iterate-protocol/SKILL.md # SSOT — 드라이버가 매 사이클 읽는 불변식·게이트 구조 단일 출처
skills/iterate-protocol/CORE.md  # 에이전트 공통 커널 — 역할 에이전트는 SSOT 대신 이 발췌본만 읽는다 (호출당 고정비 절감, v0.9)
examples/iterate.config.md       # 어댑터 템플릿 (Flutter 앱·Go 백엔드·React/TS 웹 예시 3종)
CHANGELOG.md                     # 버전별 변경·어댑터 영향 (업데이트 전 확인)
```

### 역할 8개 — 누가 무엇을 하고, 무엇을 못 하나

| 역할 | 하는 일 | 격리 장치 (못 하는 것) |
|---|---|---|
| **드라이버** (메인 Claude) | 카드 해석·에이전트 오케스트레이션·**게이트 측정·최종 판정**·완료기록 | 측정은 전부 Bash 직접 실행 — 모델의 "통과한 것 같다" 판단 배제 |
| Explore (빌트인) | 재사용 후보·기존 패턴 탐색 → architect 입력 | read-only |
| **architect** | 설계 + 인수기준(AC) + 카드 플래그 판정 | read-only. 코드 본문 작성 금지(GATE1 이 반려) |
| **designer** | 시각 스펙 (design 카드만, **구현 전**) | read-only. `DESIGN_SSOT` 없으면 진행 불가 |
| **test-author** | AC 를 독립 블랙박스 테스트로 — **명세에서만, 구현 안 봄** | 소스 루트 무수정(드라이버 전후 내용 다이제스트 백스톱) |
| **test-auditor** | GATE2: "틀린 구현이 이 테스트를 통과할 수 있나" 적대 심사 | 유일한 진짜 도구 봉인(Read/Grep/Glob) — 구현도 안 읽음 |
| **implementer** | 동결 테스트를 green 으로 만드는 구현 | 테스트 수정·추가·삭제 불가(동결 목록+체크섬이 검출) |
| **reviewer** | green 후 "테스트가 놓친 결함" 사냥 → 새 테스트로 환원 | accept/reject 권한 없음. 무수정은 드라이버 전후 내용 다이제스트 백스톱 |
| **explorer** | (explore 카드만) 실물 구동으로 **명세 밖** 탐색 — 인접 흐름·이상 상태·중단/역행 → 결함을 재현 경로와 함께 테스트로 환원 | 쓰기는 `$ARTIFACTS_DIR/explore/`(소모품)만 — 지시문 + 드라이버 전후 내용 다이제스트 백스톱 |

검사자와 피검사자가 같은 모델일 수 있으므로, 격리는 모델 다양성이 아니라 **구조**(정보 차단 + 기계 백스톱)로 강제한다. 출제자(test-author)·G2 검사자(test-auditor)에게는 plan.md 원문 대신 드라이버 발췌본(`spec.md`)만 전달하고, reviewer 는 plan.md 를 받되 Rationale·Notes 열람만 차단해 architect 의 추론이 새지 않게 한다. 검사자는 언제나 *명세에서 기대를 먼저 도출*하고 코드·테스트를 본다 — 이 커밋먼트는 산출물로 강제된다(test-auditor 는 테스트 열람 전 Mutant checklist, reviewer 는 diff 열람 전 Expected 를 먼저 고정). "테스트가 통과하니 맞겠지"를 금지한다.

역할별 모델은 에이전트 frontmatter 에 고정돼 있다: **실패가 조용한 역할**(architect·test-author·test-auditor·reviewer)과 explorer·designer 는 opus, **기계 백스톱이 결함을 시끄럽게 만드는** implementer 는 sonnet 이다. 드라이버는 세션 모델을 상속한다(opus 세션 권장). 근거·비용·커스터마이즈는 §6c.

### 산출물 위치

```
대상 레포/
├── .claude/iterate.config.md    # 프로젝트 어댑터 (당신이 작성)
└── $ARTIFACTS_DIR/              # 게이트 산출물 (gitignore 필수)
    ├── verifier_raw.txt         #   드라이버 게이트 마커 (TEST_EXIT= 등)
    ├── reviewer_raw.txt         #   reviewer 검증 마커 (별도 파일 — 마커 충돌 방지)
    ├── failures.md              #   실패 맥락 (재실행 시 architect 입력)
    ├── plan.md / plan.prev.md   #   architect 설계 영속화 (재실행 시 diff 로 재사용 기계 판정)
    ├── visual_spec.md           #   designer 시각 스펙 (design 카드)
    ├── spec.md                  #   드라이버 발췌본 (출제자·G2 검사자 전달용 — Rationale·Notes 비포함. reviewer 는 plan.md 섹션 제한)
    ├── g2_verdicts.md           #   test-auditor 판정 원문 아카이브 (회차·G2 라운드별 append, Mutation targets 제외)
    ├── reviewer_verdicts.md     #   reviewer 판정 원문 아카이브 (회차별 append)
    ├── violations.md            #   독립성 위반 인시던트 (가드 위반 기록 — 카드 간 누적, 소거 제외)
    ├── iteration_log.md         #   회차별 로그 (fail 추이·동결 다이제스트 사본·mutation 시도/kill/생존 수치)
    ├── mutlogs/                 #   mutation 변형별 테스트 출력 (트랜스크립트엔 exit 마커만)
    └── explore/                 #   explorer 일회성 탐색 스크립트·증거 (소모품)

$FROZEN_DIR/  (★ 레포 밖 — 기본 $HOME/.iterate-harness/<repo-slug>/frozen_tests)
    ├── <카드>.filelist          # 동결 시점 테스트 파일 '목록' (몰래 추가/삭제 검출)
    └── <카드>.sha256            # 내용 체크섬 (수정 검출)
```

동결 스냅샷을 레포 밖에 두는 이유: implementer 는 전체 도구를 가지므로 레포 안 스냅샷은 위조 가능하다.

---

## 2. 사이클 흐름 (한 카드 = 한 /iterate)

```
[기계 루프 — 사람 없음]
0 Explore      재사용 후보·기존 패턴 탐색
A architect    설계 + AC + 카드 플래그        ◄──────────── 게이트 실패 시 여기로 (전체 재실행)
   └ GATE1(드라이버): 설계에 코드 본문 없음 + 예상 줄수 ≤ FILE_LINE_LIMIT + [blocking] Open Q 는 사용자 확인
   └ spec.md 발췌: 출제자·검사자용 (Card flags·Interface·AC·Visual 만 — Rationale·Notes 차단)
(design?) designer   시각 스펙 — 구현 전에 구조를 한 번에 맞춤
T test-author  AC → 독립 블랙박스 테스트 (구현이 없으니 red 가 정상)
G2 test-auditor  "틀린 구현이 빠져나갈 변형" 적대 심사
   ├ escapes ≥1 → test-author 보강 (최대 2회, 소진 시 중단·사용자 개입)
   └ escapes 0 → ★ 동결 (레포 밖 목록+체크섬 스냅샷)
B implementer  동결 테스트를 green 으로 (test/ 불가침)
G 게이트(드라이버 직접 Bash)
   동결 무결성 · 영향-스코프 테스트 · LINT 전체 · 어댑터 GUARDS(줄-상한 GUARD 포함) · (e2e) E2E
   ├ 실패 → failures.md 기록 → architect 부터 전체 재실행   [회차 +1]
   └ 0 fail ↓
2b Mutation 게이트(드라이버 직접)
   격리 사본(rsync)에 변형 주입 → 미러 테스트가 잡는지 실증
   ├ 생존 ≥1 → architect 부터 전체 재실행 (test-author 가 그 변형 잡는 케이스 추가)  [회차 +1]
   └ 생존 0 ↓
C reviewer     테스트가 놓친 결함 사냥 (명세→기대 도출→반례 증거)
   ├ 갭 ≥1 → 결함은 test-author 케이스로 환원(G2 재심사→재동결 경유) → 게이트 재진입 / 중복·죽은코드는 implementer 정리
   │        ★lean(기본): 본심사는 카드당 1회 — 갭이 전부 환원·게이트 green 이면 해소(재사냥은 deep 만)
   └ 갭 해소 ↓
(explore?) lean: qa_queue.md 등록 → ★ 자동 green(QA 대기) — 탐색은 /iterate-qa 배치가 환경 1회 기동으로 수행
          deep·explore-inline: explorer 인라인 탐색 (어댑터 EXPLORE_QA — 인접 흐름·이상 상태·중단/역행, 시간상자)
   ├ 결함 ≥1 → 재현 경로를 test-author 케이스로 환원(G2 재심사→재동결 경유) → 게이트 재진입
   └ 결함 0 → ★ 자동 green 확정

[사람 게이트 — 맨 끝, 단 한 번]
(human-visual?) 사람 시각확인 → "투박/틀림" 이면 새 라운드로 환원 / OK 면 done
              → 골든/스냅샷 기준은 이 승인분에서만 생성
```

**단계별 핵심 규칙**:

- **게이트 실패 = 전체 재실행.** 콕 집어 한 에이전트만 다시 부르지 않는다 — 테스트 실패는 계획→디자인→테스트→구현으로 연쇄할 수 있어서 architect 부터 통째로 다시 흘린다. 단 **입력이 안 바뀐 단계는 산출을 그대로 재사용**(불필요한 재생성·테스트 churn 금지). 단순 구현 버그면 architect 가 계획을 유지하고 노트로 버그 위치만 짚는다.
- **재동결은 매 G2 escapes-0 직후에만.** 재실행에서 test-author 가 정당하게 케이스를 추가하면 G2 재심사 통과 후 재동결된다. 그 외 구간(implementer 단계)의 테스트 변경만 변조로 판정 — 정당 수정과 변조를 시점으로 구별한다. 이 관문은 **경로 불문**이다: 게이트 실패 재실행·reviewer 갭 환원·test_dispute 수정·mutation 생존 케이스 추가 — 테스트가 변경되는 모든 경로가 G2 재심사→재동결을 거친다.
- **Mutation 의 INVALID 판정**: 변형이 컴파일/로드 실패를 내면 kill 이 아니라 **무효**다(컴파일러가 잡은 것이지 테스트가 잡은 게 아님). 폐기하고 컴파일되는 변형으로 재주입한다.
- **escape hatch**: implementer 가 설계 불충분을 발견하면 `spec_gap.md`(→architect 재진입), 어떤 올바른 구현도 통과 못 하는 테스트를 발견하면 `test_dispute.md`(→test-author 가 그 케이스만 수정, 최대 2회). 구현자가 테스트를 직접 고치는 길은 없다.

### 자동 green 판정 (드라이버의 AND-gate)

```
자동 green = TEST_EXIT==0                      (영향-스코프 테스트 전부 통과)
           AND FROZEN_LIST==0 AND FROZEN_OK==0 (동결 파일목록·체크섬 일치)
           AND ANALYZE_EXIT==0                 (LINT 무경고)
           AND (e2e 플래그 없음 OR E2E_EXIT==0)
           AND 어댑터 GUARDS 위반 0             (줄-상한 GUARD 포함)
           AND MUT_SURVIVED==0                 (mutation 생존 0, INVALID 제외)
           AND reviewer 갭 해소 (lean: 본심사 1회 갭 전부 환원 / deep: 재사냥 갭 0)
           AND (explore 없음 OR lean: qa_queue 등록 OR deep·explore-inline: explorer 결함 0)
done      = 자동 green AND (human-visual 없음 OR 사람 시각확인 통과)
```

---

## 2b. 실행 모드 — lean(기본) / deep (v0.8 · 토큰·시간 최적화)

`/iterate <카드> [N] [deep]` — `deep` 토큰이 없으면 **lean 이 기본**이다. 기계 백스톱(동결 체크섬·mutation·GUARDS·LINT)은 모드 무관 동일 — 모드는 **LLM 심사의 반복 횟수**만 조절한다.

| 항목 | lean(기본) | deep |
|---|---|---|
| G2 재심사 | 델타 심사(변경·신규 테스트 파일 + 상호작용만 — 최초 심사는 항상 전체) | 매번 전체 재심사 |
| reviewer | 본심사 1회 — 갭 환원·게이트 green 이면 해소 | 갭 0 까지 재사냥 반복 |
| explorer | qa_queue 등록 → `/iterate-qa` 배치 | 인라인(green 전 차단) |
| designer | 스펙 캐시 히트 시 스킵(모드 무관) | 동일 |

**deep 을 쓸 때**: 보안·데이터 정합 카드, 직전 카드에서 갭/결함이 연쇄(2건+)로 나온 경우, 릴리스 직전. 일반 기능 카드는 lean 으로 충분하다.

**분리 명령 2개(v0.8)**:
- **`/iterate-harness:iterate-qa`** — lean 이 큐에 등록한 explore 카드들을 **환경 1회 기동**으로 묶어 배치 탐색(카드×초점 fan-out 은 Workflow 병렬 허용 — 하네스에서 유일한 예외). 카드 간 **교차 상호작용**까지 봄. 결함 발견 시 해당 카드 `/iterate` 재실행으로 환원.
- **`/iterate-harness:iterate-design`** — designer 를 사이클 밖에서 화면군 단위로 1회 실행해 visual_spec 을 캐시(design_cache.md · DESIGN_SSOT 다이제스트 키). 이후 design 카드는 캐시 히트로 designer 스킵.

## 3. iteration(회차)과 N — 횟수 지정

**회차(iteration) = [architect → (designer) → test-author → (G2) → implementer → 게이트] 파이프라인 한 바퀴.** 첫 게이트가 0 fail 이면 1회차다. 리뷰어가 되돌린 횟수가 아니다.

**N 은 회차의 하드 상한**이다. 지정 방법 — **인자 맨 뒤에 정수 하나**:

```
/iterate-harness:iterate "T2.7a"        # N 미지정 → 기본 10
/iterate-harness:iterate "T2.7a" 4      # N=4 — 4회차 안에 자동 green 못 만들면 중단·사용자 개입
/iterate-harness:iterate "로그인 실패 카피 개선" 3   # 직접 작업 문자열 + N=3
/iterate-harness:iterate "T2.9" 6 deep   # deep 모드 — 전체 G2 재심사·reviewer 재사냥·explorer 인라인
```

- N 은 **상한이지 목표가 아니다** — 0 fail 이 되면 N 전이라도 즉시 자동 green 으로 넘어간다.
- N 초과, 또는 **같은 fail 이 2회 연속 같은 원인으로 안 잡히면** 드라이버가 중단하고 사용자를 부른다(architect 재계획이 원인을 못 짚고 있다는 신호).
- 카드 원장(`PLAN_PATH`)을 쓰면 카드에 `**권장 N** 4` 처럼 적어두고 호출 시 생략하는 운영을 권장한다.
- **"1회차 green" 은 자랑이 아니다** — implementer 는 정답지(동결 테스트)를 보고 짜므로 1회차 통과는 거의 항진명제다. 품질 지표는 회차 수가 아니라 **mutation-kill-rate**(변형을 심었을 때 테스트가 잡는 비율)다.

---

## 4. 설치

**개인 설치** (Claude Code 안에서):
```
/plugin marketplace add MMSS9402/iterate-harness
/plugin install iterate-harness@iterate-harness
```
- private 레포이므로 GitHub 인증이 필요하다 — `gh auth login`(HTTPS) 또는 SSH 키가 등록돼 있으면 그대로 동작한다. 백그라운드 자동 업데이트까지 원하면 `GH_TOKEN`(repo 스코프) 환경변수를 둔다.
- 설치 후 호출명은 **`/iterate-harness:iterate`** 다(플러그인 명령은 항상 `플러그인명:명령` 네임스페이스 — bare 폴백 없음). 짧은 `/iterate` 를 원하면 대상 레포에 한 줄짜리 로컬 셔틀을 둔다:
  ```markdown
  <!-- .claude/commands/iterate.md -->
  ---
  description: iterate-harness 플러그인 드라이버 셔틀
  ---
  Skill 도구로 `iterate-harness:iterate` 를 args `$ARGUMENTS` 로 호출하라.
  ```

**업데이트** (이 레포에 새 커밋이 올라간 뒤):
```
/plugin marketplace update iterate-harness      # ① 마켓플레이스 클론(git pull) 갱신
/plugin update iterate-harness@iterate-harness  # ② 플러그인 재설치(새 버전 반영)
```
- ①만 하면 카탈로그만 갱신되고 설치본은 그대로다 — ②까지 해야 명령·에이전트·SSOT 가 새 버전으로 바뀐다. `/plugin update` 가 없는 구버전 Claude Code 는 `/plugin uninstall` 후 `/plugin install` 로 재설치하면 된다.
- 적용 확인: `/plugin` 관리 UI 에서 버전(`plugin.json` 의 `version`)을 본다. 실행 중인 세션에는 소급 적용되지 않으니 새 세션에서 확인.
- **어댑터는 업데이트와 무관하다** — `.claude/iterate.config.md`(ROLE_* 포함)는 대상 레포에 있으므로 플러그인을 갱신해도 그대로 유지된다. 단 새 버전이 어댑터 필드를 추가했으면([CHANGELOG.md](CHANGELOG.md) 참조) `examples/iterate.config.md` 를 보고 해당 필드만 보강하라.

**팀 자동 설치** (대상 레포에 커밋):
```json
// .claude/settings.json
{
  "extraKnownMarketplaces": {
    "iterate-harness": { "source": { "source": "github", "repo": "MMSS9402/iterate-harness" } }
  },
  "enabledPlugins": { "iterate-harness@iterate-harness": true }
}
```
팀원이 클론 후 폴더를 신뢰하면 설치 제안을 받는다.

---

## 5. 사용법

### 5-1. 어댑터부터 (레포당 1회)

1. 대상 레포에서 `/iterate-harness:iterate-init` 를 실행한다 — 스택을 감지해 필수 필드를 레포 실측값으로 채운 `.claude/iterate.config.md` 를 생성하고 자가 검증까지 한다. **폴백(수동)**: `examples/iterate.config.md` 를 대상 레포의 `.claude/iterate.config.md` 로 복사하고, 각 필드에서 당신 스택에 맞는 예시 블록(Flutter/Go/Web 중)만 남긴 뒤 값을 프로젝트에 맞춘다.
   - 필수: `PROJECT`·`TEST_CMD`·`LINT_CMD`·`GUARDS`·`FILE_LINE_LIMIT`·`FROZEN_DIR`·`ARTIFACTS_DIR`·`PROJECT_INVARIANTS`·`TEST_FILE_GLOB`·`TEST_SUPPORT_GLOBS`(없으면 `(없음)`)·`MUTATION_EXCLUDES`·`TEST_SCOPE_RULES`
   - 선택: `BUILD_GEN_CMD`(코드생성)·`E2E_CMD`·`PLAN_PATH`(카드 원장)·`DESIGN_SSOT`(디자인 시스템)·`HUMAN_GATE`(시각확인 방법)·`EXPLORE_QA`(탐색 QA 방법)·`ROLE_*`(역할별 파인튜닝)
2. `ARTIFACTS_DIR`(기본 `.iterate/artifacts`)를 `.gitignore` 에 추가한다. `FROZEN_DIR` 는 레포 밖이라 불필요.
3. 어댑터 없이 실행하면 드라이버가 Step 0 에서 멈추고 이 안내를 준다 — 어댑터가 항상 먼저다.

### 5-2. 실행 모드 2가지

**카드 원장 모드** (`PLAN_PATH` 설정 시): 원장 마크다운에 카드를 쌓아두고 ID 로 호출한다. 드라이버가 카드 블록만 Grep+부분 Read 하고(대형 원장 컨텍스트 보호), 끝나면 **완료기록**(상태·회차·fail 추이·날짜·특이사항)을 원장에 남긴다 — 크로스 세션 감사 추적의 원천.
```
/iterate-harness:iterate "T2.7b"
```

**직접 작업 모드** (`PLAN_PATH` 없거나 ID 가 아닌 문자열): 인자 문자열 자체가 작업이다. 원장 조회·완료기록 없이 종료 보고만 한다.
```
/iterate-harness:iterate "프로필 화면에 로그아웃 확인 다이얼로그 추가" 3
```

### 5-3. 카드 플래그 4종 (검증 방법이 갈린다)

| 플래그 | 언제 | 효과 |
|---|---|---|
| `design` | 새 시각 표면(화면·위젯·페이지)을 만들 때 | designer 단계 추가(구현 **전**에 시각 스펙). `DESIGN_SSOT` 필수 |
| `human-visual` | 자동 테스트가 못 보는 렌더/미감 | 자동 green 후 맨 끝에 **사람 확인 1회**. 그 전까진 'verification passed' |
| `e2e` | 헤드리스에서 안 뜨는 표면(webview·브라우저 등) | `E2E_CMD` 가 기계 게이트에 포함. 환경 미기동 = 통과 아님 |
| `explore` | 사용자 흐름이 얽힌 화면·상태 많은 표면 — **명세가 상상 못 한 결함**까지 볼 가치 | reviewer 갭 0 후 explorer 가 `EXPLORE_QA` 방법으로 실물 탐색(시간상자). 결함은 테스트로 환원. 환경 미기동 = 통과 아님 |

플래그가 없으면 순수 로직 카드 — 자동 green 이 곧 done. 카드에 명시가 없으면 드라이버가 작업 성격으로 판정하고, 모호하면 1줄 물어본다.

### 5-4. 돌아가는 동안 사용자가 하는 일

기계 루프에는 사람이 없다. 개입 지점은 정확히 네 곳:
1. **설계 확정 전 확인** — architect Open questions 에 `[blocking]` 항목(AC 확정에 답 필요)이 있거나, architect 가 원장/사전 판정된 카드 플래그를 **끄려** 할 때 드라이버가 1줄 확인을 물어온다.
2. **designer Open Question** — 새 디자인 토큰 등 정책 결정이 필요할 때 드라이버가 물어온다.
3. **중단 보고** — MAX_ITERS 초과·동일 fail 2회·spec_gap·test_dispute 소진·GATE2 반려 소진 시.
4. **사람 시각확인** (human-visual 카드) — 자동 green 후 드라이버가 셀프검증 스크린샷과 체크리스트를 제시하면 실물을 보고 OK/지적. 지적은 새 라운드(가능하면 테스트 케이스로 환원)가 된다. **골든/스냅샷 기준 산출물은 이 승인 후에만 생성된다.**

### 5-5. 실전 사례 (원조 레포의 T2.7a)

Flutter 앱 "모임 합류 경로" 카드 — 이 하네스의 첫 실전:
- **회차 1**: 기계 게이트 0 fail(테스트 1,038) — 그러나 reviewer 가 "빈 로스터 공개모임에서 게이트가 무력화돼 둘러보기 4곳 중 3곳에서 기능이 안 보임"(GAP1)을 잡아 테스트 3건으로 환원 → red
- **회차 2**: 시드 보강으로 GAP1 해소, 0 fail(1,041)
- **사람 게이트**: 사용자가 "뒤로가기 버튼이 2개"(Scaffold 중첩) 발견 → AC51 쌍으로 환원 → **회차 3**: 구조 수정 후 0 fail(1,043) → 사람 OK → done
- 그 과정에서 G2 가 escape 2건을 동결 전에 막았고, implementer 는 테스트의 컴파일 결함을 `test_dispute` 로 정당하게 반려했으며, mutation 12/12 kill 로 마감. **기계가 한 결함을, 사람이 다른 결함을 잡았다 — 층이 다르면 그물도 달라야 한다는 것이 이 하네스의 요지다.**

---

## 6. 어댑터 작성 요령

- **TEST_CMD/LINT_CMD** 는 영향 경로를 인자로 받는 형태로 둔다(드라이버가 경로를 덧붙임). **래퍼가 아니라 기저 러너로** — `make test` 는 trailing 경로 인자를 별도 타깃으로 해석해 깨지고, `npm test` 류는 watch 모드 기본이면 게이트가 무한 대기한다(`go test -race`·`npx vitest run` 처럼 풀어 쓸 것 — iterate-init 이 스모크 실행으로 검증해 준다). 전체 스위트를 매번 돌리지 않는다 — 전체 회귀는 Phase 경계에서만.
- **GUARDS** 는 `{이름·커맨드·통과조건}` 선언형이다. *기계로 0/1 판정 가능한 규칙*만 넣는다(리터럴 금지 grep·fmt diff 등). 값 테스트로 못 잡는 규약(예: 식별자==wire 인 enum 직렬화)이 GUARD 의 대표 용도다. **`FILE_LINE_LIMIT` 를 참조하는 줄-상한 가드는 필수** — 없으면 드라이버가 Step 0a 에서 중단한다(줄-상한 판정은 GUARDS 로 일원화).
- **게이트 커맨드는 xargs 파이프로** — zsh 는 무인용 변수 확장을 단어분리하지 않아 `$(cat list)` 류가 한 덩어리로 붙는다.
- **PROJECT_INVARIANTS** 는 SSOT 불변식의 프로젝트 슬롯을 채운다 — 아키텍처 규약(DI·추상화 경계), 헤드리스 미렌더 표면 목록, 운영 분기 일치 규칙 등.
- **통합 테스트가 실 DB 를 요구하면** TEST_CMD 에 전제조건을 명시하라(예: `사전조건: docker compose up -d`). `connection refused` 류 실패는 코드 결함이 아니라 환경 미기동이다.
- **E2E 도구는 스택을 따라간다** — 모바일은 시뮬레이터(`flutter test integration_test`/simctl), 웹은 **Playwright**(`npx playwright test`, `webServer` 자동 기동), 백엔드는 빌드태그 통합 테스트. 헤드리스 단위 테스트(jsdom/widget test)가 원리상 못 보는 표면(실브라우저 렌더·webview·쿠키)을 `e2e` 플래그 + `E2E_CMD` 가 메운다. **E2E 테스트 파일 경로는 동결 글롭(`TEST_FILE_GLOB` ∪ `TEST_SUPPORT_GLOBS`) 안에 둘 것** — 밖이면 implementer 의 e2e 테스트 변조를 동결 체크섬이 못 잡는다(드라이버가 0b 에서 확인).
- **HUMAN_GATE 골든 규약**을 쓰면 **기계 식별 수단(태그 또는 글롭)을 반드시 함께 정의**한다(예: Flutter `@Tags(['golden'])` + `--exclude-tags golden`) — 드라이버가 골든을 기계 게이트·mutation 스코프에서 제외하고, 기준(baseline)은 사람 승인 후에만 생성한다.
- 세 예시(Flutter 모바일 앱·Go 백엔드·React/TS 웹)가 UI 있는/없는 스택, 원장 있는/없는 운영, 시뮬레이터/브라우저 E2E 를 모두 보여준다.

## 6b. 프로젝트별 파인튜닝 — `ROLE_*` 섹션

플러그인으로 설치하면 에이전트 프롬프트 자체는 수정할 수 없다. 대신 **어댑터에 역할별 지침 섹션**을 두면, 각 에이전트가 실행 시 자기 섹션을 읽어 **추가 지침**으로 따른다:

```markdown
<!-- .claude/iterate.config.md 에 추가 -->
## ROLE_REVIEWER
- 최우선 사냥 축: 입력 검증 우회·인젝션·인증/인가 구멍. 발견 시 반드시 테스트로 환원하라.

## ROLE_ARCHITECT
- 외부 입력이 닿는 AC 에는 검증 실패(거부) 케이스를 반드시 포함하라.
```

- 지원 섹션 7종: `ROLE_ARCHITECT`·`ROLE_DESIGNER`·`ROLE_TEST_AUTHOR`·`ROLE_TEST_AUDITOR`·`ROLE_IMPLEMENTER`·`ROLE_REVIEWER`·`ROLE_EXPLORER`.
- **우선순위: SSOT 불변식·격리·게이트 구조 > ROLE_* 지침.** 게이트를 약화시키는 지시(escapes 묵인·test/ 수정 허용 등)는 에이전트가 무시하고 산출물에 충돌로 보고한다 — "무엇을 더 신경 쓸까"를 조정하는 계층이지 "무엇을 건너뛸까"가 아니다.
- 플러그인 업데이트를 받으면서 프로젝트별 특화를 유지할 수 있다(코어=플러그인, 특화=어댑터). 역할을 통째로 바꿔야 할 만큼 다른 프로젝트라면 에이전트 `.md` 를 프로젝트 `.claude/agents/` 로 복사해 수정하는 로컬 오버라이드도 가능하지만, 그 역할은 이후 플러그인 개선을 받지 못한다.

## 6c. 모델 라우팅과 비용

역할별 **모델·effort 는 에이전트 frontmatter 에 고정**돼 있다 — 게이트 심사 깊이가 사용자 세션 설정에 좌우되지 않게 하기 위해서다("어댑터/세션이 게이트를 약화시킬 수 없다" 원칙의 모델 파라미터 판).

| 역할 | 모델 | effort | 근거 |
|---|---|---|---|
| architect·test-author·test-auditor·reviewer | opus | xhigh | **실패가 조용한 역할** — 얕은 AC·얕은 케이스·느슨한 심사는 dead-green 으로 조용히 통과한다. 심사 깊이가 곧 게이트 엄격성 |
| explorer | opus | xhigh | 명세 밖을 실물 구동으로 사냥하는 적대적·에이전틱 역할 |
| designer | opus | high | 유일하게 DESIGN_SSOT 제약 내 정형 산출에 가까움 |
| implementer | sonnet | xhigh | **실패가 시끄러운 역할** — 동결 테스트+체크섬·mutation·GUARDS·reviewer 가 결함을 기계로 잡아준다 |
| 드라이버 (메인 Claude) | 세션 모델 상속 | 세션 설정 | 게이트 판정·mutation 변형 설계를 직접 하므로 **opus 세션 권장** |

**비용 구조**: 게이트 실패 1회의 비용은 "회차 +1"이 아니라 **architect 부터 전체 파이프라인 재실행**이다 — 총비용은 회차 수에 선형이고, **N(회차 상한)이 비용 상한 레버**다. 단 입력이 안 바뀐 단계의 산출은 재사용되므로(§2) 재실행이 항상 풀코스는 아니다.

**실측(v0.9 근거 — 실사용 프로젝트 1건·서브에이전트 733회 호출·73.7M 토큰)**: test-author+test-auditor 가 총비용의 **59%**(230+213회 호출 — 카드당 평균 ~6회의 반려·환원 핑퐁), 재심사도 Read 무제한이면 최초 심사와 같은 비용(재심사 p10 6.7만 토큰), 호출당 SSOT+어댑터 재열람 고정비 합계 ~10M 토큰. v0.9 는 이 세 지점을 겨눈다 — ① 역할 에이전트는 SSOT 대신 CORE.md(발췌 커널)만 읽고, ② 증분 회차는 대상 파일 밖 Read 금지(작성·심사 모두), ③ 같은 라운드의 결함은 한 번의 재실행으로 배치.

**커스터마이즈**: 어댑터 `ROLE_*` 로는 모델을 **바꿀 수 없다**(그 계층은 프롬프트 추가 지침일 뿐 frontmatter 에 못 닿는다). 모델/effort 변경은 §6b 말미의 **로컬 오버라이드** 경로로 — 해당 에이전트 `.md` 를 대상 레포 `.claude/agents/` 로 복사해 frontmatter 만 수정한다(플러그인 포크 불필요). 단 그 역할은 이후 플러그인 업데이트를 받지 못한다.

**강등(비용 절감) 가이드**:
- 강등 후보 = 실패가 시끄러운 역할부터: **implementer → explorer → designer** 순.
- 강등 금지 = 실패가 조용한 역할: **test-auditor·reviewer·architect·test-author** — 검사 깊이가 곧 mutation-kill-rate 다. 검사자(test-auditor·reviewer)의 effort 를 출제자·구현자보다 낮추지 마라.
- **opus 복귀 트리거**: 같은 카드에서 게이트(G) 실패가 2회 이상 반복되거나, mutation 생존이 구현 기인으로 확인되면 해당 역할을 opus 로 되돌려라.

## 7. 제약과 중단 조건

- 사람 게이트가 필요한 카드(human-visual)는 **완전 무인으로 done 이 되지 않는다** — 그게 설계다.
- 루프 상한: `MAX_ITERS`(N, 기본 10) 초과 · 같은 fail 2회 연속 동일 원인 미해결 · `spec_gap`/`test_dispute`(각 최대 2회) · GATE2 반려 2회 소진 → 중단·사용자 개입.
- 서브에이전트는 전부 포그라운드로 뜬다(진행이 실시간으로 보임). 완료된 에이전트의 SendMessage 재개·백그라운드 도구는 하네스 안에서 금지.
- 산출물(`ARTIFACTS_DIR`)은 gitignore 필수. 동결(`FROZEN_DIR`)은 레포 밖.
