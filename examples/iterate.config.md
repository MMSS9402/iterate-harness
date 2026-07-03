# iterate.config.md — 프로젝트 어댑터 (템플릿 + 예시 2종)

> **이 파일이 코어(플러그인)와 프로젝트 특화의 경계다.** `/iterate` 드라이버가 Step 0 에서 이 파일을 Read 해 테스트/린트 커맨드·소스 경로·가드·불변식을 얻는다. 코어(명령·에이전트·SSOT)에는 프로젝트 고유 값이 하드코딩돼 있지 않다 — 전부 여기서 온다.
>
> **설치**: 이 파일을 대상 레포의 `.claude/iterate.config.md` 로 복사하고, 각 필드에서 **당신 스택에 맞는 예시 블록 하나만 남겨**(다른 건 삭제) 값을 맞춰라. 아래 두 예시(**A · Flutter(이 레포)** / **B · Go 백엔드**)는 스키마가 스택별로 어떻게 채워지는지 보여주기 위한 것이다.
>
> **드라이버 파싱 계약**: 드라이버는 `## FIELDNAME` 헤딩을 찾고, 그 아래 **첫 코드펜스(```)** 를 스칼라 값으로, **불릿/`### GUARD:` 하위블록**을 리스트 값으로 읽는다. **실제 어댑터에는 필드당 한 블록만** 둔다(템플릿은 예시가 둘이라 A 를 읽는다). `(선택)` 표시 필드는 비워도 되며, 비면 해당 기능이 꺼진다(design/e2e/원장/골든).
>
> **zsh 함정(가드·동결 커맨드 공통)**: `shasum $(cat filelist)`·`gofmt -l $(...)` 처럼 **명령치환 결과를 unquoted 확장**하면 zsh 는 기본적으로 단어분리를 안 해 파일명이 한 덩어리가 된다. 반드시 **xargs 파이프**로: `... | xargs shasum`, `... | xargs gofmt -l`.

---

## PROJECT
한 줄 소개 + 소스 루트(구현 코드가 사는 최상위 디렉터리).

**예시 A · Flutter(이 레포)**
```
독서교환 플랫폼 Flutter 모바일 앱(함께 읽기 리더). 소스 루트: lib/
```
**예시 B · Go 백엔드**
```
Go(chi) + MySQL/DocumentDB/Redis REST API 서버. 소스 루트: internal/
```

## TEST_CMD
테스트 실행 커맨드. 드라이버가 **영향 경로들을 인자로 덧붙여** 호출한다(전체 스위트 아님).

**예시 A · Flutter**
```
flutter test
```
**예시 B · Go**
```
go test -race
```
(★ 통합 테스트가 실 DB 를 요구하면 전제조건을 여기 명시하라 — 예: `사전조건: docker compose up -d mysql mongo redis` — 드라이버는 TEST_CMD 실행 전 전제조건 줄을 사용자에게 확인/기동한다. dial tcp connection refused 류 실패는 코드 결함이 아니라 환경 미기동이다.)

## LINT_CMD
정적 분석/컴파일 회귀 커맨드(전체). 항상 돈다.

**예시 A · Flutter**
```
flutter analyze lib test
```
**예시 B · Go**
```
go vet ./...
```

## BUILD_GEN_CMD (선택)
코드 생성 커맨드(모델/직렬화/목 등). 없으면 비워라(mutation 사본 복원·implementer 코드젠 단계가 생략된다).

**예시 A · Flutter**
```
dart run build_runner build --delete-conflicting-outputs
```
**예시 B · Go**
```
go generate ./...
```

## E2E_CMD (선택)
헤드리스에서 못 뜨는 표면(시뮬레이터/브라우저/풀스택)용 E2E. 드라이버가 `timeout` 으로 감싼다. 없으면 `e2e` 플래그를 쓸 수 없다. 주석에 **실행 환경 기동법**을 함께 적어라(드라이버가 미기동 시 기동 시도).

**예시 A · Flutter** (부팅된 iOS 시뮬레이터 필요 — `xcrun simctl boot <UDID>`)
```
sh -c 'DEV=$(xcrun simctl list devices booted | grep -oE "[0-9A-F-]{36}" | head -1); flutter test integration_test --timeout 120s -d "$DEV"'
```
**예시 B · Go** (선택 — 없으면 e2e 미사용. 통합 테스트가 있으면 빌드태그로)
```
go test -tags=e2e ./test/e2e/...
```

## TEST_SCOPE_RULES
영향 스코프 산정 규칙(산문). **디렉터리 미러 규칙** + **횡단 계층 목록**(변경 시 그것을 import 하는 상위 모듈 테스트까지 포함).

**예시 A · Flutter**
```
미러: lib/<path>/foo.dart 의 테스트는 test/<path>/foo_test.dart. 변경한 lib 디렉터리의 미러 test/ 를 직접 포함.
횡단 계층: lib/{shared,core,data,app}. 이들을 바꾸면 `grep -rl "<심볼>" lib/features` 로 의존 feature 를 찾아 그 test/features/<feature>/ 포함.
항상: 동결 5종 + `flutter analyze lib test`(전체).
```
**예시 B · Go**
```
미러: Go 는 _test.go 가 같은 패키지에 산다. 변경한 패키지의 `go test ./<pkg>` 를 직접 포함.
횡단 계층: internal/{domain,store,httpx,platform}. 이들을 바꾸면 `grep -rl "<심볼>" ./internal` 로 의존 패키지를 찾아 그 패키지 테스트 포함.
항상: 동결 스냅샷 + `go vet ./...`(전체).
```

## GUARDS
선언형 가드 목록. 각 가드는 `### GUARD: <이름>` + `- 커맨드:` + `- 통과:`(대개 "출력 비어야 함" 또는 "exit 0"). 드라이버가 순회 실행해 위반을 기록한다. **줄-상한 가드는 `FILE_LINE_LIMIT` 를 참조**한다.

**예시 A · Flutter**
```
### GUARD: 줄-상한
- 커맨드: find lib test -type f -name '*.dart' ! -name '*.g.dart' ! -name '*.freezed.dart' -exec wc -l {} + | awk '$2!="total" && $1>800 {print}'
- 통과: 출력 비어야 함 (모든 .dart ≤ FILE_LINE_LIMIT)

### GUARD: 색-리터럴-금지
- 커맨드: grep -rlE 'Color\(0x|Color\.from(ARGB|RGBO)' lib/features lib/shared --include='*.dart'
- 통과: 출력 비어야 함 (색은 theme 토큰만 — fromARGB/RGBO 우회 포함)

### GUARD: enum-name-직렬화-금지
- 커맨드: grep -rnE '[A-Za-z_][A-Za-z0-9_]*\??\.name\b' lib/data/models --include='*.dart' --exclude='*.g.dart' --exclude='*.freezed.dart'
- 통과: 출력 비어야 함 (enum 직렬화는 .wire 만; 식별자==wire 는 값테스트로 못 잡아 grep 백스톱)
```
**예시 B · Go**
```
### GUARD: 줄-상한
- 커맨드: git ls-files '*.go' | grep -v /vendor/ | grep -vE '(_gen|\.pb)\.go$' | xargs wc -l | awk '$2!="total" && $1>800 {print}'
- 통과: 출력 비어야 함 (xargs 파이프 — zsh 단어분리 회피)

### GUARD: gofmt-diff
- 커맨드: git ls-files '*.go' | grep -v /vendor/ | xargs gofmt -l
- 통과: 출력 비어야 함 (포맷 안 된 파일 0)

### GUARD: golangci-lint
- 커맨드: golangci-lint run ./...
- 통과: exit 0
```

## FILE_LINE_LIMIT
파일당 줄 상한(숫자). 줄-상한 GUARD 가 참조.

**예시 A · Flutter** / **예시 B · Go** (공통)
```
800
```

## FROZEN_DIR
동결 스냅샷 위치 — **반드시 레포 밖**(implementer 쓰기 범위 차단, P1 격리). 기본 `$HOME/.iterate-harness/<repo-slug>/frozen_tests`. `repo-slug` = git 최상위 basename(공백/특수문자→`-`): `basename "$(git rev-parse --show-toplevel)" | tr -c 'A-Za-z0-9._-' '-'`. git 밖이면 현재 디렉터리 basename.

**예시 A · Flutter**
```
$HOME/.iterate-harness/flutter-ebook-reader/frozen_tests
```
**예시 B · Go**
```
$HOME/.iterate-harness/ebook-api/frozen_tests
```

## ARTIFACTS_DIR
게이트 산출물 위치(verifier_raw.txt·reviewer_raw.txt·failures.md 등). **gitignore 필수**(커밋에 섞이면 안 됨). 기본 `.iterate/artifacts`.

**예시 A · Flutter** / **예시 B · Go** (공통 — `.gitignore` 에 `.iterate/` 추가)
```
.iterate/artifacts
```

## PLAN_PATH (선택)
카드 원장 경로. 설정 시 드라이버가 ID 로 카드를 찾고 완료기록을 갱신한다. **없으면 직접 작업 문자열 모드**(인자 문자열 자체가 작업, 원장 갱신 없음).

**예시 A · Flutter** (원장 사용)
```
harness/PLAN.md
```
**예시 B · Go** (원장 없음 — 직접 작업 모드. 이 필드를 비워둠)
```
(없음)
```

## DESIGN_SSOT (선택)
디자인 시스템 문서 경로. **`design` 플래그 카드에 필수** — 없으면 designer 가 진행 불가(어댑터 보강 요청). UI 없는 프로젝트는 비운다(design 플래그 미사용).

**예시 A · Flutter**
```
DESIGN.md
```
**예시 B · Go** (백엔드 — UI 없음, 비움)
```
(없음)
```

## PROJECT_INVARIANTS
프로젝트 불변식 목록(SSOT 불변식 2·3·3b·5 슬롯을 채운다). architect·implementer·test-author 가 따른다.

**예시 A · Flutter**
```
- Mock 우선: 추상 Repository + *Mock. 백엔드는 Phase 4 *Api. 컨트롤러는 provider 주입(전역 싱글턴 금지).
- 미렌더 표면 seam: InAppWebView/UiKitView/AndroidView 는 headless 미렌더 → seam+Fake 로 배선만, 실렌더는 e2e/사람게이트.
- enum 직렬화 .wire: enum_json_converters.dart 의 toJson 은 object.wire(절대 .name). [GUARD 백스톱]
- webview 브릿지 격리: flutter_inappwebview import 는 inapp_reader_web_view.dart 외 금지. 브릿지 채널명은 Dart↔JS 양쪽 일치.
- 토큰만: 색·간격·radius·모션은 app/theme 토큰. [색-리터럴 GUARD 백스톱]
```
**예시 B · Go**
```
- context 전파: 모든 요청 경로 함수는 첫 인자 ctx context.Context. 핸들러/스토어에서 context.Background()/TODO() 금지(요청 ctx 전파).
- 에러 랩핑: 경계 넘는 에러는 fmt.Errorf("...: %w", err) 로 랩. 상위는 errors.Is/As 로 분기(문자열 매칭 금지).
- 전역 상태 금지: DB/설정은 의존성 주입(패키지 전역 var 금지). 테스트가 격리 가능하게.
- SQL 파라미터화: 쿼리는 플레이스홀더 바인딩만(문자열 연결 금지 — injection).
- table-driven 테스트: 분기/경계는 표 기반 서브테스트로.
```

## TEST_FILE_GLOB
동결 대상 테스트 파일 패턴. 드라이버가 이 패턴으로 목록 스냅샷을 뜬다(find 루트+name 으로 변환; zsh `**` 확장 함정 회피).

**예시 A · Flutter** (find 변환: `find test -name '*_test.dart'`)
```
test/**/*_test.dart
```
**예시 B · Go** (find 변환: `find . -path ./vendor -prune -o -name '*_test.go' -print`)
```
**/*_test.go
```

## MUTATION_EXCLUDES
Mutation 게이트·줄-상한 가드에서 제외할 패턴(생성 파일 — 비커밋·재생성 대상이라 바이트 조작 부정확). 손으로 쓴 impl 만 변형 대상.

**예시 A · Flutter**
```
- *.g.dart
- *.freezed.dart
```
**예시 B · Go**
```
- *_gen.go
- *.pb.go
- mock_*.go
```

## HUMAN_GATE (선택)
`human-visual` 카드의 시각확인 방법 + (있으면) 골든/스냅샷 규약. 방법·골든 두 줄. 없으면 human-visual 플래그 미사용(순수 로직만).

**예시 A · Flutter**
```
- 방법: 부팅된 iOS 시뮬레이터에 `flutter run -d ios` 로 띄우고 스크린샷 캡처 → 사용자에게 제시.
- 골든: matchesGoldenFile 사용. 기준 이미지(--update-goldens)는 사람 시각확인 통과분에서만 생성(드라이버 임의 갱신 금지). 기계 게이트 스코프 제외.
```
**예시 B · Go** (백엔드 — 시각 없음. 필요 시 CLI/응답 확인만, 골든 없음)
```
- 방법: (선택) 로컬 기동 후 curl 로 엔드포인트 응답을 사용자에게 제시. 대개 순수 로직이라 human-visual 미사용.
- 골든: 없음.
```
