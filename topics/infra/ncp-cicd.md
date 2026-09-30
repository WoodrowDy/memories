---
title: NCP로 이벤트 드리븐 CI/CD 만들기
aliases: [NCP 파이프라인, 이벤트 드리븐 배포]
created: 2026-09-16
updated: 2026-09-16
tags: [infra/cicd, infra/ncp, infra/deployment]
status: growing
---

# NCP 로 이벤트 드리븐 CI/CD 만들기

2026-09 · NestJS + pm2 + PostgreSQL · 서버 한 대에 dev · stage · prod 세 벌

NCP SourceCommit / SourceBuild / SourceDeploy / SourcePipeline 으로
"브랜치에 머지하면 알아서 배포되는" 구조를 만든 기록. 하루 걸렸고,
그중 설정에 쓴 시간은 한 시간쯤이다.

---

## 1. 네 서비스의 역할

NCP 는 CI/CD 를 넷으로 쪼개 놓았다. **이걸 먼저 이해하지 않으면 "어디를 고쳐야 하나"를 계속 틀린다.**

| 서비스 | 하는 일 | **안** 하는 일 |
|---|---|---|
| SourceCommit | git 저장소. PR · 머지 룰 | 아무것도 실행하지 않는다 |
| SourceBuild | 컨테이너에서 빌드 → zip | 서버에 접근하지 않는다 |
| SourceDeploy | 서버에 파일 놓고 명령 실행 | 빌드하지 않는다 |
| SourcePipeline | 위 셋을 엮고 **트리거를 건다** | 스스로 하는 일이 없다 |

### ★ 트리거는 파이프라인에만 있다

SourceBuild 설정에 "SourceCommit 연결" 칸이 있다. 그래서 빌드를 만들면 머지할 때
자동으로 돌 것 같은데 **아니다.** 그 칸은 *소스를 어디서 가져올지*일 뿐이다.

빌드만 만들어두면 영원히 아무 일도 일어나지 않는다. 여기서 한 번 헤맸다.

---

## 2. 전체 흐름

```
개발자 로컬
   │ Pull Request
   ▼
SourceCommit ──── develop 머지 = Push 이벤트 ────▶ SourcePipeline
                                                      │
                                    ┌─────────────────┴─────────────────┐
                                    │ 1단계                       2단계 │
                                    ▼                                   ▼
                              SourceBuild                        SourceDeploy
                         nodejs 22 컨테이너                     (에이전트 경유)
                         yarn install → build                          │
                                    │                                  │
                                    ▼                                  │
                            Object Storage ──── build.zip ─────────────┘
                                                                       │
                                                                       ▼
                                                              앱 서버 (admuser)
                                                              deploy.sh <env>
                                                                git reset --hard
                                                                yarn install
                                                                yarn build
                                                                pm2 reload
```

---

## 3. 환경 분리 설계

서버 한 대에 셋이 나란히 산다. **이 설계가 파이프라인보다 먼저다.**

| | dev | stage | prod |
|---|---|---|---|
| 브랜치 | `develop` | `stage` | `main` |
| 디렉터리 | `~/apps/<svc>-dev` | `~/apps/<svc>-stage` | `~/apps/<svc>-prod` |
| pm2 이름 | `<svc>-dev` | `<svc>-stage` | `<svc>-prod` |
| 포트 | 8080 | 8081 | 8082 |
| DB | 인스턴스 하나, **이름으로만 분리** | | |
| 파이프라인 트리거 | **자동 (Push)** | 없음 (수동) | 없음 (수동) |

### 빌드 산출물은 환경을 모른다

`dist/` 안에 dev 인지 prod 인지가 없다. 환경은 **배포 시점에** 이 사슬로 정해진다.

```
deploy.sh <env> 인자
  → DEPLOY_ROOT · APP_NAME · ENV_FILE
  → pm2 ecosystem 의 env: { NODE_ENV }
  → envs/.env.<env>
  → DB 접속 정보
```

