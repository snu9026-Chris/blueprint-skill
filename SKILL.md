---
name: blueprint
description: |
  신규 프로젝트 워크플로 오케스트레이터. 0)INQUIRY → 1)PRODUCT → 2)DESIGN → 3)ARCHITECTURE →
  4)IMPLEMENT → 5)REVIEW → 6)SHIP → 7)POST-SHIP 의 8-phase 파이프라인을 관리한다.
  대부분 phase는 기존 gstack 스킬(/office-hours, /design-consultation, /autoplan, /qa,
  /ship 등)에 위임하지만, Phase 0 INQUIRY는 Claude가 직접 리서치·발견을 수행한다.

  사용 시점:
  - 새 프로젝트 시작: `/blueprint` (init mode 자동 감지)
  - 진행 중 재개: `/blueprint` (resume mode)
  - 중간 점검: `/blueprint check`

  Auto-scaffolds: docs/INQUIRY.md, docs/PRODUCT.md, docs/DESIGN.md, docs/ARCHITECTURE.md,
  docs/adr/, plans/, .blueprint/state.md, CLAUDE.md.
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - WebSearch
  - WebFetch
  - AskUserQuestion
  - TodoWrite
  - Skill
---

# /blueprint — Development workflow orchestrator

## 절대 원칙 — anti-cognitive-overload

이 스킬의 존재 이유는 *사용자 인지 과부하 감소*. 다음 위반 시 스킬 가치 사라짐:

1. **한 턴에 질문 하나만.** 5개 묶지 않는다.
2. **항상 RECOMMENDATION 먼저.** AskUserQuestion 첫 옵션에 추천안과 이유.
3. **Phase 전환 시 TodoWrite 자동 갱신.** 사용자가 손으로 안 함.
4. **state.md는 50줄 미만 유지.** 길어지면 세부는 docs/로 위임.
5. **재개 시 첫 줄은 "Last session: ..." 한 줄 요약.** 사용자가 기억 복원할 필요 없음.
6. **Phase 4(코딩=IMPLEMENT) 진입 전 PRODUCT.md / ARCHITECTURE.md 검사. 비어있으면 하드 차단.**

## 시각화 우선순위 — 중요

**진실 원본은 `.blueprint/state.md`.** Antigravity에서 Ctrl+Shift+V로 마크다운 프리뷰 띄우고 탭 고정. 영구적으로 보임.

보조로 TodoWrite를 호출하지만 일부 IDE 버전에서 렌더 안 될 수 있음. **TodoWrite에 의존하지 말 것 — state.md가 primary.**

스캐폴딩 직후 사용자에게 *반드시* 안내:
> `.blueprint/state.md` 를 열고 Ctrl+Shift+V → 탭 고정해두세요. 진행 상황은 그 파일이 진실 원본입니다.

## 모드 감지 (호출 시 첫 동작)

```bash
if [ "$BLUEPRINT_ARG" = "check" ]; then
  echo "MODE: check"
elif [ -f .blueprint/state.md ]; then
  echo "MODE: resume"
else
  echo "MODE: init"
fi
```

사용자가 `/blueprint check` 라고 입력 → CHECK. 없고 state.md 없으면 INIT, 있으면 RESUME.

---

## INIT MODE

### Step 1: 단일 확인

AskUserQuestion (한 질문만):
- 질문: "현재 폴더 `{현재경로}` 에 새 블루프린트를 만들까요? `docs/`, `.blueprint/`, `plans/`, `CLAUDE.md` 가 생성됩니다."
- A) (Recommended) 만들기 — Completeness: 9/10
- B) 취소

B면 종료.

### Step 2: 스캐폴딩

```bash
mkdir -p .blueprint docs/adr plans
PROJECT_NAME=$(basename "$PWD")
TODAY=$(date +%Y-%m-%d)
SKILL_DIR=~/.claude/skills/blueprint/templates
```

