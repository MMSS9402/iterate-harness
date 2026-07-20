---
description: 디자인 스펙 단독 패스 — designer 를 사이클 밖에서 1회 돌려 화면군의 visual_spec 을 산출/갱신하고 design_cache 에 등록한다. 이후 /iterate(design 카드)는 캐시 히트로 designer 를 스킵한다. 디자인이 확정된 프로젝트에서 사이클 비용을 상각하는 명령.
argument-hint: [화면군/카드 스코프 설명] [--refresh — DESIGN_SSOT 변경 반영 재산출, 선택]
---

당신(메인 Claude)이 **디자인 패스 드라이버**다. `${CLAUDE_PLUGIN_ROOT}/skills/iterate-protocol/SKILL.md`(SSOT)의 designer 규정 아래, **사이클 밖에서** 시각 스펙만 산출/갱신한다. 목적: 디자인이 확정된 프로젝트에서 매 design 카드마다 designer 를 돌리는 대신, 화면군 단위로 스펙을 한 번 만들어 두고 `/iterate` 가 캐시로 재사용하게 한다(SSOT §카드 플래그 design 의 스펙 캐시).

**인자**: $ARGUMENTS

## 절차
1. SSOT 와 `.claude/iterate.config.md` 어댑터를 Read. `DESIGN_SSOT`·`ARTIFACTS_DIR` 필수(없으면 중단·어댑터 보강 요청).
2. 인자에서 화면군 스코프를 해석한다(예: "리더보드+제출 목록 계열", 카드 ID 목록). `--refresh` 면 기존 `visual_spec.md` 가 있어도 재산출.
3. `$ARTIFACTS_DIR/design_cache.md` 를 Read — 스코프의 기존 항목과 `shasum <DESIGN_SSOT 경로>` 현재값을 대조. **이미 일치하고 `--refresh` 아님** → "캐시 유효 — 재산출 불요" 보고 후 종료(호출 0회).
4. `Agent(iterate-harness:designer, run_in_background: false)` ← 화면군 스코프 + `DESIGN_SSOT` 경로 + (있으면) 기존 visual_spec 경로("이 위에 델타로 갱신 — 전면 재작성 금지"). 산출을 당신이 `$ARTIFACTS_DIR/visual_spec.md`(화면군별 분리가 필요하면 `visual_spec_<화면군>.md`)로 Write.
5. `design_cache.md` 에 `[<화면군>] <visual_spec 경로> · <DESIGN_SSOT shasum> · <날짜>` 1줄 append(기존 항목은 갱신). designer Open Question(새 토큰 등)은 사용자에게 확인 후 반영.
6. 종료 보고: 산출/갱신된 스펙 경로·캐시 키·이후 `/iterate` 에서 이 화면군의 design 카드는 designer 스킵됨을 명시.

**주의**: 이 명령은 스펙 문서만 만든다 — 구현·테스트는 건드리지 않는다(read-only designer). `DESIGN_SSOT` 자체를 바꾸는 작업은 이 명령 몫이 아니다(그건 일반 편집 후 `--refresh` 로 반영).