덕분에 **빌드 명령어에 환경 분기가 없다.** 세 파이프라인이 다른 이유는 빌드가 달라서가 아니라
*어느 브랜치를 가져와 어느 배포 스테이지로 보내느냐*만 다르다.
이게 정리되고 나서야 파이프라인 복제가 5분 일이 됐다.

### 마지막 안전장치는 앱이 갖는다

```
NODE_ENV=dev 인데 운영 DB 에 붙으면        → 부팅 거부
대기 중인 마이그레이션이 있으면              → 부팅 거부
운영인데 외부 연계 계정이 비어 있으면         → 부팅 거부
```

DB 인스턴스가 하나이고 계정도 같다면 **이름 하나가 유일한 경계**다.
그 검사를 스크립트마다 복사하면 한쪽만 고쳐지고 다른 쪽은 뚫린 채로 남는다.
**앱이 부팅 때 스스로 거부하게 만드는 것**이 가장 얇고 확실하다.

---

## 4. 자동이냐 수동이냐

**dev 만 자동이다.** stage · prod 는 일부러 수동으로 둔다.

스키마가 바뀐 배포는 순서가 정해져 있다 — **마이그레이션 먼저, 그다음 배포.**
CI 는 이 순서를 모른다. 트리거를 걸면 머지하는 순간 배포가 먼저 출발한다.

다행인 건 앱이 **대기 중인 마이그레이션이 있으면 뜨기를 거부**한다는 것이다.
그래서 순서를 어긴 자동배포는 *조용히 성공하지 않고* 그 자리에서 빨갛게 실패한다.
서비스는 안 내려간다 — pm2 가 옛 프로세스를 물고 있다.

**"잘못된 순서가 조용히 성공하지 않게" 만들어 두는 것.** 이게 이 구조에서 제일 마음에 드는 부분이다.
CI 가 순서를 판단할 필요가 없어진다.

수동이어도 **빌드 실패 시 배포로 안 넘어가는 이점은 그대로 남는다.**

---

## 5. SourceBuild 설정

세 프로젝트가 **브랜치 하나만 빼고 완전히 같다.**

```
빌드 환경    운영체제    Ubuntu
             이미지      nodejs 22.14          ★ 관리 이미지의 최대 버전
             컴퓨팅      2 vCPU / 4 GB

소스          SourceCommit / <repo>
             브랜치      develop | stage | main   ← 여기만 다르다

환경변수      HUSKY=0

빌드 전 명령어
  npm i -g yarn@1.22.22
  yarn install --frozen-lockfile

빌드 명령어
  yarn build

빌드 후 명령어
  ls -l dist/src/main.js

빌드 결과물   업로드    사용
             백업      운영만 사용
             버킷/폴더  <bucket>/<repo>/<env>/
             파일명     build.zip
```

### 왜 이렇게 되었나

**yarn 이 관리 이미지에 없다.** node · npm · git · zip 정도만 있다. 컨테이너가 일회용이라
**매 빌드 설치**해야 한다. 버전을 안 박으면 yarn 4 가 깔리는데, yarn 4 는 `--frozen-lockfile` 대신
`--immutable` 을 쓰고 lockfile 형식도 달라서 저장소의 `yarn.lock` 과 안 맞는다.

**`HUSKY=0`.** `prepare: husky` 가 `yarn install` 때 자동으로 돈다. CI 컨테이너에는 git 훅을
걸 자리가 없어 실패하고, **`prepare` 실패는 설치 전체를 실패로 만든다.**

**`--frozen-lockfile`.** lock 을 고치지 말고 안 맞으면 실패하라는 뜻. 없으면 yarn 이 조용히
lock 을 갱신해서, 코드가 한 줄도 안 바뀌었는데 어제 통과한 빌드가 오늘 깨진다.
컨테이너는 매번 새로 만들어지므로 아무도 눈치채지 못한다.