각 템플릿 복사 + 치환:
- `$SKILL_DIR/state.md.tmpl` → `.blueprint/state.md`
- `$SKILL_DIR/INQUIRY.md.tmpl` → `docs/INQUIRY.md`
- `$SKILL_DIR/PRODUCT.md.tmpl` → `docs/PRODUCT.md`
- `$SKILL_DIR/DESIGN.md.tmpl` → `docs/DESIGN.md`
- `$SKILL_DIR/ARCHITECTURE.md.tmpl` → `docs/ARCHITECTURE.md`
- `$SKILL_DIR/roadmap.md.tmpl` → `plans/roadmap.md`
- `$SKILL_DIR/adr/ADR-template.md` → `docs/adr/ADR-template.md`
- `$SKILL_DIR/plans/feature.md.tmpl` → `plans/_template.md`

각 파일에서 `{{PROJECT_NAME}}` → 실제 폴더명, `{{DATE}}` → 오늘 날짜로 치환.

**CLAUDE.md 처리 — 기존 파일 보호:**
- 루트에 `CLAUDE.md` 없으면 → `$SKILL_DIR/CLAUDE.md.tmpl` → `CLAUDE.md` 복사 + 치환
- 있으면 → 끝에 `## Blueprint integration` 섹션만 append (덮어쓰지 않음)

### Step 3: TodoWrite 초기화 (보조)

8개 항목:
- Phase 0: INQUIRY — `status: in_progress`
- Phase 1: PRODUCT — pending
- Phase 2: DESIGN — pending
- Phase 3: ARCHITECTURE — pending
- Phase 4: IMPLEMENT — pending
- Phase 5: REVIEW — pending
- Phase 6: SHIP — pending
- Phase 7: POST-SHIP — pending

TodoWrite 안 보일 수 있으니 state.md 프리뷰 핀 안내 반드시 같이.

### Step 4: 다음 액션 안내 출력

```
✅ 스캐폴딩 완료.

다음 단계:
1. Antigravity에서 `.blueprint/state.md` 열고 Ctrl+Shift+V → 탭 고정 (필수)
2. Phase 0 INQUIRY 바로 시작 — 아래 "Phase 0 INQUIRY" 절차대로 기본 질문 → 리서치 → docs/INQUIRY.md 초안
3. INQUIRY 확정되면 Phase 1 PRODUCT(/office-hours)로 진행
```

스캐폴딩 직후 바로 Phase 0 INQUIRY로 진입한다 (별도 호출 없이 이어서).

---

## RESUME MODE

### Step 1: state.md 읽고 첫 줄 요약 — 절대 원칙 #5

state.md 파싱 후 응답의 *맨 첫 줄*:
```
Last session: Phase {N} ({phase_name}) 완료. Next: {next_action_one_liner}.
```

### Step 1.5: INQUIRY 백필 (자동·필수 — 묻지 않는다)

RESUME 시 state.md Progress에 **INQUIRY phase가 없으면 무조건 자동으로 박아넣는다.** AskUserQuestion 없이 바로 편집. (INQUIRY는 파이프라인 0번 고정 phase — 옛 프로젝트라도 항상 존재해야 함.)

판정: Progress 줄들 중 `Phase \d+(\.\d+)?: INQUIRY` 가 하나도 없으면 백필 실행. 이미 있으면 아무것도 안 함 (idempotent — 매 resume마다 중복 추가 금지).

백필 절차 (state.md 직접 편집):
1. Progress 맨 위에 `- [ ] Phase 0: INQUIRY (백필)` 삽입. 체크박스는 **blank `[ ]`** (이미 진행 중이라도 공식 INQUIRY는 안 했으니 미완으로 둠 — 줄 존재가 핵심).
2. 기존 모든 phase 번호 **+1** (정수·소수 동일: `0→1`, `0.5→1.5`, `4.5→5.5` ...). 체크박스·이름·완료일·메타는 그대로 두고 **숫자만** 변경.
3. `## Next action` 등 본문의 phase 번호 참조도 +1로 맞춤 (있으면).
4. 한 줄 통지: "Phase 0 INQUIRY를 추가하고 나머지 phase를 +1 재번호했습니다."

