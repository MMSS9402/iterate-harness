---
name: iterate-protocol
description: /iterate 멀티에이전트 TDD 하네스의 불변식 SSOT. 모든 역할(architect·designer·test-author·test-auditor·implementer·reviewer·explorer·드라이버)이 매 task 읽는다. 언어/프레임워크 무관 코어 — 프로젝트 특화는 대상 레포의 .claude/iterate.config.md 어댑터가 정의한다.
---

# Iteration Protocol — SSOT (모든 역할이 매 task 읽는다)

> 서브에이전트 7종 + 메인 Claude 오케스트레이션 하네스. **이 문서가 불변식의 단일 출처(SSOT)다.** architect·designer·test-author·test-auditor·implementer·reviewer·explorer·드라이버(iterate.md) 모두 이 규약을 따른다.
>
> **이 문서는 언어/프레임워크 무관 코어다.** 테스트/린트 커맨드·소스 경로·가드·불변식의 프로젝트별 구체값은 **대상 레포의 `.claude/iterate.config.md`(프로젝트 어댑터)** 가 정의한다. 이 SSOT 가 *철학·격리·게이트 구조*를 규정하고, 어댑터가 *무엇을 실행하고 무엇을 금지하는가*를 채운다. 아래에서 `$TEST_CMD`·`$LINT_CMD`·`$FROZEN_DIR`·`$ARTIFACTS_DIR`·`$GUARDS`·`$TEST_FILE_GLOB`·`$FILE_LINE_LIMIT` 등 대문자 토큰은 **어댑터 필드**를 가리킨다(드라이버가 Step 0 에서 Read 해 치환).
>
> **핵심 원칙: iteration 은 "리뷰어가 되돌렸나"가 아니라 "독립 테스트가 통과했나"로 돈다.** 테스트를 짜는 에이전트(test-author)가 모든 카드에 대해 테스트를 짜고, 그 테스트가 **전부 통과할 때까지** 루프를 돌며(=iteration 카운트), 전부 통과하면 그 때 **맨 끝에 단 한 번** 사람이 눈으로 본다. tier 구분은 폐기 — 대신 카드 플래그(`design`·`human-visual`·`e2e`·`explore`)로만 분기한다.
>
> **핵심 원칙 2 (독립성·끼워맞추기 차단): "회차 1"은 품질의 증거가 아니다.** TDD 에서 implementer 는 동결 테스트(정답지)를 보고 통과시키므로 1회차 green 은 거의 항진명제다(감사 실증: 1회차 비율이 높아도 mutation 생존율이 유의미하게 남아 dead-green 구간이 실재). 두 가지로 막는다: (1) **문제를 내는 test-author 와 그 깊이를 검사하는 test-auditor·reviewer 는 반드시 독립** — 같은 모델이라도 *컨텍스트 분리 + 명세에서만 도출 + 적대적 프레이밍*으로 강제(모델 다양성에 기대지 않는다). (2) **"정답지를 보고 끼워맞추기"가 통하지 않으려면 테스트가 mutation-proof 여야 한다** — 동결 전 **GATE2(테스트 엄격성 게이트)** + green 후 **Mutation 게이트**. 따라서 1차 품질 지표는 회차 수가 아니라 **mutation-kill-rate** 다.

## 프로젝트 어댑터 (코어와 프로젝트 특화의 경계)

이 하네스는 **코어(플러그인: 명령·에이전트·이 SSOT)** 와 **어댑터(대상 레포 `.claude/iterate.config.md`)** 로 나뉜다. 어댑터가 없으면 드라이버는 Step 0 에서 멈추고 어댑터 작성을 요구한다. 어댑터가 제공하는 값:

- `PROJECT` — 한 줄 소개 + 소스 루트(예: `lib/`·`internal/`·`src/`).
- `TEST_CMD` / `LINT_CMD` — 테스트·정적분석 실행 커맨드(영향 경로를 인자로 받는 형태). `BUILD_GEN_CMD`(선택, 코드생성)·`E2E_CMD`(선택).
- `TEST_SCOPE_RULES` — 영향 스코프 산정 규칙(디렉터리 미러·횡단 계층 목록).
- `GUARDS` — 선언형 가드 목록(각 {이름·커맨드·통과조건}). 800줄 상한·리터럴 금지·fmt diff 등.
- `FILE_LINE_LIMIT` — 파일당 줄 상한(숫자). 줄-상한 가드가 참조.
- `FROZEN_DIR` — 동결 스냅샷 위치(레포 밖). `ARTIFACTS_DIR` — 산출물 위치(gitignore).
- `PLAN_PATH`(선택) — 카드 원장. `DESIGN_SSOT`(선택) — 디자인 시스템 문서(design 카드 필수).
- `PROJECT_INVARIANTS` — 프로젝트 불변식(아래 §불변식의 2·3·3b·5 슬롯을 채운다).
- `TEST_FILE_GLOB` — 실행·동결 대상 테스트 파일 패턴. `TEST_SUPPORT_GLOBS` — 테스트 지원 파일(헬퍼·픽스처·Fake) 패턴(없으면 `(없음)`). `MUTATION_EXCLUDES` — 변형 제외 패턴(생성 파일 등).
- `HUMAN_GATE`(선택) — 시각확인 방법 + 골든/스냅샷 규약.
- `EXPLORE_QA`(선택) — `explore` 플래그 카드의 탐색 QA 방법: 실행 환경 기동법 · 조작 도구(playwright/시뮬레이터/curl 등) · 프로젝트 초점 영역. 없으면 `explore` 플래그를 쓸 수 없다.
- `ROLE_ARCHITECT`·`ROLE_DESIGNER`·`ROLE_TEST_AUTHOR`·`ROLE_TEST_AUDITOR`·`ROLE_IMPLEMENTER`·`ROLE_REVIEWER`·`ROLE_EXPLORER`(전부 선택) — **역할별 프로젝트 특화 행동지침(파인튜닝 계층)**. 각 역할 에이전트가 시작 시 자기 섹션을 Read 해 프롬프트에 **추가된 지침**으로 따른다(드라이버는 파싱하지 않고 전달도 안 함 — 에이전트가 직접 읽는다). **우선순위: 이 SSOT 의 불변식·격리·게이트 구조 > ROLE_* 지침** — 게이트를 약화시키는 지시(escapes 묵인·검사 생략·test/ 수정 허용 등)는 무시되고, 해당 에이전트가 산출물에 충돌로 보고한다. 프로젝트 성격에 따라 역할의 초점을 조정하는 용도다(예: 보안 API 서버 → reviewer 에 보안 최우선, 콘텐츠 서비스 → designer 에 타이포 위계 강조).

