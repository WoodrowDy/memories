---
title: Pi 패키지로 스킬을 연결하는 원리
aliases: [Pi 패키지, Pi 스킬, pi package, pi skills, pi install, 스킬 레포, my-pi]
created: 2026-11-16
updated: 2026-11-16
tags: [ai/agent, ai/pi, ai/skills, tools/git]
status: growing
---

# Pi 패키지로 스킬을 연결하는 원리 (Pi Packages & Skills)

> 스킬은 내 git 레포에 있고 두 PC의 Pi는 설치만 했는데, 어떻게 같은 스킬이 불러지는지 알고 싶어서 정리했다.

---

## 핵심

- 연결을 맡는 것은 Pi의 **패키지(Pi Packages)** 기능이다. 패키지 소스로 git 주소를 주면 레포 자체가 패키지가 된다.
- `pi install`은 레포를 `~/.pi/agent/git/` 아래에 클론하고, 그 주소를 `~/.pi/agent/settings.json`에 기록한다.
- Pi는 시작할 때 기록된 패키지 폴더에서 `skills/` 같은 정해진 폴더를 찾아 스킬의 이름과 설명만 읽는다. 호출되면 그때 `SKILL.md` 전체를 불러온다.
- PC끼리 직접 연결되는 것은 없다. 각 PC의 Pi가 각자 GitHub에서 받아 오고, GitHub가 가운데서 둘을 잇는다.

---

## 1. 구성 요소

### 패키지

Pi 패키지는 스킬, 프롬프트 템플릿, 익스텐션, 테마를 한 단위로 묶어 npm이나 git으로 나누는 방식이다. 레포 하나가 패키지 하나다. 익스텐션은 패키지에 담는 내용물의 한 종류일 뿐, 패키지와 같은 말이 아니다.

| 종류 | 폴더 | 정체 |
|---|---|---|
| 스킬 | `skills/` | `SKILL.md` 지침 문서 |
| 프롬프트 템플릿 | `prompts/` | 재사용하는 프롬프트 `.md` |
| 익스텐션 | `extensions/` | Pi 동작을 바꾸는 `.ts`/`.js` 코드 |
| 테마 | `themes/` | 화면 색상 `.json` |

`package.json`에 `pi` 항목이 있으면 거기 적힌 경로를 따르고, 없으면 위 폴더 이름으로 찾는다. 스킬만 있는 레포는 `package.json` 없이 폴더만으로 인식된다. 직접 설치해서 확인했다.

### 스킬

스킬은 `name`과 `description` 머리말이 있는 `SKILL.md`에 지침을 적은 것이다. 형식은 Agent Skills라는 공개 표준이라 다른 에이전트로도 옮길 수 있다.

```markdown
---
name: jira-ticket-format
description: 무엇을 하는 스킬인지, 언제 쓰는지
---

# 본문 지침
```

`description`이 없으면 스킬을 불러오지 않는다. `name`은 소문자·숫자·하이픈만 쓰고, 부모 폴더 이름과 같아야 한다. 다르면 경고만 내고 불러온다.

### 레포 구조

```
my-pi/
├── skills/
│   └── jira-ticket-format/
│       └── SKILL.md
├── prompts/
└── extensions/
```

---

## 2. 연결되는 흐름

### 설치

```
pi install git:https://github.com/<계정>/my-pi
      │
      ├─▶ ~/.pi/agent/git/github.com/<계정>/my-pi 로 클론
      │
      ├─▶ ~/.pi/agent/settings.json 패키지 목록에 주소 기록
      │
      └─▶ package.json 이 있으면 npm install
```

비공개 레포도 Pi가 따로 인증하지 않는다. Pi는 그 PC의 `git clone`을 실행할 뿐이라서, git이 레포에 접근할 수 있으면 Pi도 받는다. `gh`로 로그인해 두면 HTTPS 주소로 SSH 키 없이 받을 수 있다.

### 시작과 호출

```
pi 실행
      │
      ▼
settings.json 의 패키지 목록을 읽음
      │
      ▼
각 패키지 폴더에서 skills/**/SKILL.md 탐색
      │
      ▼
스킬 이름·설명만 시스템 프롬프트에 올림
      │
      ▼
/skill:jira-ticket-format 호출 또는 작업이 설명과 맞을 때
      │
      ▼
SKILL.md 전체를 읽어서 따름
```

처음부터 전체를 읽지 않는 이유는 컨텍스트를 아끼기 위해서다. 스킬이 수십 개여도 시작할 때 드는 비용은 이름과 설명 몇 줄이다.

### 수정과 갱신

