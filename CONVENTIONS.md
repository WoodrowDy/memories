---
title: 작성 규칙 (CONVENTIONS)
created: 2026-05-29
updated: 2026-05-29
tags: [meta]
---

# 작성 규칙

GitHub에서 읽히는 위키이자, 나중에 옵시디언 Vault로 그대로 쓰기 위한 규칙.
핵심 원칙: **폴더는 가볍게, 분류는 태그·링크로, 파일은 옮기지 않는다.**

## 두 개의 축

- **흐름**: `daily`(날것 TIL) → `topics`(정제된 위키). 익을수록 `status`가 올라간다.
- **주제**: os / db / lang / infra / cs / ai / tools. 폴더로 느슨하게 묶기만 한다.

## 프론트매터 (모든 노트 맨 위)

```yaml
---
title: PostgreSQL 인덱스
aliases: [PG 인덱스, postgres index]   # 다른 이름으로도 검색·링크
created: 2026-05-29
updated: 2026-05-29
tags: [db/postgresql]
status: seedling
---
```

- `status`: `seedling`(씨앗) → `growing`(자라는 중) → `evergreen`(위키로 굳음) → `archived`(끝/관심 밖)
- `archived`로 바꿔도 **파일은 옮기지 않는다.** 상태 변경은 한 줄 수정으로 끝.
- `aliases`: 한/영 혼용 검색용. 필요 없으면 비워둬도 됨.

## 네이밍

- 파일·폴더는 **영문 kebab-case** (`my-note.md`). 공백·한글 파일명 금지(URL·링크 안정).
- **한 개념 = 한 파일.** 커지면 쪼갠다 (`postgresql.md` → `postgresql/indexing.md`).
- 파일명에 날짜·draft 같은 휘발성 단어 금지 (위키 페이지는 경로가 고정 ID).
- 예외: `daily`는 `YYYY-MM-DD.md`.

## 링크

- **표준 마크다운 링크** 사용: `[인덱스](indexing.md)`. GitHub·옵시디언 둘 다 동작.
- `[[위키링크]]`는 GitHub에서 깨지므로, 옵시디언 전용이 되기 전까지 쓰지 않는다.
- 각 페이지 하단에 **`## 관련 문서`** 섹션으로 손수 연결(백링크 대체 + 큐레이션).

## daily → topics 승격

1. 배운 건 `daily/<연도>/<날짜>.md`에 빠르게 적는다.
2. 건질 내용은 `topics/<주제>/`에 새 페이지로 정제한다(daily 내용을 복사·다듬기).
3. daily 노트에서 그 topic 페이지로 링크. daily 파일은 저널로 그 자리에 남긴다.

## MOC (폴더마다 README.md)

- 각 폴더의 `README.md`가 그 영역의 **지도**다 (GitHub 자동 렌더 + 옵시디언 MOC).
- 익은 페이지부터 큐레이션해 링크한다. 모든 파일을 다 나열할 필요는 없다.

## PARA는 여기서만

- `projects/`(끝 있는 목표)와 `personal/`(언어 등 Area)의 **인덱스 페이지에만** 가볍게 적용.
- dev 지식(`topics`)은 PARA를 쓰지 않는다 — 주제로만 둔다.
- 즉 PARA는 "폴더 트리"가 아니라 "인덱스 몇 개의 상태 관리"로만 쓴다.