**규칙: 코어 문서(이 SSOT·명령·에이전트)에는 프로젝트 고유 커맨드/경로/금지어를 하드코딩하지 않는다.** 새 프로젝트 규칙이 생기면 어댑터에 추가한다.

## 카드 플래그 (tier 대체 — 카드마다 0~4개)

각 카드는 다음 플래그를 가질 수 있다. 없으면 순수 로직/데이터 카드(테스트만으로 검증). 드라이버가 원장 카드에서 읽거나(명시), 없으면 작업 성격으로 판정한다. **정본 순서 = 원장 카드 명시 > architect Card flags(Explore·설계 후 최다 정보 시점) > 드라이버 0b 잠정 판정(사이클 조기 차단용).**

- **`design`** — 새 시각 표면(화면·위젯·페이지)을 만든다 → **designer** 단계로 디자인 가이드 + `DESIGN_SSOT` 시각 스펙을 받는다. 순수 배선·로직만이면 생략. **`DESIGN_SSOT` 미설정이면 design 카드는 진행 불가** — designer 가 사용자에게 어댑터 보강을 요청한다.
- **`human-visual`** — 자동 테스트가 못 보는 렌더/미감을 사람이 눈으로 봐야 한다 → 자동 green 후 **사람 시각확인 게이트**(어댑터 `HUMAN_GATE` 방법으로·1회).
- **`e2e`** — 헤드리스에서 못 뜨는 표면(webview·네이티브뷰·브라우저·풀 시스템)이라 E2E 가 필요하다 → E2E(`E2E_CMD`)를 **기계 게이트**에 포함(루프 안). `E2E_CMD` 미설정이면 이 플래그는 사용할 수 없다(어댑터 보강 요청).
- **`explore`** — 사용자 흐름이 얽힌 화면·상태 많은 표면이라 **명세(AC)가 상상 못 한 결함**까지 실물 구동으로 탐색할 가치가 있다 → reviewer 갭 0 후·자동 green 확정 직전에 **explorer** 가 어댑터 `EXPLORE_QA` 방법으로 탐색 QA(시간상자·1회/green 후보). AC 기반 테스트는 명세 *안쪽*만 확장하므로, 명세 *바깥*(인접 흐름·이상 상태·중단/역행)은 이 단계가 메운다. `EXPLORE_QA` 미설정이면 사용 불가(어댑터 보강 요청).

## 사이클 (한 카드 = 한 /iterate)

```
[기계 루프 — 사람 없음]
0 Explore     →  작업 관련 모듈·재사용 후보 탐색
A architect   →  설계 + 인수기준(AC) + 카드 플래그 판정   ◄── 실패 시 여기로 올라와 전체 재실행
(design?) designer →  디자인 가이드 + DESIGN_SSOT 시각 스펙 (구현 전 = 구조를 한 번에 맞춤)
T test-author →  AC(+시각 스펙)를 독립 블랙박스 테스트로.
G2 test-auditor(독립) → 명세에서 "틀린 구현이 이 테스트를 통과할 수 있나" 적대 검사. 얕으면 test-author 로 반려(최대 2회). 통과해야 **동결**(구현자 불가침)
B implementer →  동결 테스트를 통과시키는 구현. 간결하게. 테스트는 절대 못 건드림
G 게이트(드라이버) →  test 0 fail + LINT + (e2e) E2E + GUARDS + **Mutation(변경 impl 변형→테스트가 잡나)**. 통과/실패만 잰다
     ├─ M개 fail OR mutation 생존 → A 로 올라가 **파이프라인 전체 재실행**(생존 변형은 test-author 가 잡는 케이스 추가)   [← iteration +1]
     └─ 0 fail + 생존 0 → C
C reviewer(독립) →  적대적 = "테스트가 놓친 결함" 사냥(명세에서 기대를 먼저 도출 → 반례 증거 제시) → 찾으면 새 테스트(→G2 재심사·재동결→G) 또는 정리 수정(→implementer→G)
(explore?) explorer(독립) →  명세 밖 실물 구동 탐색(어댑터 EXPLORE_QA — 인접 흐름·이상 상태·중단/역행·경계 밖 입력, 시간상자) → 결함은 재현 경로와 함께 새 테스트로 환원(→G2 재심사·재동결→G)

[여기서 자동 green 확정]

[사람 게이트 — 맨 끝, 단 한 번]
(human-visual?) 사람 시각확인 → "투박/틀림" 이면 A 로 올라가 새 라운드 / OK 면 done
```

