# iterate — 멀티에이전트 TDD 하네스 (Claude Code 플러그인)

한 작업(카드)을 **독립 서브에이전트 6종 + 메인 Claude 드라이버**가 TDD 사이클로 완성하는 하네스다. 핵심 아이디어는 하나:

> **게이트는 리뷰어의 도장이 아니라 테스트다.** 독립적으로 작성·동결된 테스트가 전부 통과하고, 그 테스트가 mutation-proof 임이 *기계로* 실증될 때만 통과한다.

LLM 에게 구현을 맡길 때 가장 흔한 실패는 "자기가 짜고 자기가 통과시켰다고 주장"하는 자가채점이다. 이 하네스는 **출제자(테스트 작성)·검사자(엄격성 심사)·구현자를 서로 다른 에이전트로 분리**하고, 테스트를 **레포 밖에 동결**하고, green 이 된 뒤에도 **변형(mutation)을 심어 테스트가 정말 지키고 있는지**를 드라이버가 직접 실증한다.

플러그인은 **언어/프레임워크 무관 코어**(명령·에이전트·SSOT)와 대상 레포마다 채우는 **프로젝트 어댑터**(`.claude/iterate.config.md`)로 나뉜다. 코어가 철학·격리·게이트 구조를 규정하고, 어댑터가 "무엇을 실행하고 무엇을 금지하는가"(테스트/린트 커맨드·가드·프로젝트 불변식)를 채운다.

---

## 1. 구조

### 파일 구성

```
commands/iterate.md              # 드라이버 절차서 — 메인 Claude 가 이대로 오케스트레이션
agents/architect.md              # 설계자
agents/designer.md               # 시각 스펙 (design 카드 전용)
agents/test-author.md            # 테스트 출제자
agents/test-auditor.md           # 테스트 엄격성 검사자 (GATE2)
agents/implementer.md            # 구현자
agents/reviewer.md               # 적대적 갭 사냥꾼
skills/iterate-protocol/SKILL.md # SSOT — 모든 역할이 매 작업 읽는 불변식 단일 출처
examples/iterate.config.md       # 어댑터 템플릿 (Flutter 앱·Go 백엔드·React/TS 웹 예시 3종)
```

### 역할 7개 — 누가 무엇을 하고, 무엇을 못 하나

| 역할 | 하는 일 | 격리 장치 (못 하는 것) |
|---|---|---|
| **드라이버** (메인 Claude) | 카드 해석·에이전트 오케스트레이션·**게이트 측정·최종 판정**·완료기록 | 측정은 전부 Bash 직접 실행 — 모델의 "통과한 것 같다" 판단 배제 |
| Explore (빌트인) | 재사용 후보·기존 패턴 탐색 → architect 입력 | read-only |
| **architect** | 설계 + 인수기준(AC) + 카드 플래그 판정 | read-only. 코드 본문 작성 금지(GATE1 이 반려) |
| **designer** | 시각 스펙 (design 카드만, **구현 전**) | read-only. `DESIGN_SSOT` 없으면 진행 불가 |
| **test-author** | AC 를 독립 블랙박스 테스트로 — **명세에서만, 구현 안 봄** | 소스 루트 무수정(드라이버가 전후 git-diff 로 백스톱) |
| **test-auditor** | GATE2: "틀린 구현이 이 테스트를 통과할 수 있나" 적대 심사 | 유일한 진짜 도구 봉인(Read/Grep/Glob) — 구현도 안 읽음 |
| **implementer** | 동결 테스트를 green 으로 만드는 구현 | 테스트 수정·추가·삭제 불가(동결 목록+체크섬이 검출) |
| **reviewer** | green 후 "테스트가 놓친 결함" 사냥 → 새 테스트로 환원 | accept/reject 권한 없음. 무수정은 드라이버 git-status 백스톱 |

같은 모델(전부 opus)이 서로를 검사하므로, 격리는 모델 다양성이 아니라 **구조**(정보 차단 + 기계 백스톱)로 강제한다. 검사자는 언제나 *명세에서 기대를 먼저 도출*하고 코드·테스트를 본다 — "테스트가 통과하니 맞겠지"를 금지한다.

### 산출물 위치