**`ls -l dist/src/main.js`.** `nest build` 는 성공하는데 파일이 엉뚱한 데 생기는 일이 실제로 있었다
(`tsconfig` 의 `rootDir` 추론이 저장소 루트까지 올라가면 출력이 `dist/src/` 로 내려간다).
`ls` 는 파일이 없으면 비0 으로 끝나 **빌드가 그 자리에서 실패한다.** 있으면 크기와 시각이 로그에 남는다.
한 줄이 검사와 증거를 같이 한다.

**테스트는 뺐다.** 2 vCPU 컨테이너에서 jest 가 `os.cpus()` 로 **호스트** CPU 를 보고
워커를 열 개 넘게 띄워 OOM(SIGKILL)으로 죽는다. 되살릴 때는 `--maxWorkers=2`.
컨테이너 안에서 `os.cpus()` · `os.totalmem()` 을 믿는 도구는 전부 이 함정이 있다 —
cgroup 제한을 안 보고 호스트를 본다.

### 콘솔 함정 둘

**① 명령어를 고쳐도 되돌아온다.** "빌드 시작하기" 화면에서 고치면 **이번 실행에만** 적용된다.
프로젝트 설정 위저드로 들어가 **최종 확인까지** 눌러야 저장된다.

**② 결과물 저장이 기본으로 꺼져 있다.** 끈 채로 배포하면
"설정된 배포 파일이 존재하지 않아 배포 실행을 실패하였습니다" 가 뜬다.
메시지가 서버 쪽 경로 문제처럼 읽히는데 **배포가 아니라 빌드 문제**다.
빌드 요약의 `빌드 결과물 -` 과 "수집된 로그가 없습니다"로 확인한다.

**③ 결과물 백업.** 지정 경로와 **별도로** `sourcebuild_backup/<uuid>/` 에 한 벌 더 남긴다.
한 환경만 켜져 있으면 그 환경만 배포가 파일을 못 찾는다.
배포 파일 위치를 `Source Build` 로 두면 백업 경로와 무관해진다.

---

## 6. SourceDeploy 설정

```
프로젝트      <repo>
배포 대상     Server
스테이지      dev · stage · prod        ★ 예시는 dev/test/real 이지만 임의 이름도 들어간다

시나리오
  배포 전 명령어   (없음)
  배포 파일        위치   Source Build           ★ Object Storage 아님
                   대상   /home/<user>/apps/<svc>-<env>
  배포 후 명령어   실행 계정  <user>              ★ root 로 두면 안 된다
                   명령     CI_DEPLOY=true bash .../deploy.sh <env>
```

**실행 계정을 지정해야 하는 이유.** 에이전트는 root 로 돈다. 그대로 두면
① `$HOME` 이 `/root` 가 되어 버전 매니저(mise·nvm) 경로가 빗나가고
② pm2 데몬이 root 용으로 한 벌 더 떠서 원래 계정으로는 `pm2 status` 에 안 보인다.

### 에이전트 설치 — python2 문제

에이전트는 **서버에 직접 설치하는 데몬**이다. 없으면 SourceDeploy 가 서버에 아무것도 못 한다.
그리고 이게 가장 오래 걸린 부분이었다.

```
[Check Python Version] => Python 2.6+ needed
[Install Python]       => - ERROR : python install failed
```

**설치 스크립트가 python2 를 요구한다.** Ubuntu 24.04(noble) 저장소에는 python2 가 아예 없다.

여기서 크게 잘못 판단했다. 문서의 지원 OS 표에 "Ubuntu 22 이하"라고 적힌 걸 보고
*설치해보지도 않고* "이 서버엔 못 깐다, 크론으로 가자"고 결론 냈다.
실제로 해보니 **벽은 python2 하나뿐**이었고 표는 낡은 것이었다(24.04 에서 잘 돈다).