| Phase | 역할 | 하는 일 |
|---|---|---|
| **0 Explore** | Explore(빌트인) | 재사용 후보·기존 패턴 → architect 컨텍스트 |
| **A architect** | architect | 설계(Context/Files/Reuse[file:line]/Interface/AC/Rationale/Open-Q) + **카드 플래그**. **게이트 실패 시 여기부터 전체 재실행**(실패 맥락으로 재계획, 아래 §게이트 실패) |
| **(design?) designer** | designer | 디자인 가이드 + `DESIGN_SSOT` Visual Design Spec → test-author·implementer 입력 |
| **T test-author** | test-author | **모든 카드**. AC(+시각 스펙)를 독립 테스트로 작성. 구현 전이라 red 정상 |
| **G2 test-auditor** | test-auditor | **독립 엄격성 게이트(동결 전)**. 명세에서 각 AC 마다 "틀린 구현(=구체 변형)이 이 테스트를 통과할 수 있나"를 적대적으로 검사. 통과해야 **동결**. 얕으면 test-author 반려(최대 2회). test-author 와 별개 에이전트(독립) |
| **B implementer** | implementer | 동결 테스트를 통과시키는 구현. **간결하게(불변식 10)**·**test/ 불가침**·범위 엄수 |
| **G 게이트** | 드라이버 | 테스트 + LINT + (e2e) E2E + GUARDS + **Mutation 게이트**(변경 impl 변형→생존 0) 측정. **통과/실패만 잰다 — 실패·생존이면 architect 부터 전체 재실행** |
| **C reviewer** | reviewer | **독립** 적대 = 테스트가 놓친 결함 사냥(명세에서 기대를 먼저 도출 후 반례 증거 제시). 결함→새 테스트(→G) / 중복·죽은코드→정리 수정(→implementer→G) |
| **(explore?) explorer** | explorer | `explore` 카드만. reviewer 갭 0 후 어댑터 `EXPLORE_QA` 방법으로 실물을 구동해 **명세 밖**(인접 흐름·빈/대량 데이터·중단·역행·연타)을 시간상자 안에서 탐색. 결함은 재현 경로와 함께 test-author 케이스로 환원(→G2→재동결→G). 탐색 스크립트는 `$ARTIFACTS_DIR/explore/` 에만(소모품 — 동결 대상 아님) |
| **(human-visual?)** | 사람 | 자동 green 후 1회 시각확인 → done 판정 |

> 간결화(polish)는 **별도 단계가 아니다** — implementer 가 처음부터 간결히 쓰고(불변식 10), 리뷰어가 중복/죽은코드를 발견하면 implementer 가 일반 수정으로 고친다(→ G 재실행).


## iteration 카운트 = 파이프라인 한 바퀴 회차

**"회차 X"는 [architect → designer → test-author → implementer → 게이트] 한 바퀴를 X번 돌았다는 뜻이다**(첫 게이트가 0 fail 이면 1회). 리뷰어 되돌림 횟수가 아니다. 완료기록에 `회차 X (테스트 N개 중 1회차 M fail → … → 0 fail)`처럼 **fail 추이**를 남긴다. 첫 회차 0 fail 도 정상(앞단 AC·테스트가 빡세면 구현이 한 번에 맞음) — 단 그 땐 test-author 테스트가 충분히 깊었는지 reviewer 가 더 적대적으로 본다.

## 게이트 실패 → 파이프라인 전체 재실행

게이트는 잴 뿐이고, **실패하면 콕 집어 라우팅하지 않고 처음(architect)부터 파이프라인을 통째로 다시 돌린다.** 테스트 실패는 한 곳만의 문제가 아닐 수 있다 — architect 가 구현 계획을 바꾸면 그에 따라 **디자인이 바뀌고 · 테스트 항목이 새로 추가/수정되고 · 구현이 바뀐다**(연쇄). 그러니 어느 단계를 콕 집지 않고 전부 다시 흘린다:

1. 드라이버가 `$ARTIFACTS_DIR/failures.md`(실패 테스트·메시지)를 남기고 **architect 재호출**(실패 맥락 포함). ★ **failures.md 스키마**: `## Failed tests`(테스트명 + 단언 실패 출력 원문 tail — 요약·재서술 금지) / `## Mutation survivors`(파일:변형 종류 + 주입 diff 1줄) / `## Gaps`(reviewer·explorer 산출물의 해당 항목 원문 발췌). **드라이버의 원인 진단·수정 제안 기입 금지** — 진단은 architect 몫(plan.md `### Notes`, architect.md §게이트 실패 3분류). architect 재호출 프롬프트의 '실패 맥락'은 failures.md 경로 전달로 한정하고, 추가 지시는 SSOT 가 명시한 것(reviewer 갭 시 동형 확장 지시)만. **reviewer 갭 재실행이면 동형 확장 필수** — architect 에게 "이 갭과 동형인 결함 표면(같은 원인·다른 소비처)을 전수 나열해 한 회차에 묶어라"를 명시(reviewer 가 같은 원인 계열 태그로 묶은 Notes 항목 포함 — 동형성 판정은 reviewer 태그를 따르고 드라이버는 태그 기반 기계 라우팅만 한다). 동형 표면을 회차마다 하나씩 발견하면 재파이프라인이 그만큼 반복된다(실측 회당 ≈1시간).
2. architect 재계획(바꾸거나 유지 — 단순 구현 버그면 계획은 유지하되 `### Notes` 섹션에만 'X가 버그'라 짚음) → (design 플래그면) designer → test-author(테스트 추가/수정/유지) → **G2 test-auditor 재심사(escapes 0 이어야) → 재동결** → implementer → **게이트**. ★ **test-author 가 케이스를 변경/추가했으면 재실행에서도 G2·재동결을 건너뛰지 않는다** — 그 추가분이 독립 심사를 통과해야 동결된다(끼워맞추기 백스톱이 재실행에서 빠지면 안 됨). test-author 산출이 직전과 완전히 동일할 때만 직전 escapes-0 판정·동결 스냅샷을 재사용(churn 금지, 아래 3).
3. **각 단계는 입력이 바뀌었을 때만 산출을 바꾼다** — architect 계획이 동일하면 designer/test-author 도 동일 산출(불필요한 재생성·테스트 churn 금지). ★ **'계획 동일'은 자가보고가 아니라 기계 판정이다**: 드라이버가 새 `plan.md` 를 직전 회차분(`plan.prev.md`)과 `diff` — 차이가 없거나 `### Notes` 섹션 안에만 국한되면 designer/test-author/G2 를 스킵하고 직전 동결 스냅샷을 유지하며, Notes 밖이 한 줄이라도 다르면 계획 변경으로 간주해 전체 흐름을 다시 흘린다. **단, 이번 재실행의 failures.md 가 mutation 생존·reviewer 갭·explorer 결함 환원을 담고 있으면 plan diff 가 Notes-only 여도 test-author→G2→재동결은 반드시 흘린다(스킵 허용은 designer 뿐).** Explore 의 코드 맵은 코드베이스가 그대로면 재사용(architect 가 새 영역 필요 시만 재탐색). test-author 산출이 동일하면 G2 도 직전 escapes-0 판정 재사용(churn 금지).