```
대상 레포/
├── .claude/iterate.config.md    # 프로젝트 어댑터 (당신이 작성)
└── $ARTIFACTS_DIR/              # 게이트 산출물 (gitignore 필수)
    ├── verifier_raw.txt         #   드라이버 게이트 마커 (TEST_EXIT= 등)
    ├── reviewer_raw.txt         #   reviewer 검증 마커 (별도 파일 — 마커 충돌 방지)
    └── failures.md              #   실패 맥락 (재실행 시 architect 입력)

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
   └ GATE1(드라이버): 설계에 코드 본문 없음 + 예상 줄수 ≤ FILE_LINE_LIMIT
(design?) designer   시각 스펙 — 구현 전에 구조를 한 번에 맞춤
T test-author  AC → 독립 블랙박스 테스트 (구현이 없으니 red 가 정상)
G2 test-auditor  "틀린 구현이 빠져나갈 변형" 적대 심사
   ├ escapes ≥1 → test-author 보강 (최대 2회, 소진 시 중단·사용자 개입)
   └ escapes 0 → ★ 동결 (레포 밖 목록+체크섬 스냅샷)
B implementer  동결 테스트를 green 으로 (test/ 불가침)
G 게이트(드라이버 직접 Bash)
   동결 무결성 · 영향-스코프 테스트 · LINT 전체 · 줄-상한 · 어댑터 GUARDS · (e2e) E2E
   ├ 실패 → failures.md 기록 → architect 부터 전체 재실행   [회차 +1]
   └ 0 fail ↓
2b Mutation 게이트(드라이버 직접)
   격리 사본(rsync)에 변형 주입 → 미러 테스트가 잡는지 실증
   ├ 생존 ≥1 → architect 부터 전체 재실행 (test-author 가 그 변형 잡는 케이스 추가)  [회차 +1]
   └ 생존 0 ↓
C reviewer     테스트가 놓친 결함 사냥 (명세→기대 도출→반례 증거)
   ├ 갭 ≥1 → 결함은 test-author 케이스로 환원(G2 재심사→재동결 경유) → 게이트 재진입 / 중복·죽은코드는 implementer 정리
   └ 갭 0 → ★ 자동 green 확정

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
           AND 줄-상한 위반 0 AND 어댑터 GUARDS 위반 0
           AND MUT_SURVIVED==0                 (mutation 생존 0, INVALID 제외)
           AND reviewer 갭 0
done      = 자동 green AND (human-visual 없음 OR 사람 시각확인 통과)
```

---

## 3. iteration(회차)과 N — 횟수 지정

**회차(iteration) = [architect → (designer) → test-author → (G2) → implementer → 게이트] 파이프라인 한 바퀴.** 첫 게이트가 0 fail 이면 1회차다. 리뷰어가 되돌린 횟수가 아니다.

**N 은 회차의 하드 상한**이다. 지정 방법 — **인자 맨 뒤에 정수 하나**:

