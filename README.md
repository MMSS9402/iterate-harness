# iterate — 멀티에이전트 TDD 하네스 (Claude Code 플러그인)

`/iterate` 는 한 작업(카드)을 **독립 서브에이전트 6종 + 메인 Claude 드라이버**로 TDD 사이클을 돌려 완성하는 하네스다. 핵심은 **게이트가 테스트라는 것** — 리뷰어의 도장이 아니라 *독립적으로 작성·동결된 테스트가 전부 통과하고, 그 테스트가 mutation-proof 임이 기계로 실증될 때* 통과한다.

이 플러그인은 **언어/프레임워크 무관 코어**(명령·에이전트·SSOT)와, 대상 레포마다 채우는 **프로젝트 어댑터**(`.claude/iterate.config.md`)로 나뉜다. 코어는 철학·격리·게이트 구조를 규정하고, 어댑터가 "무엇을 실행하고 무엇을 금지하는가"를 채운다.

## 구성

```
commands/iterate.md              # 드라이버(메인 Claude 가 구동). Step 0 에서 어댑터 로드
agents/{architect,designer,test-author,test-auditor,implementer,reviewer}.md   # 서브에이전트 6종(model: opus)
skills/iterate-protocol/SKILL.md # SSOT — 모든 역할이 매 task 읽는 불변식 단일 출처
examples/iterate.config.md       # 어댑터 템플릿(Flutter·Go 예시 2종)
```

## 설치

**개인 설치** (Claude Code 안에서):
```
/plugin marketplace add MMSS9402/iterate-harness
/plugin install iterate-harness@iterate-harness
```
- private 레포이므로 GitHub 인증이 필요하다 — `gh auth login`(HTTPS) 또는 SSH 키가 등록돼 있으면 그대로 동작한다. 백그라운드 자동 업데이트까지 원하면 `GH_TOKEN`(repo 스코프) 환경변수를 둔다.
- 설치 후 호출명은 **`/iterate-harness:iterate`** 다(플러그인 명령은 항상 네임스페이스). bare `/iterate` 를 원하면 대상 레포에 한 줄짜리 로컬 셔틀을 둔다:
  ```markdown
  <!-- .claude/commands/iterate.md -->
  ---
  description: iterate-harness 플러그인 드라이버 셔틀
  ---
  Skill 도구로 `iterate-harness:iterate` 를 args `$ARGUMENTS` 로 호출하라.
  ```

**팀 자동 설치** (레포에 커밋):
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

## 첫 실행 — 어댑터부터

1. **어댑터 작성**: `${CLAUDE_PLUGIN_ROOT}/examples/iterate.config.md` 를 대상 레포의 `.claude/iterate.config.md` 로 복사한다. 각 필드에서 당신 스택에 맞는 예시 블록 하나만 남기고(Flutter/Go 중), 값을 프로젝트에 맞춘다. 필수 필드: `PROJECT`·`TEST_CMD`·`LINT_CMD`·`GUARDS`·`FILE_LINE_LIMIT`·`FROZEN_DIR`·`ARTIFACTS_DIR`·`PROJECT_INVARIANTS`·`TEST_FILE_GLOB`·`MUTATION_EXCLUDES`·`TEST_SCOPE_RULES`. 선택: `BUILD_GEN_CMD`·`E2E_CMD`·`PLAN_PATH`·`DESIGN_SSOT`·`HUMAN_GATE`.
2. **산출물 gitignore**: `ARTIFACTS_DIR`(기본 `.iterate/artifacts`)를 `.gitignore` 에 추가한다(예: `.iterate/`). 게이트 산출물이 커밋에 섞이면 안 된다. `FROZEN_DIR` 는 레포 밖(`$HOME/.iterate-harness/...`)이라 gitignore 불필요.
3. **실행**:
   ```
   /iterate "T1.18"          # PLAN_PATH 설정 시 카드 ID. 반복 상한 기본 10
   /iterate "T1.18" 6        # N = 반복 상한(선택)
   /iterate "직접 작업 설명" 4   # PLAN_PATH 없거나 자유 작업
   ```
   드라이버는 Step 0 에서 어댑터를 Read 한다. **어댑터가 없으면 그 사실을 알리고 중단**하니, 위 1번을 먼저 해야 한다.

## 어댑터 작성 요령

- **TEST_CMD/LINT_CMD** 는 영향 경로를 인자로 받는 형태로 둔다(드라이버가 경로를 덧붙임). 전체 스위트를 매번 돌리지 않는다.
- **GUARDS** 는 `{이름·커맨드·통과조건}` 선언형이다. 줄-상한(FILE_LINE_LIMIT 참조)·리터럴 금지·fmt diff·정적분석 등 *기계로 0/1 판정 가능한 규칙*만 넣는다. gate 커맨드는 **xargs 파이프**로 써서 zsh 단어분리 함정을 피한다(`$(cat …)` unquoted 확장 금지).
- **PROJECT_INVARIANTS** 는 SSOT 불변식의 프로젝트 슬롯(2·3·3b·5)을 채운다 — 아키텍처 규약(DI·추상화 경계·테스트 더블 우선), 헤드리스 미렌더 표면 목록, 값 테스트로 못 잡아 GUARD 로 잡는 직렬화 규약 등.
- **DESIGN_SSOT** 가 없으면 `design` 플래그 카드는 진행 불가(designer 가 어댑터 보강을 요청). `E2E_CMD` 가 없으면 `e2e` 플래그 불가. `PLAN_PATH` 가 없으면 직접 작업 문자열 모드(원장 갱신 없음).
- 두 예시(Flutter 모바일 앱·Go 백엔드)가 UI 있는 스택과 없는 스택, 원장 있는/없는 운영을 모두 보여준다 — 당신 스택에 가까운 쪽을 시작점으로.