→ 한 바퀴 = iteration +1. 0 fail 까지(최대 N회). **이 '전체 재실행' 루프가 진짜 iteration 이다.** 드라이버는 추측으로 특정 에이전트만 부르지 않는다 — 재계획은 설계 맥락을 쥔 architect 가 하고, 그 아래가 그대로 흐른다.

## 독립 테스트 (모든 카드)

- **테스트는 항상 test-author 가 짠다.** implementer 는 인수 테스트를 못 짠다(자기 결과를 자기가 채점 금지 — 불변식 1). 구현자가 자기 테스트를 짜는 예외는 폐기.
- test-author 는 **AC + (있으면) Visual Design Spec** 에서 블랙박스로 짠다. **구현(소스 루트)을 읽고 거기 맞추지 마라** — 코드가 아니라 *명세*가 정답이다("테스트가 코드를 따라간다" 차단).
- 작성된 테스트는 **동결**: `$FROZEN_DIR` 에 **파일 목록 + 체크섬** 두 스냅샷(`$TEST_FILE_GLOB` ∪ `$TEST_SUPPORT_GLOBS` 매칭 파일 전체 열거 → filelist, 그 목록의 내용 → sha256). **글롭 둘의 역할 구분**: `TEST_FILE_GLOB` 은 '실행할 테스트' 열거용이고, 동결 범위는 test-author 가 쓸 수 있는 모든 것(헬퍼·픽스처·Fake 포함) — 그래서 `TEST_SUPPORT_GLOBS` 까지 합쳐 뜬다. test-author 가 `TEST_SUPPORT_GLOBS` 밖에 지원 파일을 만들면 드라이버가 반려한다(동결 밖 파일 = implementer 가 건드려도 못 잡는 구멍). **레포 밖에 두는 이유**: `$ARTIFACTS_DIR` 는 implementer(전체 도구) 쓰기 범위라 스냅샷 자체를 재생성해 게이트를 무력화할 수 있다(P1 격리) — 단 레포 밖 배치는 필요조건일 뿐(git-status 가시성 제거)이고, 기계 백스톱은 드라이버가 동결 직후 자기 대화 컨텍스트에 고정해 둔 스냅샷 다이제스트다(디스크는 전부 쓰기 가능하므로). implementer 가 테스트를 수정·추가·삭제하면 드라이버가 G 에서 목록/체크섬 불일치로 잡는다.
- **재동결 시점 = 매 G2 escapes-0 직후에만.** 재실행에서 test-author 가 정당히 케이스를 추가하면 그건 G2 통과 후 **재동결**(스냅샷 갱신)된다. 그 외(implementer 단계) 스냅샷은 **고정** — 이 고정 구간의 test 변경만이 implementer 변조다. 즉 동결 갱신을 G2 통과와 1:1로 묶어, 'test-author 정당 수정'과 'implementer 변조'를 구별한다(이 규칙이 없으면 정당 추가가 체크섬 거짓 양성을 부른다). **이 관문은 경로 불문이다** — 게이트 실패 재실행만이 아니라 **reviewer 갭 환원·test_dispute 수정·mutation 생존 케이스 추가 등 test 파일이 변경되는 모든 경로**가 G2 재심사→재동결을 거친다(G2 없이 스냅샷을 갱신하지 않는다).
- **자동으로 못 보는 것은 사람 게이트로**: 시각 미감(예뻐 보이나)·헤드리스 안 렌더는 테스트가 완전히 못 잡는다 → 골든(사람 승인분 기준)·E2E·사람 눈이 그 부분을 메운다. test-author 는 *기계로 검증 가능한 것*(분기·계산·네비·상태)을 최대한 깊게 짜고, 나머지는 플래그가 책임진다.

## 에이전트 독립성 (끼워맞추기 차단)

