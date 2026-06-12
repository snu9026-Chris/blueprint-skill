# /blueprint

Development workflow orchestrator. Runs the 7-phase pipeline:
PRODUCT → DESIGN → ARCHITECTURE → IMPLEMENT → CHECKPOINT → SHIP → POST-SHIP.

각 phase는 기존 gstack 스킬에 위임한다. 직접 일하지 않고 라우터 역할만.

## Quickstart

1. Antigravity에서 새 프로젝트 폴더 열기
2. `/blueprint` 호출
3. 스캐폴딩 확인 (A 선택)
4. **`.blueprint/state.md` 열고 Ctrl+Shift+V → 탭 고정 (필수)** — 진행 상황의 진실 원본
5. `/office-hours` 호출해서 PRODUCT.md 채우기
6. `/blueprint` 다시 호출 → Phase 1 진행

## Sub-commands

- `/blueprint` — init (state.md 없을 때) 또는 resume (있을 때) 자동 감지
- `/blueprint check` — Phase 4 중간 점검

## 생성되는 파일

```
프로젝트/
├── .blueprint/state.md         ← 진행도 + 카운터 + 트리거 (50줄 미만 유지)
├── docs/
│   ├── PRODUCT.md              ← What / Why / NON-GOALS
│   ├── DESIGN.md               ← Visual + screens
│   ├── ARCHITECTURE.md         ← Stack / domains / contracts
│   └── adr/                    ← Architecture Decision Records
├── plans/                       ← feature별 plan 파일
└── CLAUDE.md                   ← 위 파일들과 연결된 작업 규칙
```

## Phase → sub-skill 매핑

| Phase | Sub-skill | Output |
|---|---|---|
| 0 PRODUCT | /office-hours | docs/PRODUCT.md |
| 1 DESIGN | /design-consultation, /design-shotgun | docs/DESIGN.md |
| 2 ARCHITECTURE | /autoplan | docs/ARCHITECTURE.md, plans/*.md |
| 3 IMPLEMENT | (코드) + /investigate, /codex | source |
| 4 CHECKPOINT | /code-review + /retro | docs/adr/, checkpoint-*.md |
| 5 SHIP | /qa → /review → /ship | merged PR |
| 6 POST-SHIP | /land-and-deploy → /document-release → /retro | deployed |

## 자동 알람 (Phase 4 트리거)

다음 조건 충족 시 다음 `/blueprint` 호출 응답 최상단에 배너:
- `/ship` 5회 누적 (마지막 점검 후)
- 마지막 점검 후 14일 경과
- plans/*.md 3개 생성됐는데 ARCHITECTURE.md 안 읽음

`strict_mode: true`면 배너 대신 AskUserQuestion 블로킹.
`quiet_until: YYYY-MM-DD`로 한시적 음소거.

## 인지 과부하 방지 원칙 (내장)

1. 한 턴에 질문 하나만
2. 항상 RECOMMENDATION 먼저
3. Phase 진입 시 TodoWrite 자동 갱신 (보조)
4. state.md 50줄 미만 유지
5. 재개 시 첫 줄에 "Last session: ..." 한 줄
6. PRODUCT.md / ARCHITECTURE.md 없이 Phase 3 진입 차단

## 왜 extension 아니고 스킬인가

- gstack 철학(skills as markdown) 위에서 살아야 Antigravity native 통합 유지
- biz-plan-extension 같은 풀 webview 묶기엔 워크플로가 결정론적이지 않음
- 향후 안정화되면 그때 스캐폴딩 init만 미니 extension화 검토