해결은 jammy 저장소를 **낮은 우선순위로** 붙여 python2 만 가져오는 것.
우선순위를 낮추는 게 핵심이다 — 안 그러면 다른 패키지까지 jammy 로 내려간다.

```bash
echo "deb http://archive.ubuntu.com/ubuntu jammy main universe" \
  | sudo tee /etc/apt/sources.list.d/jammy.list

sudo tee /etc/apt/preferences.d/jammy <<'EOF'
Package: *
Pin: release n=jammy
Pin-Priority: 100
EOF

sudo apt update && sudo apt install -y python2
```

> **교훈.** 문서의 호환성 표는 "테스트한 범위"지 "동작하는 범위"가 아니다.
> 30분이면 확인할 걸 문서만 보고 포기했다.

### `Failed to connect agent`

에이전트는 살아 있는데(`systemctl status` 가 `active (running)`) 배포가 이 메시지로 실패하면,
인증키 파일이 플레이스홀더 그대로일 가능성이 높다. 키는 `/opt/NCP_AUTH_KEY` 에 **평문**으로 들어간다.

**바이트 수로 바로 안다.**

```bash
wc -c /opt/NCP_AUTH_KEY
```

- **56 바이트** → 플레이스홀더. (`NCP_ACCESS_KEY=`15 + 플레이스홀더12 + 줄바꿈1) × 2 = 56
- **100 바이트 안팎** → 실제 키

키는 **서브 계정도 직접 발급할 수 있다** (`My Account → 계정 및 보안 관리 → 보안 관리 → 접근 관리`).
단 그 계정에 API Gateway Access 접근 유형이 있어야 메뉴가 보인다.
고친 뒤 `sudo systemctl restart sdagent`.

---

## 7. 비대화형 셸 함정

에이전트가 서버에서 스크립트를 실행하는데 `node` 를 못 찾는다.
SSH 로 들어가 같은 명령을 손으로 치면 잘 된다.

> **"손으로는 되는데 CI 에서만 안 된다" = 셸 환경 차이.**

우분투 `.bashrc` 는 맨 앞에 이게 있다.

```bash
case $- in
    *i*) ;;
      *) return;;    # ← 비대화형이면 여기서 끝
esac
```

버전 매니저 활성화가 그 아래에 있다. 에이전트가 띄우는 셸은 비대화형이라 거기 도달하지 못한다.

해결은 **배포 스크립트 안에서 PATH 를 직접 잡는 것**이다.

```bash
export PATH="$HOME/.local/share/mise/shims:$PATH"
```

선택지가 셋 있었고 각각 이유가 있다.

- **`mise activate` 를 쓴다** → 안 된다. `mise` 실행파일이 PATH 에 있어야 하는데, 지금 문제가 그 PATH 다.
- **설치 경로를 직접 박는다** → node 를 올릴 때마다 여기도 고쳐야 한다.
- **shims 를 쓴다** → 버전 파일을 보고 알아서 고른다. 버전이 바뀌어도 이 줄은 그대로다. ✓

**콘솔이 아니라 스크립트에 넣는다.** 콘솔 설정에는 git 이력이 없다. 누가 언제 왜 바꿨는지
남는 곳이 스크립트뿐이고, 환경 세 곳에 같은 걸 복사할 필요도 없어진다.

---

## 8. 자기 자신을 고치는 스크립트

배포 스크립트에 수정을 넣고 푸시했는데 **또 같은 에러가 났다.**

**bash 는 스크립트를 다 읽고 실행하지 않는다.** 파일 디스크립터와 오프셋을 들고
한 덩어리씩 읽어가며 실행한다. 그리고 파일이 바뀌는 방법은 두 가지다.

- **제자리 수정** (같은 inode): 오프셋이 엉뚱한 데를 가리켜 문법 오류가 난다. 위험.
- **갈아치우기** (rename, 새 inode): 옛 파일은 이름표만 떨어지고, **열고 있는 프로세스에겐 그대로 살아 있다.**