**모델 배치**: 검사자·출제자·설계자(architect·test-author·test-auditor·reviewer)와 explorer 는 **opus 고정** — 얕은 AC·얕은 케이스·느슨한 심사는 dead-green 으로 *조용히* 통과하는 지점이라 강등 불가. implementer 는 동결 체크섬·mutation·GUARDS·reviewer 라는 기계 백스톱이 품질을 강제하므로 **sonnet**(변경은 README §6c 로컬 오버라이드). 드라이버는 세션 모델을 상속한다 — 게이트 판정·mutation 변형 설계를 하므로 opus 세션 권장. 어느 조합이든 **격리는 모델 다양성이 아니라 구조(정보 차단·역할 분리·기계 백스톱)로 강제**한다 — 같은 모델은 같은 사각지대를 공유하므로, 모델이 같아도 달라도 이 구조가 유일한 방어다. **effort 도 역할 frontmatter 가 model 과 함께 고정한다** — 게이트 심사 깊이가 사용자 세션 effort 에 좌우되지 않게 하기 위함('어댑터/세션이 게이트를 약화시킬 수 없다' 원칙의 모델 파라미터 판). 검사자(test-auditor·reviewer)의 effort 를 출제자·구현자보다 낮추지 말 것 — 검사 깊이가 곧 mutation-kill-rate 다.

- **호출 형태(포그라운드 강제)**: 드라이버는 모든 역할을 매번 **새 `Agent()` 호출 + `run_in_background: false`** 로 띄운다(Agent 도구 기본값은 백그라운드). 완료 에이전트의 `SendMessage` 재개는 항상 백그라운드라 금지 — 반려/바운스도 새 호출로, 맥락은 프롬프트로 전달. `Workflow`·기타 백그라운드 도구도 이 하네스엔 금지.
- **문제↔채점 분리**: 테스트(시험문제)를 짜는 **test-author** 와, 그 깊이를 검사하는 **test-auditor**(동결 전)·**reviewer**(green 후)는 **반드시 다른 에이전트 호출**이다(한 에이전트가 자기 테스트를 자기가 합격시키는 것 금지 — P1 격리의 확장: **출제자(test-author) ≠ 검사자(test-auditor·reviewer) ≠ 구현자(implementer)**). 드라이버는 이들을 별개 `Agent()` 로 부른다.
- **명세에서만 도출(코드 금지)**: test-author·test-auditor 는 소스 루트 구현을 읽지 않는다. **AC/Visual Spec(+architect Interface 시그니처)이 정답**이고, "코드를 보고 거기 맞추기"는 테스트가 구현을 추종하게 만들어 끼워맞추기를 부른다. reviewer 도 *기대 동작을 명세에서 먼저 도출한 뒤* 테스트·구현을 본다(테스트를 먼저 믿지 않는다).
  - **도구 봉인 vs 지시문(정직한 구분)**: 진짜 도구 봉인은 **test-auditor**(Read/Grep/Glob)뿐이다. **reviewer 는 검증 실행에 Bash 가 필요해 도구로 수정을 막을 수 없다** — "수정 금지"는 지시문 강제이고, 드라이버가 reviewer 전후 **내용 포함 다이제스트** `{ git status --porcelain <경로>; git diff HEAD -- <경로>; git ls-files -o --exclude-standard -z <경로> | xargs -0 shasum; } | shasum` 비교로 무수정을 백스톱한다(상태 전용 `git status | shasum` 은 이미-dirty·untracked 파일의 *내용* 변조를 못 잡는다 — 회차 중 트리는 늘 dirty). **explorer 도 동일**(실물 구동에 Bash·탐색 스크립트에 Write 필요) — "소스 루트·테스트 디렉터리 무수정, 쓰기는 `$ARTIFACTS_DIR/explore/` 만"은 지시문 강제 + 드라이버가 같은 다이제스트 커맨드로 전후 비교 백스톱(`--exclude-standard` 덕에 gitignore 된 ARTIFACTS_DIR 는 자연 제외). **test-author** 도 Write/Edit 가 소스 루트에 미치므로 "test/ 만 쓴다"는 지시문 강제 — 드라이버가 소스 루트 경로에 같은 다이제스트 커맨드 전후 비교로 백스톱. "구현 안 봄·시그니처는 architect Interface 에서"도 지시문 강제이고, 빠져나간 추종은 동결 전 G2 + green 후 Mutation 게이트가 사후 차단한다(이중 백스톱).
- **정답지 가시성의 안전조건**: implementer 가 동결 테스트를 보는 것은 TDD 상 정상이다 — **단 그 테스트가 mutation-proof 일 때만 안전**하다. 그래야 "테스트에 끼워맞추기 == 올바르게 구현하기"가 성립한다. 이를 GATE2(사전 심사)와 Mutation 게이트(사후 실증)가 보장한다.
- **추론 누수 금지**: 드라이버는 역할 간에 *필요한 산출물*만 넘긴다 — AC·Interface·Visual 은 드라이버가 **spec.md 로 발췌**해 출제자·검사자에 전달(plan.md 원문 비노출 — Rationale·Notes 는 implementer·architect 재진입 전용), 그 외 Visual Spec·동결 테스트 경로·failures.md·(바운스 시) 검사자 산출물의 **발견 항목 리스트**만. 한 에이전트의 내부 추론·판정 서사(Verdict·Covered·제안 케이스)와 피검사자의 수정 설명은 다음 에이전트에 전달하지 않으며, G2 재검사에 직전 escapes 의 변형 설명을 덧붙이더라도 전체 재심사 지시가 우선한다.

## GATE2 — 테스트 엄격성 게이트 (동결 전, 독립)

test-author 가 테스트를 쓴 뒤, **동결·implementer 직전**에 `Agent(test-auditor)` 가 검사한다:

