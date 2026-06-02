---
title: 커밋 & 발행 치트시트
created: 2026-06-02
updated: 2026-06-02
tags: [meta]
---

# 커밋 & 발행 치트시트

새 글을 어디에 · 어떤 이름으로 · 어떻게 커밋·푸시하는지 한 장 요약.
규칙 근거는 [CONVENTIONS.md](../CONVENTIONS.md), 루틴은 [start-here.md](start-here.md).

## TL;DR — 새 글 3단계

1. 템플릿 복사해서 파일 생성 (`topics/<주제>/<영문이름>.md`)
2. 프론트매터 + 내용 채우기 → 그 폴더 `README.md`(MOC)에 링크 한 줄
3. `git add` → `git commit -m "notes(<주제>): <제목>"` → `git push`

---

## 1. 어디에 — 카테고리 ↔ 폴더

| scope | 폴더 | 넣는 것 (예) | tag 접두 |
| --- | --- | --- | --- |
| `os` | `topics/os/` | 프로세스/스레드, 스케줄링, 메모리, 리눅스 | `os/` |
| `db` | `topics/db/` | SQL, 인덱스, 트랜잭션, 격리수준 | `db/` |
| `lang` | `topics/lang/` | Java/TS, JVM, GC, 문법·런타임 | `lang/` |
| `infra` | `topics/infra/` | Docker, K8s, CI/CD, 네트워크, AWS | `infra/` |
| `cs` | `topics/cs/` | 자료구조, 알고리즘, 컴퓨터구조 (os·db에 안 맞는 일반 CS) | `cs/` |
| `ai` | `topics/ai/` | ML/LLM, 임베딩 등 | `ai/` |
| `tools` | `topics/tools/` | Git, IDE, 생산성 | `tools/` |
| `daily` | `daily/2026/` | 날짜별 TIL | `til` |
| `projects` | `projects/` | 끝이 있는 목표 | — |
| `personal` | `personal/` | 영어·스페인어 등 | — |

**1초 결정 규칙**: 더 구체적인 폴더가 이긴다. `os`가 있으면 OS 개념은 `cs` 말고 `os`로.
폴더는 통(bucket)일 뿐, 진짜 분류는 `tags`와 `## 관련 문서` 링크가 한다. **파일은 나중에 옮기지 않는다.**

## 2. 파일명 — 1초 결정

- **영문 kebab-case**: `process-vs-thread.md`, `transaction-isolation.md`
- 한 개념 = 한 파일 / **한글·공백·날짜 금지** (링크·URL 안정)
- 커지면 쪼갠다: `postgresql.md` → `postgresql/indexing.md`
- 예외: daily만 `daily/2026/YYYY-MM-DD.md`

## 3. 프론트매터 — 복붙용

**topics (개념 노트)** — `templates/topic-note.md`

```yaml
---
title: 제목
aliases: [한글별칭, english-alias]
created: 2026-06-02
updated: 2026-06-02
tags: [os/process]
status: seedling   # seedling → growing → evergreen → archived
---
```

**daily (TIL)** — `templates/daily-note.md`

```yaml
---
title: TIL 2026-06-02
created: 2026-06-02
tags: [til]
---
```

- `status`는 익을수록 한 줄만 바꾼다 (파일은 안 옮김).
- `aliases`는 한/영 검색용, 없으면 비워둬도 됨.

## 4. 커밋 메시지 규칙

형식: **`<type>(<scope>): <무엇>`**  (Conventional Commits 축약)

| type | 언제 |
| --- | --- |
| `notes` | 새 글 추가 / 내용 확장 |
| `fix` | 오타·오류 수정 |
| `move` | 위치 이동 / 이름 변경 |
| `docs` | MOC(README)·규칙·메타 문서 |
| `chore` | 설정(.gitignore, 템플릿 등) |

`scope` = 카테고리(os/db/lang/infra/cs/ai/tools/daily…).

예시:

```
notes(os): 프로세스 vs 스레드
notes(db): 트랜잭션 격리수준
fix(os): 데드락 예시 오타
move(os): cs → topics/os 로 이동
docs(os): MOC에 프로세스 링크 추가
chore: .idea gitignore 추가
```

## 5. 새 개념 글 올리기 — 복붙 명령

```bash
# 예: os 카테고리, 이름 my-note
cp templates/topic-note.md topics/os/my-note.md

# (편집) 프론트매터·내용 작성
# (편집) topics/os/README.md 의 목록에  - [제목](my-note.md)  추가

git add topics/os/my-note.md topics/os/README.md
git commit -m "notes(os): 제목"
git push
```

> 파일과 MOC 링크를 **한 커밋**에 같이 올리면 지도가 항상 최신.

## 6. daily(TIL) 올리기 — 복붙 명령

```bash
cp templates/daily-note.md daily/2026/$(date +%F).md
# (편집) 배운 것 적기, 건질 건 '승격 후보'에

git add daily/2026/$(date +%F).md
git commit -m "notes(daily): $(date +%F) TIL"
git push
```

`$(date +%F)` = 오늘 날짜(YYYY-MM-DD) 자동.

## 7. 자주 하는 후속 작업

- **승격(daily→topics)**: daily에서 건질 내용을 `topics/<주제>/`로 정제 → daily에 링크. `notes(<주제>): …`
- **상태 올리기**: `status: seedling → growing → evergreen`. 파일 안 옮김. `docs(<주제>): status growing`
- **링크 보강**: 하단 `## 관련 문서`에 **표준 마크다운 링크만** `[제목](파일.md)`. `[[위키링크]]`는 GitHub에서 깨지니 금지.

## 관련 문서

- [작성 규칙 (CONVENTIONS)](../CONVENTIONS.md)
- [Start Here — 운영 가이드](start-here.md)