`git reset --hard` 는 후자다. 실제로 재현해 보면 이렇다.

```
[1] 스크립트 시작 — 지금 실행 중인 판: 옛날판
[2] inode: 802916
    ← (여기서 git reset --hard 실행)
[3] sleep 끝. 나는 여전히 옛날판이다.
[4] 이 시점 디스크의 inode: 802941      ← 디스크는 이미 바뀜
[5] 이 시점 디스크의 내용: ... 새판 ★    ← 내용도 새판
```

스크립트가 자기를 읽어보니 이미 새판인데, 정작 실행되는 건 옛날판이다. 둘 다 참이다.

배포 스크립트에 대입하면:

```
 26줄  set -euo pipefail
 42줄  export PATH="...shims:$PATH"       ← 새로 넣은 줄. 옛 파일엔 없다
126줄  git reset --hard origin/<branch>   ← 여기서 새 파일이 디스크에 놓인다
163줄  node 검사                           ← PATH 가 비어 실패
```

**새 코드를 가져오는 줄이, 새 코드가 필요한 줄보다 뒤에 있다.**
1차는 반드시 실패하고, 디스크에는 새 파일이 놓였으니 **2차부터 정상**이다.
부작용은 없다 — `exit 1` 이 163줄이라 빌드·기동에 닿지도 않는다.
서버에서 미리 `git pull` 해두면 1차부터 통과한다.

---

## 9. 지금 구조의 빚 — 이중 빌드

```
지금                                바꾼 뒤
────────────────────────────        ────────────────────────────
SourceBuild  yarn build             SourceBuild  yarn build
             build.zip (버려짐)                  build.zip  ← 실제로 쓴다
                                                  dist/
                                                  package.json · yarn.lock
                                                  ecosystem.config.js
SourceDeploy zip 풀기 (덮어씀)       SourceDeploy zip 풀기
             deploy.sh                            yarn install --production
               git reset --hard                   pm2 reload
               yarn install
               yarn build           ← 서버에서 사라지는 것:
               pm2 reload              git · yarn build · node 버전 문제
```

컨테이너와 서버가 **같은 일을 두 번** 한다. 그래서 `build.zip` 의 내용은 실제로 아무 데도 안 쓰인다.
서버에 풀리긴 하지만 곧바로 `git reset --hard` 가 덮어쓴다.
zip 은 "배포할 파일이 있어야 한다"는 SourceDeploy 의 형식 요구만 채우고 버려진다.

의도한 게 아니라, 에이전트가 붙는지 확인하는 데 하루를 다 써서 거기까지 못 간 상태다.

**바꾸면 얻는 것 넷**

- 2 vCPU 박스에서 빌드가 사라진다
- 자기수정 스크립트 문제(8절)가 사라진다
- 서버에 git 이 필요 없어진다
- 무엇이 배포됐는지가 zip 하나로 고정된다 (지금은 빌드한 물건과 배포된 물건이 *각자 따로 빌드된 다른 물건*이다)

**주의 둘**

- **설정 파일은 zip 에 넣지 않는다.** DB 비밀번호와 시크릿이 들어 있다.
- **`node_modules` 도 넣지 않는다.** 컨테이너(node 22)에서 설치한 걸 서버(node 24)에서 쓰게 된다.
  의존성이 전부 순수 JS 면 괜찮지만 네이티브 모듈이 하나라도 들어오면 그날로 깨진다.

---

## 10. 원칙 — 환경 차이는 값에만 둔다

브랜치 · DB 이름 · 포트 · 암호화 키처럼 **값이 다른 것은 어쩔 수 없다.**
하지만 **무엇을 검사하는가 · 어떤 절차를 밟는가**가 환경마다 다르면,
dev 에서 통과한 것이 운영에서 깨지고 **그게 가장 늦게 발견된다.**