- 입력: AC(+Visual Spec) + test-author 가 쓴 테스트 파일. **구현은 없음(읽지도 않음)**.
- 핵심 질문(적대적): **"각 AC 에 대해, 그럴듯한 *틀린 구현*이 이 테스트를 통과할 수 있는가?"** — 통과할 수 있으면 그 구체적 변형(예: "delete 를 no-op 으로 둬도 통과", "저장 후 되읽기 단언이 없어 값이 뒤집혀도 통과")을 지목한다.
- 검사 순서: **테스트 열람 전** 명세에서 변형 목록을 먼저 고정하고(산출물 `## Mutant checklist` 로 기록), 그 뒤 테스트와 대조한다 — reviewer 의 '기대 먼저 도출'(§에이전트 독립성)과 동형.
- 판정: 빠져나갈 변형이 1개라도 있으면 **반려** → test-author 가 그 변형을 잡는 케이스 추가(최대 2회). 빠져나갈 변형 0 이어야 **동결** 진행. **반려 2회 소진 후에도 escapes ≥1 → 중단·사용자 개입**(AC 가 테스트 불가능하게 쓰였을 가능성 — architect 재계획 여부를 사용자와 결정). **AC defects ≥1 → test-author 반려가 아니라 architect 재계획으로 라우팅(반려 상한 2회에 합산).**
- 특히 사냥할 패턴(감사 실증): ① **상태 변이의 부수효과를 read 경로로 안 잡음**(create/update/delete/save/toggle 후 list/of/getter 되읽기 부재) ② **분기 한쪽만 잠금** ③ **성공/예외 단정만으로 동작 미검증**(no-op codify) ④ **불변식 미인코딩**(경계·프라이버시·원자성·상태기계 거부전이).

## Mutation 게이트 (green 후, 기계 실증)

green(테스트 0 fail) 이후, 드라이버가 **변경된 소스 impl 에 미묘한 변형을 주입하고 영향-스코프 테스트가 잡는지 기계적으로 확인**한다. "정답지를 봤든 안 봤든, 테스트가 틀린 구현을 떨어뜨리는가"를 모델 판단 없이 증명하는 최종 백스톱이다.

- 대상: 이 카드가 **변경/생성한 손으로 쓴 impl 파일**만(전 코드베이스 아님). **`MUTATION_EXCLUDES` 패턴(생성 파일 등) 제외** — 비커밋·재생성 대상이라 바이트 save/restore 가 불확실(생성 코드의 로직은 손으로 쓴 원본 쪽 변형으로 도달). test-auditor 가 지목한 고가치 지점(G2 산출물 `## Mutation targets` — escapes 0 이어도 필수 산출) 우선.
- 변형 예: 비교 `<`↔`<=`·경계 off-by-one·가드문 1줄 제거·보존 필드 `→ null`·상태변이 no-op·분기 조건 반전·필터 술어 제거.
- 한 번에 한 변형 → 영향-스코프 `$TEST_CMD` → **잡힘(fail)=kill / 통과(green)=생존** → 리셋 → 다음. 변형별 테스트 출력은 `$ARTIFACTS_DIR/mutlogs/` 로 리다이렉트하고 트랜스크립트에는 exit 마커만 남긴다(변형 N개 × 전체 로그의 드라이버 컨텍스트 유입 차단). **기본 = 격리 사본에서 수행**(rsync 로 작업트리를 scratchpad 에 복제 → (있으면)`$BUILD_GEN_CMD`/의존성 복원 → 사본에서 변형·테스트·리셋). 메인 작업트리는 미커밋 WIP 를 담고 있어 주입~원복 사이 세션 크래시가 사용자 코드를 오염시킨다 — **메인 트리 무수정이 원칙**. `git worktree add … HEAD` 는 미커밋 구현이 빠져 무의미(금지). rsync 불가 환경에서만 정확 바이트 save/restore + 주입 전 scratchpad 영구 백업. `git checkout/stash/clean` 금지.
- **INVALID 변형(중요)**: 변형이 **컴파일/로드 실패**를 일으키면 그건 **kill 이 아니라 무효(INVALID)** 다 — 모든 테스트가 컴파일 단계에서 실패해 "잡힌 것처럼" 보이지만, 이는 테스트 엄격성을 하나도 증명하지 못한다. INVALID 변형은 **세지 말고 폐기**하고, **컴파일은 되지만 동작이 달라지는 변형**으로 다시 주입한다(가드문 제거·경계 이동·no-op 화 등 타입-보존 변형 선택). kill 은 오직 *컴파일되는 변형을 테스트 단정이 fail 시킬 때*만 인정한다.
- 판정: **생존 변형 0 이어야 green 확정.** 생존이 1개라도 있으면 → **architect 부터 파이프라인 전체 재실행**(test-author 가 그 변형 잡는 케이스 추가 → **G2 재심사·재동결 포함** → implementer → 게이트) = iteration +1. 콕 집어 test-author 만 부르지 않는다(§게이트 실패와 동일 경로 — 끼워맞추기 백스톱이 재실행에서도 유지되게). 생존율이 곧 dead-green 척도다.
- 비용 관리: 카드당 변형 수는 변경 impl 의 분기/상태변이 지점에 비례해 소수(대략 ≤ 영향 파일당 2~4)로. 순수 데이터 카드는 가볍게, 상태기계·계산·필터 카드는 빡세게.

## 테스트 스코프 (영향 기반 — per-G green, 누적 전체 금지)

per-G `$TEST_CMD` 는 전체 스위트를 돌리지 않는다(누적될수록 느려지는 것 차단). **영향 집합**만(구체 규칙은 어댑터 `TEST_SCOPE_RULES`):

1. **직접** — 이 task 의 test-author 가 만든 테스트 + 변경한 소스 디렉터리의 미러 테스트.
2. **영향(전이 의존)** — 변경 파일이 횡단 계층(어댑터가 명시)이면 그것을 import 하는 상위 모듈의 테스트 포함(`grep -rl "심볼" <소스 루트>` → 의존 모듈 → 그 테스트).
3. **동결 무결성** — 동결 스냅샷(`$FROZEN_DIR`)은 **항상** 목록+체크섬 비교, 동결 테스트 실행.
4. **`$LINT_CMD`(전체)** — **항상**. 컴파일·타입 회귀 안전망.
5. **전체 회귀** — Phase 경계/명시 요청에서만.