```
PC A 작업 폴더에서 수정
      │
      ▼
git push → GitHub 원본
      │
      ├─▶ PC A: pi update --extensions → 사본 갱신
      │
      └─▶ PC B: pi update --extensions → 사본 갱신
```

고치는 곳은 따로 클론한 작업 폴더 하나다. Pi가 받아 둔 사본은 쓰는 곳이고, 쓰기만 하는 PC에는 작업 폴더가 필요 없다.

---

## 3. 왜 이 구조인가

npm으로도 패키지를 나눌 수 있지만, 그러려면 고칠 때마다 버전을 올려 `npm publish`를 해야 한다. git 소스는 push만 하면 된다. 혼자 쓰는 설정 레포에는 git 쪽이 맞다.

| | npm 소스 | git 소스 |
|---|---|---|
| 받는 곳 | `~/.pi/agent/npm/` | `~/.pi/agent/git/` |
| 배포 | `npm publish` | `git push` |
| 갱신 기준 | 배포한 버전 | 최신 커밋 |
| 고정 | 버전을 지정 | 주소 뒤에 `@v1.0.0` 같은 ref |

ref를 붙여 고정한 패키지는 갱신 대상에서 빠진다.

---

## 4. 헷갈리는 지점

**`pi update` ≠ 패키지 갱신.** Pi 1.5.2에서 `pi update`는 Pi 본체만 올린다. 패키지는 `pi update --extensions`, 둘 다는 `pi update --all`이다. 예전 문서에는 `pi update`가 패키지까지 갱신한다고 적혀 있어서 처음에 헷갈렸다. 설치된 버전에서 `pi update --help`로 확인하는 게 정확하다.

**`--extensions` ≠ 익스텐션만.** 옵션 이름과 달리 설치된 패키지 전체를 갱신한다. 스킬만 든 레포도 이 옵션으로 받는다.

**사본 폴더 ≠ 작업 폴더.** `~/.pi/agent/git/...`을 직접 고치면 다음 갱신 때 원격 상태로 덮인다. 갱신할 때 `HEAD is now at <커밋>`이 찍히는 걸 보면 원격 상태로 맞추는 방식으로 보인다.

**파일 이름은 `SKILL.md`여야 한다.** `skills/jira-ticket-format/jira-ticket-format.md`로 올렸더니 설치는 됐는데 스킬로 잡히지 않았다. 하위 폴더 안에서는 `SKILL.md`만 스킬로 찾는다.

**설치 주소는 이메일이 아니라 GitHub 사용자 이름.** 형식이 틀리면 Pi가 git 주소로 읽지 못하고 현재 폴더 안의 경로로 찾다가 `Path does not exist`를 낸다.

---

## 5. 심화

- 패키지를 프로젝트 단위로 설치하려면 `-l`을 붙인다. 그러면 `.pi/settings.json`에 기록된다.
- `pi config`로 설치된 패키지의 스킬·익스텐션을 하나씩 켜고 끌 수 있다.
- 남의 패키지는 코드를 실행할 수 있으니 설치 전에 내용을 확인한다.
- 공식 문서: [Pi Packages](https://pi.dev/docs/latest/packages), [skills.md (0.73.1)](https://cdn.jsdelivr.net/npm/@mariozechner/pi-coding-agent@0.73.1/docs/skills.md)

---

## 더 볼 것

- Pi를 시작할 때 패키지를 자동으로 갱신하는 설정이 있는지 확인하지 못했다.
- 갱신이 `git pull`인지 원격 기준 강제 리셋인지 문서로 확인하지 못했다. 위 내용은 출력을 보고 한 추정이다.
- 한 PC에서 push한 변경을 다른 PC에서 받는 것은 아직 직접 해 보지 않았다.

---

## 면접 단골 질문

### Q. 에이전트가 스킬을 처음부터 다 읽지 않는 이유는?
컨텍스트 비용 때문이다. 시작할 때는 이름과 설명만 올려 두고, 작업이 맞을 때 본문을 불러온다(progressive disclosure).

### Q. 여러 PC에서 같은 에이전트 설정을 쓰려면?
설정을 git 레포로 두고 각 PC가 같은 소스를 패키지로 설치한다. 인증 정보와 API 키는 레포에 넣지 않고 PC마다 따로 설정한다.

---

## 한 줄 정리

> Pi는 git 레포를 패키지로 클론해 두고, 시작할 때 그 안의 `skills/`에서 이름과 설명만 읽었다가 호출될 때 `SKILL.md`를 불러온다.

---

## 관련 문서

- [폐쇄형 vs 오픈웨이트 vs 오픈소스](ai-model-types.md)
- Pi 익스텐션 작성법
- Agent Skills 표준