실제로 겪었다. 외부 연계 계정을 "운영에서만 필수"로 검증하게 해뒀더니
dev·stage 는 멀쩡히 돌고 운영만 안 떴다. 그건 의도된 가드라 괜찮았지만
(계정 없이 운영을 띄우면 가짜 데이터가 쌓이므로), **같은 패턴을 빌드 검사에까지 만들면 안 된다.**

`ls -l dist/src/main.js` 를 운영에만 두면, "배포 전에 빨리 알자"는 목적 자체가 사라진다.

**예외는 적을수록 좋고, 있으면 문서에 적혀 있어야 한다.**

---

## 11. 왜 폴링이 아니라 이벤트인가

처음 만든 건 서버 크론이었다. 1분마다 `git fetch` 로 새 커밋을 들여다보는 스크립트.
30분이면 만들 수 있었지만 안 쓰기로 했다.

**늦다.** 최악 1분. 그리고 그 1분을 줄일수록 헛일이 는다 — 하루 1,440번 fetch 해서 서너 번만 일이 있다.

**안 돌아도 아무도 모른다.** 크론이 죽어도, `flock` 이 걸린 채 남아도, PATH 가 달라 node 를 못 찾아도 조용하다.
*실패를 알리지 않는 자동화는 자동화가 아니라 시한폭탄이다.*

**기록이 없다.** 어느 커밋이 언제 나갔는지 서버 로그를 뒤져야 한다.

셋 다 "폴링이라서" 생기는 문제다. 이벤트로 가면 한꺼번에 사라진다.

---

## 12. 진단에 실제로 쓴 방법은 셋뿐이었다

**① 길이를 세라.** 내용이 안 보이는 칸도 길이는 보인다.
빌드 명령어 20바이트로 "안 지워졌다"를, 인증키 56바이트로 "플레이스홀더다"를
파일을 열지 않고 확정했다. 추측으로 두 시간 걸릴 걸 계산으로 1분에 끝냈다.

**② 안 변한 숫자를 찾아라.** `pm2` uptime 이 18시간에 멈춰 있다는 것 하나로
"파이프라인은 초록이지만 서버에 닿은 적이 없다"가 확정됐다.
*성공 표시보다 상태값을 믿어라.* 초록불은 "내 구간을 끝냈다"지 "효과가 났다"가 아니다.

**③ 되는 쪽과 안 되는 쪽을 나란히 놓아라.**
한 환경만 실패 → 설정 차이. 손으로는 되는데 CI 만 실패 → 셸 차이.
*차이가 나는 곳에만 원인이 있다.*

그리고 판단 실수 하나 — **문서의 호환성 표를 보고 시도도 안 해보고 포기한 것.**

---

## 부록 — 점검 명령어

```bash
# 배포가 실제로 닿았나 — uptime 이 초 단위로 떨어졌는지
pm2 status

# 어느 커밋이 올라가 있나
git log --oneline -1

# 환경이 맞나
pm2 logs <app> --lines 40 --nostream | grep -E "서버환경=|DB="

# 부팅 실패 원인 — pm2 를 거치면 재시작 로그에 묻힌다. 직접 띄우는 게 빠르다
NODE_ENV=<env> node dist/src/main.js

# 에이전트
sudo systemctl status sdagent
sudo journalctl -u sdagent -n 50 --no-pager

# 인증키가 실물인가 (56 이면 플레이스홀더)
wc -c /opt/NCP_AUTH_KEY

# 비대화형 셸 기준으로 도구가 잡히나 ← 대화형으로 확인하면 안 된다
bash -c 'export PATH="$HOME/.local/share/mise/shims:$PATH"; node -v; yarn -v; pm2 -v'

# errored 프로세스 정리 — save 를 빠뜨리면 재부팅 때 되살아난다
pm2 delete <app> && pm2 save
```