드라이버는 영향 집합을 **경로로 명시**해 test-author/implementer/reviewer 에 전달한다.

## 사람 시각확인 게이트 (1회 — 터미널·시뮬레이터·브라우저 등 어댑터 HUMAN_GATE)

**사람은 루프 안에 들어가지 않는다**(매 G 마다 사람을 기다리면 병목). `human-visual` 플래그 카드는:
- **자동 green(§green-bar 의 전 항목 — mutation·guards·explore 포함 — 충족)이 확정된 뒤, 맨 끝에 단 한 번** 사람이 어댑터 `HUMAN_GATE` 방법(시뮬레이터/브라우저 스크린샷·CLI 출력 등)으로 본다. (항목은 §green-bar 가 정본 — 여기서 재나열하지 않는다.)
- 그 전까지 자동 green 은 **'verification passed'**(완료기록 `실기 시각확인 날짜: –`). 사람이 OK 하면 그 필드를 채워 **'done'**.
- 사람이 "투박하다/틀렸다" 하면 → 그건 **디자인/동작 테스트 실패**다 → 새 라운드(가능하면 그 지적을 test-author 새 케이스나 architect 재계획으로 환원해 다음엔 자동으로 잡히게).
- `human-visual` 없는 순수 로직 카드(직렬화·계산 등)는 사람 게이트 면제 — 자동 green = done.

## 시각 검증 — 골든/스냅샷 (기준 산출은 사람 승인분에서만)

- **골든/스냅샷 테스트**(위젯 이미지·UI 스냅샷 등, 어댑터 `HUMAN_GATE` 골든 규약이 있을 때만): 시각 회귀를 헤드리스·빠르게. **기준(baseline) 산출물은 사람 시각확인 통과분에서만** 생성(AI/드라이버 임의 갱신 금지 — 잘못된 디자인이 정답으로 박제되는 dead-green 차단). 그래서 골든 기준 생성은 사람 게이트와 묶인다.
- **기계 게이트 스코프에서 제외**: 골든/스냅샷 *기준 산출*은 Mutation·자동 재생성 대상이 아니다(사람 승인분에서만 산출). 헤드리스에서 못 뜨는 표면(브라우저·네이티브뷰 안)은 골든 불가 → E2E·사람 눈이 메운다.
- 어댑터에 골든 규약이 없으면 이 절은 해당 없음(생략).

## 불변식 (모든 역할 준수)

> **번호 체계는 고정**이다(교차 참조 안정). 프로젝트 특화 불변식(2·3·3b·5)은 **슬롯만 코어에 남기고 실제 내용은 어댑터 `PROJECT_INVARIANTS`/`GUARDS` 로 이동**한다 — 아래에 "→ 어댑터"로 표기.

1. **P1 격리(보편)** — 구현자는 자기 결과를 채점 못 한다. **모든 카드에서 implementer 는 test/ 를 못 건드린다**(test-author 가 짜고 동결). **출제자(test-author) ≠ 검사자(test-auditor·reviewer) ≠ 구현자(implementer)** — 셋은 별개 `Agent()` 호출이고, 검사자는 명세에서 기대를 도출한다(테스트·코드 추종 금지). 같은 모델이라도 컨텍스트·정보 차단으로 독립 강제(§에이전트 독립성).
2. **[프로젝트 불변식 — 아키텍처 규약]** → 어댑터 `PROJECT_INVARIANTS`. 의존성 주입·추상화 경계·레이어 규칙·테스트 더블 우선(예: Mock/Fake 우선, 실백엔드는 후속 Phase)을 어댑터가 정의한다. architect·implementer 는 어댑터 값을 따른다.
3. **[프로젝트 불변식 — 디자인 토큰/리터럴 규율]** → 어댑터 `GUARDS`(+`PROJECT_INVARIANTS`). 색·간격·radius·모션 등 매직 리터럴 금지 규칙과 그 grep 백스톱을 어댑터가 정의(예: 색 리터럴 금지 가드). designer 는 `DESIGN_SSOT` 토큰만 쓰게 스펙을 짠다.
3b. **[프로젝트 불변식 — 값 테스트로 못 잡는 지점의 grep 가드]** → 어댑터 `GUARDS`. 식별자==표현이라 값 테스트로 못 거르는 직렬화/매핑 규약(예: enum wire 값 강제)은 **테스트가 아니라 드라이버 grep 가드**로 잡는다. test-auditor 는 이를 test-author 에 떠넘기지 말고 "가드 몫"으로 지적한다.
4. **파일 줄 상한** — 모든 소스/테스트 파일 ≤ `$FILE_LINE_LIMIT`(어댑터, 예: 800). 드라이버 wc 백스톱(줄-상한 GUARD). 넘으면 architect 가 분할.
5. **헤드리스 미렌더 표면 seam(보편 커널) + [프로젝트 구체]** — 헤드리스 테스트에서 못 뜨는 표면(webview·네이티브뷰·GPU·브라우저)은 **직접 pump/렌더하지 말고 seam+Fake 로 배선만** 검증하고, 실렌더는 `e2e`/사람 게이트로 넘긴다. 무한 hang 방지 timeout 필수(E2E 는 드라이버가 `timeout` 래핑). **어떤 API 가 미렌더 표면인가**의 구체 목록은 어댑터 `PROJECT_INVARIANTS`.
6. **종단 배선** — '컨트롤러.X()가 하위.X() 부른다' 패스스루로 끝내지 말 것. **사용자 입력→실제 동작** AC 필수. 트리거 없는 고립 부품은 미완.
7. **운영 분기 일치** — 테스트는 앱/서비스 라우팅·DI 가 실제 타는 분기를 검증(반대 분기만 green 금지).
8. **no-op codify 금지** — '예외 없음+상태 유지'로 미구현 통과 금지. 미구현은 '부재' 또는 '준비중 표시' 단언.
9. **전역 부재 스냅샷 금지** — "트리/시스템 전체에 X 없음" 단언 금지. 이 task 산출물로 한정.
10. **간결성** — 동작 충족 최소 코드. 중복(DRY)·미사용 추상화(YAGNI)·과잉설계·이른 최적화·불필요 주석 금지. 기존 재사용 우선.
11. **부수효과는 read 경로로 단언** — 상태를 바꾸는 메서드(create/update/delete/save/toggle 등)는 성공/예외 단정만으로 끝내지 말 것. **변이 후 list/of/getter 로 되읽어** 실제 반영을 단언한다(no-op·반전·드롭 차단). test-author 책임, test-auditor·Mutation 게이트가 백스톱.
12. **회차 수는 비-신호** — '1회차'를 품질로 보고하지 말 것. 품질 지표는 mutation-kill-rate 와 사후정련/사람게이트 반려 횟수다. 완료기록에 reviewer 핑퐁·디자인 정련·사람게이트 반려를 **숨기지 말고 총 라운드로 합산** 기록.

