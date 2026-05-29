---
title: Start Here — 운영 가이드
created: 2026-05-29
updated: 2026-05-29
tags: [meta]
---

# Start Here

이 저장소를 어떻게 굴릴지 한 장에 정리. **할 일 → 매뉴얼 → 방향성** 순서.
규칙 상세는 [CONVENTIONS.md](../CONVENTIONS.md), 옵시디언 연동은 [obsidian.md](obsidian.md).

## 1. 할 일 (체크리스트)

**지금**
- [ ] 스캐폴드를 `memories` 레포에 합치고 첫 커밋/푸시
- [ ] `topics/db`(완성 예시)를 보고 위키 페이지 1개 직접 작성
- [ ] `daily`에 오늘 TIL 1개 적고 → topics로 1회 승격 연습

**자리잡으면 (1~2주 뒤)**
- [ ] 옵시디언으로 레포 폴더 열기 + 설정 3개 ([obsidian.md](obsidian.md))
- [ ] Obsidian Git 플러그인으로 auto-pull 켜기
- [ ] Dataview 깔고 `topics/db/README`에 자동 목록 한 줄 넣어보기

**나중에 (선택)**
- [ ] 공개 위키로 발행하고 싶으면 Quartz 검토
- [ ] active-note 워크플로가 정말 필요하면 그때 Obsidian MCP

## 2. 매뉴얼 (일상 루틴)

**새 TIL 적기**
`daily/2026/YYYY-MM-DD.md` 생성, `templates/daily-note.md` 양식 복사. 배운 거 빠르게.

**위키 페이지 만들기**
`topics/<주제>/<이름>.md` 생성, `templates/topic-note.md` 복사. 프론트매터 채우고 `status: seedling`. 하단 `## 관련 문서`에 연결.

**승격 (daily → topics)**
daily에서 건질 내용을 위키 페이지로 옮겨 다듬고, daily 노트에서 그 페이지로 링크. daily 파일은 그대로 둔다.

**익히기 / 보관**
내용이 단단해지면 `status`를 `growing` → `evergreen`. 안 보게 되면 `archived`. **파일은 절대 옮기지 않는다.**

**지도 갱신**
페이지가 쓸 만해지면 그 폴더 `README.md`(MOC)에 링크 추가.

**저장 (매번)**
```bash
git add .
git commit -m "notes: <무엇>"
git push
```

**주간 리뷰 (10분)**
seedling 중 키울 것 고르기, 고아 노트 연결, 안 쓰는 건 archived로. 위키는 가끔 솎아줘야 무덤이 안 된다.

## 3. 방향성 (원칙 5)

1. **빠른 기록 → 정제**: daily에 막 적고, 좋은 것만 topics 위키로 키운다.
2. **폴더는 통, 분류는 태그·링크**: 폴더를 깊게 파지 말고 `tags`와 문서 링크로 엮는다.
3. **파일은 안 옮긴다**: 성숙도는 `status`로 표현하고 위치는 고정 (링크 안 깨지게).
4. **MOC가 네비게이션**: 폴더마다 README가 지도. 다 나열 말고 좋은 것만 큐레이션.
5. **PARA는 personal/projects 인덱스에만**: dev 지식은 주제로만. 옵시디언·발행 같은 도구는 나중에 공짜로 얹는 레이어.
