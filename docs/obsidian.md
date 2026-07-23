---
title: 옵시디언 연동 가이드
created: 2026-05-29
updated: 2026-07-23
tags: [meta, tools/obsidian]
---

# 옵시디언 연동 (실전 가이드)

이 저장소는 처음부터 옵시디언 Vault로 그대로 열리도록 설계됨. **마이그레이션 0.**
GitHub(저장·버전·공유) · 옵시디언(편집·그래프·플러그인) · (예정) AI 비서(질문·코치)가
**하나의 마크다운 소스**를 공유한다.

## 5분 시작

1. 레포를 `git clone` 한 폴더를 옵시디언에서 **Open folder as vault**로 연다.
2. 아래 설정을 맞춘다.
3. **Dataview** + **Obsidian Git** 플러그인 설치.
4. 끝. 편집 = git 커밋. 소스는 하나로 유지된다.

## 설정 (Settings)

**Files & Links**

- **Use [[Wikilinks]]: OFF** — 표준 마크다운 링크 유지(GitHub 호환). ← 제일 중요.
- **New link format: Relative path to file**.
- **Automatically update internal links: ON** — 파일명 바꿔도 링크 자동 수정.
- **Default location for new attachments: In subfolder under current folder → `media`** — 기존 `topics/cs/media`, `daily/media` 패턴과 일치.

**Editor**

- Readable line length: ON(기본) — 긴 노트 읽기 편함.

## 규칙 → 옵시디언 기능 매핑

| memories 규칙 | 옵시디언에서 |
| --- | --- |
| 프론트매터 (title·tags·status…) | **Properties** 패널로 편집 |
| `status: seedling→evergreen` | **Dataview**로 성숙도 대시보드 |
| 폴더별 `README.md` (MOC) | 그대로 지도 + Dataview로 목록 자동화 |
| 하단 `## 관련 문서` 링크 | **백링크 패널**·**그래프뷰**의 간선 |
| `aliases` | 별칭 검색·링크 |
| `tags: [db/postgresql]` | **태그 패널**·태그 검색 |
| `templates/` | **Templates / Templater**로 새 노트 |
| `daily/` | **Daily notes** 코어 플러그인 |

## 추천 플러그인 + 설정

**Obsidian Git** — GitHub 동기화 자동화 (핵심)

- Auto pull on startup: ON
- Auto commit-and-sync interval: 예) 10분 (또는 수동)
- 세밀한 커밋은 손으로 `commit-guide.md` 규칙(`notes(<scope>): …`)대로.
- → 옵시디언에서 적으면 알아서 pull/commit/push. GitHub·AI가 늘 최신을 본다.

**Dataview** — 프론트매터로 MOC·대시보드 자동화 (아래 예시).

**Templater / 코어 Templates** — `templates/topic-note.md`, `templates/daily-note.md`로 새 노트.

**Daily notes (코어)** — New file location: `daily/2026`, 템플릿: `templates/daily-note.md`, 형식 `YYYY-MM-DD`.

**Spaced Repetition** (또는 Anki 내보내기) — `personal/english`, `personal/spanish` 어휘 복습.

## Dataview 실전 예시 (복붙용)

카테고리 `README.md`(MOC)에 넣으면 목록이 자동 갱신된다.

**이 카테고리 노트 목록** — 예: `topics/db/README.md`

```dataview
TABLE status, updated
FROM "topics/db"
WHERE status != "archived"
SORT updated DESC
```

**키울 것 대시보드 (전체 seedling)** — 주간 리뷰용

```dataview
TABLE file.folder AS 폴더, updated AS 갱신
FROM "topics"
WHERE status = "seedling"
SORT updated ASC
```

**최근 손댄 노트 10개**

```dataview
TABLE status, updated
FROM "topics" OR "daily"
SORT updated DESC
LIMIT 10
```

**연결 없는(고아 후보) 노트** — `## 관련 문서`가 비어 있는 페이지

```dataview
LIST
FROM "topics"
WHERE length(file.outlinks) = 0
```

**성숙도 분포**

```dataview
TABLE length(rows) AS 개수
FROM "topics"
GROUP BY status
```

> DQL 키워드는 대소문자 무시(`table`/`TABLE` 무관). Dataview 블록은 **옵시디언에서만** 렌더되고 GitHub에선 코드로 보인다(아래 '호환' 참고).

## 백링크 · 그래프뷰 활용

- **백링크 패널**: "이 페이지를 가리키는 문서" 자동 확인 → `## 관련 문서` 큐레이션의 짝.
- **그래프뷰**: 고아 노트(아무도 안 가리키는 페이지)를 눈으로 찾아 연결. 로컬 그래프로 한 노트 주변만 보기도 가능.
- 루틴: 그래프뷰·Dataview로 고아·seedling 발견 → 링크 보강·승격 → 커밋.

## 워크플로 — 하나의 소스, 세 표면

```
옵시디언 (편집·그래프·복습)
   │  Obsidian Git = 자동 pull/commit/push
   ▼
GitHub memories (저장·버전·공유·렌더)
   │  공개 repo → 무인증 읽기
   ▼
AI 비서 (Slack): "○○ 있어?" · "어디 넣지?" 코치
```

편집은 옵시디언, 백엔드는 GitHub, 대화·코치는 AI. 같은 `.md` 하나를 셋이 공유한다.

## .obsidian/ 폴더

- 한 기기에서만 편집 → `.gitignore`에 둔다(이미 무시 처리).
- 여러 기기 동기화 → `.gitignore`에서 `.obsidian/` 줄을 지워 커밋(플러그인·설정 공유). 단 워크스페이스가 자주 바뀌면 커밋이 지저분해질 수 있음.

## GitHub 호환 유지 (중요)

어디서나(웹·모바일·AI) 읽히게 하려면 **공통분모 마크다운**을 지킨다:

- `[[위키링크]]` 대신 `[표준 링크](파일.md)` (설정에서 이미 OFF).
- 옵시디언 전용 문법은 GitHub에서 깨진다: 콜아웃 `> [!note]`, 주석 `%% ... %%`, 임베드 `![[...]]`.
- Dataview 블록은 GitHub에선 코드로 보임 → MOC는 **손 링크 몇 개 + Dataview 자동목록** 병행이 안전.
- 기준: "옵시디언에서만 예쁜 것"보다 "GitHub에서도 보이는 것"을 기본값으로.

## 모바일

- 옵시디언 모바일 + Obsidian Git(모바일 지원)으로 폰에서도 읽기·가벼운 편집.

## 나중에 공개 발행 (선택)

- **Quartz**: 옵시디언 Vault → 정적 사이트. 백링크·그래프까지 살려 위키로 발행.
- evergreen만 골라 발행하면 "디지털 가든"이 된다.

## 관련 문서

- [작성 규칙 (CONVENTIONS)](../CONVENTIONS.md)
- [Start Here — 운영 가이드](start-here.md)
- [커밋 & 발행 치트시트](commit-guide.md)