## green-bar (드라이버 최종 판정 — 리뷰어 수락이 아님)

```
동결 전제 = GATE2(test-auditor) 통과 — 명세상 빠져나갈 변형 0 (얕으면 동결 불가, test-author 반려)

자동 green = test-author 테스트 0 fail (동결 파일목록·체크섬 일치, 마커 줄시작·정확히 1회)
            AND LINT 무경고 (ANALYZE_EXIT==0)
            AND (e2e 플래그 없음 OR E2E_EXIT==0)   ★ e2e 카드에서 실행 환경(시뮬레이터/브라우저) 미기동은 통과가 아니다 — 드라이버가 기동 시도, 불가면 중단·사용자 개입(생략 green 금지)
            AND line_ok (모든 파일 ≤ FILE_LINE_LIMIT, 드라이버 wc)
            AND guards_ok (어댑터 GUARDS 전부 통과조건 충족, 드라이버 grep/커맨드)
            AND mutation_ok (MUT_SURVIVED==0 마커 — verifier_raw.txt, 줄시작·정확히 1회. INVALID 변형 제외. 변경 impl 변형 주입 시 생존 0)
            AND reviewer 적대검토에서 '테스트가 놓친 실 결함' 0 (있으면 새 테스트→G 재진입)
            AND (explore 플래그 없음 OR explorer 탐색 결함 0)   ★ explore 카드에서 실행 환경 미기동은 통과가 아니다 — 기동 시도, 불가면 중단·사용자 개입(탐색 생략 green 금지)

done = 자동 green AND (human-visual 없음 OR 사람 시각확인 통과)
```
**측정 항목(테스트 exit·LINT·e2e·줄상한·GUARDS·mutation 생존·동결 체크섬)은 reviewer LLM 이 아니라 드라이버가 직접 실행·판정한다.** reviewer 의 역할은 accept/reject 가 아니라 *테스트가 놓친 것을 찾아 테스트로 환원*하는 것(게이트는 어디까지나 테스트).

## escape hatch / 루프 상한

- **spec_gap**(implementer 가 설계 불충분 보고) → architect 재진입.
- **test_dispute**(어떤 올바른 구현도 통과 못 하는 테스트) → test-author 가 해당 케이스만 수정, 최대 2회(구현자가 테스트 못 고침). 수정분은 **G2 재심사→재동결** 후 implementer 재개. **2회 소진 후에도 재신고 → 중단·사용자 개입**(AC-테스트 모순이 구조적일 가능성).
- **G 무한루프 방지** — 같은 fail 이 연속 2회 동일 원인으로 안 잡히면 중단·사용자 개입(architect 재계획이 원인을 못 짚은 것).
- **MAX_ITERS** — G 회차 하드 상한. `/iterate` 인자 N(맨 뒤 정수), 없으면 10. 초과 시 중단·사용자 개입.
- **transient(Agent 크래시)** — 드라이버가 해당 역할 1~2회 재호출.
- **E2E hang** — 드라이버가 `timeout`으로 감싸 결정적 차단(LLM 위임 금지).

## 산출물 위치

`$ARTIFACTS_DIR`(verifier_raw.txt·reviewer_raw.txt·failures.md·iteration_log.md·g2_verdicts.md·reviewer_verdicts.md·plan.md/visual_spec.md 등, gitignore). **동결 스냅샷만 예외로 레포 밖** `$FROZEN_DIR`(git-status 가시성 제거 — 필요조건일 뿐이고, 기계 백스톱은 드라이버 컨텍스트에 고정된 스냅샷 다이제스트다. P1 격리). `PLAN_PATH`(설정 시) 가 task 원장 — 완료기록이 결과 원장이고, 검사자 판정 원문은 `$ARTIFACTS_DIR` 의 `*_verdicts.md` 가 보존한다(판정 권한 이동 없음 — 기록 주체는 드라이버). 드라이버가 자동 green 후 갱신(human-visual 카드는 '실기 시각확인 날짜' 필드를 사람이 채울 때까지 'verification passed'). `PLAN_PATH` 미설정이면 원장 갱신 없이 종료 보고만 한다.