이 백필은 대시보드 PHASES 패널이 곧바로 INQUIRY를 그리게 한다 (대시보드는 state.md를 동적으로 읽으므로 별도 작업 불필요).

### Step 2: 트리거 배너 (조건부)

state.md의 `## Triggers fired` 가 비어있지 않고 `quiet_until` 이 비어있거나 과거인 경우:

```
─────────────────────────────────
CHECKPOINT TRIGGER
조건: {fired conditions, comma-separated}
추천: /blueprint check
─────────────────────────────────
```

`strict_mode: true`인 경우 위 배너 대신 AskUserQuestion 사용 (블로킹):
- Q: "체크포인트 시점입니다 (strict 모드). 지금 점검?"
- A: 지금 점검 / B: 다음 phase 계속

### Step 3: 진입 질문 — 한 질문

AskUserQuestion:
- Q: "지금 무엇을 하실래요?"
- A) (Recommended) Phase {current} 계속
- B) 다른 Phase 점프
- C) /blueprint check 점검 모드

A → Phase delegation
B → AskUserQuestion follow-up (phase 0-7 중 선택)
C → CHECK MODE 진입

### Step 4: TodoWrite 동기화 (보조)

state.md의 진행도 읽어 TodoWrite 재구성. state.md ↔ TodoWrite 매핑:
- `[x]` → completed
- 현재 phase → in_progress
- 이후 → pending

---

## CHECK MODE — `/blueprint check`

`/blueprint check` 직접 호출 또는 RESUME → C 선택 시 진입.

### Step 1: 컨텍스트 스냅샷

```bash
git log --oneline -20 2>/dev/null || echo "no git history"
git diff --stat HEAD~5..HEAD 2>/dev/null || true
ls docs/adr/ 2>/dev/null
```

### Step 2: /code-review 위임

Skill 도구로 `/code-review` 호출. 결과 사용자에게 표시.

### Step 3: ADR 작성 여부 — 한 질문

AskUserQuestion:
- Q: "이번 점검에서 ADR로 기록할 결정이 있나요?"
- A) (Recommended) 있다 — 같이 작성
- B) 없다 — 건너뛰기

A면: ADR-{NNN}.md 생성. **한 번에 한 필드만 묻기** (Title → Context → Decision → Consequences 순). 절대 원칙 #1.

### Step 4: state.md 카운터 리셋

state.md 직접 편집:
- `ships_since_checkpoint` → 0
- `last_check` → 오늘 날짜
- `checkpoint_count` → +1
- `plans_without_arch_read` → 0
- `## Triggers fired` → (empty)

### Step 5: 체크포인트 기록 파일

`docs/checkpoint-{DATE}.md` 작성 (`$SKILL_DIR/checkpoint.md.tmpl` 사용).

---

## Phase 0 INQUIRY — 리서치 기반 발견 (Claude 직접 수행)

PRODUCT 앞단의 발견 단계. **office-hours 같은 서브스킬에 위임하지 않고 Claude가 직접** 리서치·종합해서 `docs/INQUIRY.md` 초안을 만든다. 목적: 사용자가 막연한 상태에서 시작해도, Claude가 레퍼런스·웹 리서치로 "이 앱에 필요한 것"의 초안을 먼저 제시 → 사용자는 *결정만* 하면 되게.