```
/iterate-harness:iterate "T2.7a"        # N 미지정 → 기본 10
/iterate-harness:iterate "T2.7a" 4      # N=4 — 4회차 안에 자동 green 못 만들면 중단·사용자 개입
/iterate-harness:iterate "로그인 실패 카피 개선" 3   # 직접 작업 문자열 + N=3
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

1. `examples/iterate.config.md` 를 대상 레포의 `.claude/iterate.config.md` 로 복사하고, 각 필드에서 당신 스택에 맞는 예시 블록(Flutter/Go/Web 중)만 남긴 뒤 값을 프로젝트에 맞춘다.
   - 필수: `PROJECT`·`TEST_CMD`·`LINT_CMD`·`GUARDS`·`FILE_LINE_LIMIT`·`FROZEN_DIR`·`ARTIFACTS_DIR`·`PROJECT_INVARIANTS`·`TEST_FILE_GLOB`·`MUTATION_EXCLUDES`·`TEST_SCOPE_RULES`
   - 선택: `BUILD_GEN_CMD`(코드생성)·`E2E_CMD`·`PLAN_PATH`(카드 원장)·`DESIGN_SSOT`(디자인 시스템)·`HUMAN_GATE`(시각확인 방법)
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

### 5-3. 카드 플래그 3종 (검증 방법이 갈린다)

| 플래그 | 언제 | 효과 |
|---|---|---|
| `design` | 새 시각 표면(화면·위젯·페이지)을 만들 때 | designer 단계 추가(구현 **전**에 시각 스펙). `DESIGN_SSOT` 필수 |
| `human-visual` | 자동 테스트가 못 보는 렌더/미감 | 자동 green 후 맨 끝에 **사람 확인 1회**. 그 전까진 'verification passed' |
| `e2e` | 헤드리스에서 안 뜨는 표면(webview·브라우저 등) | `E2E_CMD` 가 기계 게이트에 포함. 환경 미기동 = 통과 아님 |

플래그가 없으면 순수 로직 카드 — 자동 green 이 곧 done. 카드에 명시가 없으면 드라이버가 작업 성격으로 판정하고, 모호하면 1줄 물어본다.

### 5-4. 돌아가는 동안 사용자가 하는 일

기계 루프에는 사람이 없다. 개입 지점은 정확히 세 곳:
1. **designer Open Question** — 새 디자인 토큰 등 정책 결정이 필요할 때 드라이버가 물어온다.
2. **중단 보고** — MAX_ITERS 초과·동일 fail 2회·spec_gap·test_dispute 소진·GATE2 반려 소진 시.
3. **사람 시각확인** (human-visual 카드) — 자동 green 후 드라이버가 셀프검증 스크린샷과 체크리스트를 제시하면 실물을 보고 OK/지적. 지적은 새 라운드(가능하면 테스트 케이스로 환원)가 된다. **골든/스냅샷 기준 산출물은 이 승인 후에만 생성된다.**

### 5-5. 실전 사례 (원조 레포의 T2.7a)

Flutter 앱 "모임 합류 경로" 카드 — 이 하네스의 첫 실전:
- **회차 1**: 기계 게이트 0 fail(테스트 1,038) — 그러나 reviewer 가 "빈 로스터 공개모임에서 게이트가 무력화돼 둘러보기 4곳 중 3곳에서 기능이 안 보임"(GAP1)을 잡아 테스트 3건으로 환원 → red
- **회차 2**: 시드 보강으로 GAP1 해소, 0 fail(1,041)
- **사람 게이트**: 사용자가 "뒤로가기 버튼이 2개"(Scaffold 중첩) 발견 → AC51 쌍으로 환원 → **회차 3**: 구조 수정 후 0 fail(1,043) → 사람 OK → done
- 그 과정에서 G2 가 escape 2건을 동결 전에 막았고, implementer 는 테스트의 컴파일 결함을 `test_dispute` 로 정당하게 반려했으며, mutation 12/12 kill 로 마감. **기계가 한 결함을, 사람이 다른 결함을 잡았다 — 층이 다르면 그물도 달라야 한다는 것이 이 하네스의 요지다.**

---

## 6. 어댑터 작성 요령

- **TEST_CMD/LINT_CMD** 는 영향 경로를 인자로 받는 형태로 둔다(드라이버가 경로를 덧붙임). 전체 스위트를 매번 돌리지 않는다 — 전체 회귀는 Phase 경계에서만.
- **GUARDS** 는 `{이름·커맨드·통과조건}` 선언형이다. *기계로 0/1 판정 가능한 규칙*만 넣는다(리터럴 금지 grep·fmt diff 등). 값 테스트로 못 잡는 규약(예: 식별자==wire 인 enum 직렬화)이 GUARD 의 대표 용도다.
- **게이트 커맨드는 xargs 파이프로** — zsh 는 무인용 변수 확장을 단어분리하지 않아 `$(cat list)` 류가 한 덩어리로 붙는다.
- **PROJECT_INVARIANTS** 는 SSOT 불변식의 프로젝트 슬롯을 채운다 — 아키텍처 규약(DI·추상화 경계), 헤드리스 미렌더 표면 목록, 운영 분기 일치 규칙 등.
- **통합 테스트가 실 DB 를 요구하면** TEST_CMD 에 전제조건을 명시하라(예: `사전조건: docker compose up -d`). `connection refused` 류 실패는 코드 결함이 아니라 환경 미기동이다.
- **E2E 도구는 스택을 따라간다** — 모바일은 시뮬레이터(`flutter test integration_test`/simctl), 웹은 **Playwright**(`npx playwright test`, `webServer` 자동 기동), 백엔드는 빌드태그 통합 테스트. 헤드리스 단위 테스트(jsdom/widget test)가 원리상 못 보는 표면(실브라우저 렌더·webview·쿠키)을 `e2e` 플래그 + `E2E_CMD` 가 메운다.
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

- 지원 섹션 6종: `ROLE_ARCHITECT`·`ROLE_DESIGNER`·`ROLE_TEST_AUTHOR`·`ROLE_TEST_AUDITOR`·`ROLE_IMPLEMENTER`·`ROLE_REVIEWER`.
- **우선순위: SSOT 불변식·격리·게이트 구조 > ROLE_* 지침.** 게이트를 약화시키는 지시(escapes 묵인·test/ 수정 허용 등)는 에이전트가 무시하고 산출물에 충돌로 보고한다 — "무엇을 더 신경 쓸까"를 조정하는 계층이지 "무엇을 건너뛸까"가 아니다.
- 플러그인 업데이트를 받으면서 프로젝트별 특화를 유지할 수 있다(코어=플러그인, 특화=어댑터). 역할을 통째로 바꿔야 할 만큼 다른 프로젝트라면 에이전트 `.md` 를 프로젝트 `.claude/agents/` 로 복사해 수정하는 로컬 오버라이드도 가능하지만, 그 역할은 이후 플러그인 개선을 받지 못한다.

## 7. 제약과 중단 조건

- 사람 게이트가 필요한 카드(human-visual)는 **완전 무인으로 done 이 되지 않는다** — 그게 설계다.
- 루프 상한: `MAX_ITERS`(N, 기본 10) 초과 · 같은 fail 2회 연속 동일 원인 미해결 · `spec_gap`/`test_dispute`(각 최대 2회) · GATE2 반려 2회 소진 → 중단·사용자 개입.
- 서브에이전트는 전부 포그라운드로 뜬다(진행이 실시간으로 보임). 완료된 에이전트의 SendMessage 재개·백그라운드 도구는 하네스 안에서 금지.
- 산출물(`ARTIFACTS_DIR`)은 gitignore 필수. 동결(`FROZEN_DIR`)은 레포 밖.
