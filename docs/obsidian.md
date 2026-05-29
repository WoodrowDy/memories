---
title: 옵시디언 연동 방향
created: 2026-05-29
updated: 2026-05-29
tags: [meta, tools/obsidian]
---

# 옵시디언 연동 (준비되면)

이 저장소는 처음부터 옵시디언 Vault로 그대로 열리도록 설계됨. 마이그레이션 0.

## 시작

1. 레포를 `git clone` 한 폴더를 옵시디언에서 "Open folder as vault"로 연다.
2. 편집 = git 커밋. 소스가 하나로 유지된다.

## 설정 (Settings → Files & Links)

- **Use [[Wikilinks]]: OFF** — 표준 마크다운 링크 유지(GitHub 호환).
- **New link format: Relative path to file**.
- **Automatically update internal links: ON** — 파일명 바꿔도 링크 자동 수정.

## .obsidian/ 폴더

- 한 기기에서만 편집 → `.gitignore`에 둔다(이미 무시 처리됨).
- 여러 기기 동기화 → `.gitignore`에서 `.obsidian/` 줄을 지워 커밋한다.

## 추천 플러그인

- **Dataview**: 프론트매터로 MOC 자동 생성. 예 — `topics/db/README.md`에:

  ```dataview
  table status, updated from "topics/db" where status = "evergreen" sort updated desc
  ```

- **Templater / 코어 Templates**: `templates/`의 양식으로 새 노트 생성.
- **Spaced Repetition** 또는 Anki 내보내기: `personal/english`, `personal/spanish` 어휘 복습.
- **Daily notes (코어)**: 폴더를 `daily`로 지정.

## 활용

- **백링크 패널**: "이 페이지를 가리키는 문서" 자동 확인.
- **그래프뷰**: 고아 노트(아무도 안 가리키는 페이지)를 찾아 연결.

## 나중에 공개 발행

- **Quartz**: 옵시디언 Vault → 정적 사이트. 백링크·그래프까지 살려 위키로 발행.