절차 (절대 원칙 #1 "한 턴에 질문 하나만" 유지):

1. **기본 3문** — 한 번에 하나씩:
   - 무슨 문제를 푸는 앱인가
   - 누가 쓰는가 (타깃 사용자)
   - 어떤 형태인가 (BUILD TARGET — 웹/확장/CLI/모바일 등 + 실행·배포 방식)
2. **레퍼런스 수집** — "참고하고 싶은 서비스·사이트·스크린샷 있어요?" 한 질문.
   - 있으면 → URL은 `WebFetch`로 분석, 스크린샷은 `Read`(이미지)로 분석
   - 없으면 → 건너뛰고 Claude가 3번에서 직접 찾음 (사용자에게 부담 X)
3. **웹 리서치** (`WebSearch`/`WebFetch`, Claude 자동) — 유사·경쟁 서비스, 도메인 베스트프랙티스, 흔한 기능셋·UX 패턴·함정. **출처 첨부 요약**. 가볍게 몇 건 (대규모 조사 X).
4. **항목 리스트업 — 초안 작성** → `docs/INQUIRY.md`:
   - **기능 후보**: must-have / nice-to-have / non-goal 후보 (각 항목에 "어느 사용자 니즈" 근거)
   - **러프 UX 플로우**: 주요 화면 + 화면 전환 한 줄 스케치 (상세 화면 설계는 Phase 2 DESIGN으로 넘김)
5. **검수 — 초안 통째 + 갈림길만 질문**: INQUIRY.md 초안을 통째로 보여주고, 꼭 정해야 하는 갈림길·이견 있는 항목만 한 개씩 AskUserQuestion. 나머지는 사용자가 문서에서 직접 고치게 둔다.
6. 확정 → state.md Phase 0 `[x]` → Phase 1 PRODUCT 한 줄 안내.

INQUIRY 결과(확정 기능·플로우)는 Phase 1에서 `/office-hours`의 입력으로 들어가 PRODUCT.md로 정식화된다.

## Phase delegation table — 진실 원본

| Phase | Sub-skill | Output | Hard gate |
|---|---|---|---|
| 0 | **Claude 직접** (리서치·발견) — `WebSearch`/`WebFetch`/`Read` | `docs/INQUIRY.md` | — |
| 1 | `/office-hours` (INQUIRY 초안을 입력으로 정식화·압박) | `docs/PRODUCT.md` | INQUIRY.md non-empty |
| 2 | `/design-consultation` + **UI Composition 인터뷰** | `docs/DESIGN.md` (UI Composition Decisions 섹션 채워짐) | PRODUCT.md non-empty |
| 3 | `/autoplan` (CEO+Design+Eng 묶음) | `docs/ARCHITECTURE.md` + `plans/*.md` | PRODUCT.md non-empty, **DESIGN.md UI Composition 비어있지 않음** |
| 4 | (코딩) — 필요시 `/investigate`, `/codex` | source code | **PRODUCT + ARCHITECTURE non-empty + UI Composition non-empty** |
| 5 | `/code-review` + `/retro` | `docs/adr/`, checkpoint 파일 | — |
| 6 | `/qa` → `/review` → `/ship` | merged PR | tests pass |
| 7 | `/land-and-deploy` → `/document-release` → `/retro` | deployed app | shipped |

### UI Composition 인터뷰 — Phase 2(DESIGN) 안의 필수 sub-step (Anti-게으른 디자인)

Phase 2(DESIGN)에서 디자인 시스템(색/폰트/스페이싱) 정한 후, 각 메인 화면마다:
1. PRODUCT.md JBT 목록 다시 읽기
2. 사용자에게 *그 화면을 어떤 상황에 어떻게 볼지* 직접 질문 (한 번에 하나)
3. 후보 컴포넌트 3-4개 추천 + 각각 어느 JBT 해결하는지 명시
4. 사용자가 빠진 거 추가 / 안 필요한 거 제거
5. DESIGN.md의 `## UI Composition Decisions` 표에 결정 박음

**금지**: 데이터 구조(state.md 섹션 등)를 그대로 UI 컴포넌트로 1:1 매핑. JBT 매핑 없는 컴포넌트는 박지 않는다.

각 Phase 실행 절차:
1. Hard gate 검사 (실패 시 정지)
2. Skill 도구로 sub-skill 호출
3. Sub-skill 결과 받으면 사용자에게 단일 질문: "이대로 `docs/{file}.md`에 저장할까요?"
4. 승인 → 저장 → state.md 갱신 → TodoWrite 갱신
5. 다음 phase 한 줄 안내

## Hard gate: Phase 4 진입 차단

Phase 4(코딩=IMPLEMENT)로 들어가기 직전 검사:

```bash
PRODUCT_OK=$([ -s docs/PRODUCT.md ] && grep -q -v "^>" docs/PRODUCT.md && echo yes || echo no)
ARCH_OK=$([ -s docs/ARCHITECTURE.md ] && grep -q -v "^>" docs/ARCHITECTURE.md && echo yes || echo no)
```

(주의: 위 grep은 코멘트 라인만 있는 비활성 템플릿 상태를 잡기 위함. 더 엄밀히 보려면 `## NON-GOALS` 섹션에 실제 항목이 있는지도 검사.)

`PRODUCT_OK=no` 또는 `ARCH_OK=no`면:
```
⚠️ Phase 4 차단됨.
이유: docs/PRODUCT.md ({PRODUCT_OK}) 또는 docs/ARCHITECTURE.md ({ARCH_OK}) 가 비어있음 (또는 템플릿 그대로).
조치: /blueprint 다시 호출 → 미완 Phase 먼저 채워주세요.
```

코드 작업 거부.

---

## Alarm 트리거 평가 — REVIEW/checkpoint 능동 알림

호출 시작 시 state.md 읽어 다음 평가:

| 조건 | 발동 시 동작 |
|---|---|
| `ships_since_checkpoint >= 5` | `## Triggers fired`에 한 줄 추가 |
| 오늘 - `last_check` >= 14일 | 같음 |
| `plans_without_arch_read >= 3` | 같음 |
| (선택) 단일 도메인 폴더 새 파일 ≥ 10 (`ls docs/../{domain}/* | wc -l` 비교) | 같음 |

발동된 조건 있으면 RESUME Step 2에서 배너 출력.

### Counter 갱신 규칙

수동 (CLAUDE.md.tmpl에 적혀 있음) — Claude가 다음 시점에 state.md 편집:
- `/ship` 성공 → `ships_since_checkpoint += 1`
- 새 plan 생성 시 ARCHITECTURE.md 읽지 않은 세션 → `plans_without_arch_read += 1`
- `/blueprint check` 완료 → 위 둘 리셋, `last_check` 갱신

(향후 hook으로 자동화 가능. 현재는 사용자가 /blueprint 재호출할 때마다 카운터 재평가.)

---

## state.md 스키마 — 정식

state.md 작성은 항상 이 형식:

```markdown
# Blueprint State — {project_name}

## Progress
- [x] Phase 0: INQUIRY (2026-05-21)
- [ ] Phase 1: PRODUCT
- [ ] Phase 2: DESIGN
- [ ] Phase 3: ARCHITECTURE
- [ ] Phase 4: IMPLEMENT
- [ ] Phase 5: REVIEW
- [ ] Phase 6: SHIP (0 ships)
- [ ] Phase 7: POST-SHIP

## Next action
Phase 1 시작 — /office-hours 로 INQUIRY 초안을 PRODUCT.md로 정식화.

## Counters
- ships_since_checkpoint: 0
- last_check: 2026-05-21
- checkpoint_count: 0
- plans_without_arch_read: 0

## Triggers fired
(empty)

## Settings
- strict_mode: false
- quiet_until: (empty)

## Decisions log
(상세는 docs/adr/)
- 2026-05-21: stack confirmed — Next.js + Python
```

50줄 넘으면 Decisions log를 docs/adr/로 잘라낸다.

---

## Idempotency — 같은 phase 재호출

같은 phase 두 번째 호출 시 AskUserQuestion 단일 질문:
- Q: "Phase {N}이 이미 완료되어 있어요. 어떻게 할까요?"
- A) 덮어쓰기 (이전 산출물 백업 후)
- B) Merge — 기존 + 새 내용 병합
- C) 취소

기본은 C. 백업은 `.blueprint/backup/{file}-{timestamp}.md` 로.

---

## 의도 불명 시

호출 의도가 명확하지 않으면 (예: 빈 폴더지만 부모에 다른 .blueprint 있음) 추측하지 말고 AskUserQuestion 단일 질문으로 확인.

---

## 종료 출력

각 phase/모드 완료 시 정확히 한 줄:
```
/blueprint {mode} 완료. state.md 갱신됨. 다음: {next}.
```