## 사이클 한눈에

```
0 Explore → A architect(설계+AC+플래그) → (design?) designer(시각 스펙)
→ T test-author(AC→독립 블랙박스 테스트) → G2 test-auditor(동결 전 엄격성 검사)→동결
→ B implementer(동결 테스트 통과) → G 게이트(테스트+LINT+GUARDS+E2E+Mutation)
   ├ 실패/생존 → architect 부터 전체 재실행 [iteration +1]
   └ 0 fail + 생존 0 → C reviewer(테스트가 놓친 결함 사냥→새 테스트)
→ 자동 green → (human-visual?) 사람 시각확인 1회 → done
```

## 하네스 철학 (왜 이렇게까지)

- **자가채점 금지(P1 격리)**: 구현자는 자기 결과를 채점하지 못한다. **출제자(test-author) ≠ 검사자(test-auditor·reviewer) ≠ 구현자(implementer)** — 셋은 별개 `Agent()` 호출이고, 검사자는 *명세에서 기대를 도출*한다(코드/테스트 추종 금지). 같은 모델이라도 컨텍스트·정보 차단으로 독립을 강제한다(모델 다양성에 기대지 않음).
- **"1회차 green" 은 품질이 아니다**: implementer 는 동결 테스트(정답지)를 보고 통과시키므로 1회차 green 은 거의 항진명제다. 얕은 테스트는 얕은 끼워맞추기를 통과시킨다(dead-green). 그래서 품질 지표는 회차 수가 아니라 **mutation-kill-rate** 다.
- **끼워맞추기 이중 백스톱**: 동결 **전** `test-auditor`(GATE2)가 "틀린 구현이 이 테스트를 통과할 수 있나"를 명세에서 적대 검사하고, 동결 **후** 드라이버 **Mutation 게이트**가 변경 impl 에 변형을 심어 테스트가 잡는지 기계로 실증한다. 둘 다 통과해야 "테스트에 맞추기 == 올바르게 구현하기"가 성립한다.
- **동결(frozen tests)**: 테스트는 레포 밖(`FROZEN_DIR`)에 파일목록+체크섬으로 동결된다. implementer 가 테스트를 수정·추가·삭제하면 드라이버가 불일치로 잡는다. **재동결은 매 G2 escapes-0 직후에만** — 정당한 추가(재실행 시)와 변조를 구별한다.
- **게이트 실패 = 전체 재실행**: 실패하면 콕 집어 라우팅하지 않고 architect 부터 파이프라인을 통째로 다시 돈다(계획 변경이 디자인·테스트·구현으로 연쇄하므로). 입력이 안 바뀐 단계는 산출을 그대로 둬 churn 을 막는다.
- **드라이버가 최종 판정**: 측정 항목(테스트 exit·LINT·GUARDS·mutation 생존·동결 체크섬·e2e)은 reviewer LLM 이 아니라 **드라이버가 직접 Bash 로 실행·판정**한다. reviewer 는 accept/reject 가 아니라 *테스트가 놓친 결함을 찾아 테스트로 환원*한다.

## 제약 — 사람 게이트가 필요한 카드

- **`human-visual`**: 자동 테스트가 못 보는 렌더/미감은 자동 green 후 **맨 끝에 1회** 사람이 본다(어댑터 `HUMAN_GATE` 방법). 그 전까지는 'verification passed'(완료기록 미확정), 사람 OK 후 'done'. 사람이 "투박/틀림" 하면 새 라운드로 환원.
- **`e2e`**: 헤드리스에서 못 뜨는 표면(webview·네이티브뷰·브라우저)은 `E2E_CMD` 로 기계 게이트에 포함하되, 실행 환경(시뮬레이터/브라우저)이 미기동이면 **자동 green 선언 금지** — 드라이버가 기동 시도, 불가면 중단·사용자 개입.
- **골든/스냅샷 기준**: 시각 회귀 기준 산출물은 **사람 승인분에서만** 생성(드라이버 임의 갱신 금지 — 잘못된 기준 박제 차단). 기계 게이트 스코프에서 제외.
- **루프 상한**: `MAX_ITERS`(인자 N, 기본 10) 초과·같은 fail 2회 연속 미해결·`spec_gap`·`test_dispute`(각 최대 2회)·GATE2 반려 2회 소진 시 중단·사용자 개입.
