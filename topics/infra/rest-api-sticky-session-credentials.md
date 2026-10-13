---
title: REST API에 스티키 세션을 켰는데 가끔 404가 난다 — 로드밸런서 옵션만으로는 안 되는 이유
aliases: [스티키 세션 CORS, sticky session credentials]
created: 2026-09-30
updated: 2026-09-30
tags: [infra/load-balancing, infra/autoscaling, cs/cors]
status: growing
---

# REST API에 스티키 세션을 켰는데 가끔 404가 난다 — 로드밸런서 옵션만으로는 안 되는 이유

서버를 여러 대로 늘리면서 로드밸런서 스티키 세션을 켰는데, 요청이 여전히 여러 서버로 흩어지는 경우가 있다. 에러가 매번 나는 게 아니라 가끔 나서 더 찾기 어렵다.

원인은 대개 로드밸런서가 아니라 **브라우저가 스티키 쿠키를 보내지 않는 것**이다. 화면과 API의 origin이 다르면 스티키 세션은 로드밸런서 옵션만으로 완성되지 않는다. 프런트 호출 코드와 서버 CORS 설정까지 같이 맞춰야 한다.

스티키 세션을 다룬 글은 대부분 WebSocket(Socket.IO) 쪽이고, 일반 REST API 기준으로 정리된 글은 잘 보이지 않아서 겪은 내용을 남긴다.

---

## 상황

- NestJS API 서버. 원래 한 대, pm2 fork 1개로 돌았다.
- 오토스케일링으로 서버를 여러 대로 늘리려고 했다.
- 구조: `브라우저 → CDN(CloudFront) → 로드밸런서 → API 서버 N대`
- 화면은 `app.example.com`, API는 `api.example.com`이다. 같은 사이트지만 서브도메인이 다르다.

API에 서버 메모리에 의존하는 흐름이 하나 있었다.

1. 사용자가 엑셀을 올리면 서버가 검사하고, 문제 있는 행에 사유를 붙인 엑셀을 **서버 메모리에 10분 보관**한다. 응답으로는 다운로드 토큰만 준다.
2. 사용자가 토큰으로 파일을 내려받는다.

파일에 개인정보가 들어 있어서 DB나 Redis에 저장하지 않기로 한 설계다. 서버가 한 대일 때는 아무 문제가 없다.

서버가 두 대가 되면, 1번을 처리한 서버와 2번을 받은 서버가 다를 때 **404**가 난다.

## 스티키 세션을 켜면 끝날 줄 알았다

로드밸런서가 같은 사용자의 요청을 같은 서버로 보내 주면 해결되는 문제다. 스티키 세션은 보통 두 방식으로 동작한다.

| 방식 | 동작 | 이 구조에서 |
|---|---|---|
| 클라이언트 IP | 같은 IP면 같은 서버로 보낸다 | **안 된다.** LB 앞에 CDN이 있어서 LB가 보는 IP는 사용자가 아니라 CDN 엣지 IP다. 요청마다 바뀔 수 있다 |
| 쿠키 | LB가 응답에 "너는 A 서버" 쿠키를 심고, 다음 요청에서 그 쿠키를 보고 A로 보낸다 | 된다. **단, 브라우저가 쿠키를 돌려보내야 한다** |

쿠키 방식을 켰다. 그런데 **브라우저가 그 쿠키를 돌려보내지 않았다.**

## 원인 — 다른 origin 요청에는 쿠키가 기본으로 안 실린다

`app.example.com`과 `api.example.com`은 같은 사이트(`example.com`)지만 **origin은 다르다.** origin은 스킴 + 호스트 + 포트 전체를 본다.

브라우저의 `fetch`는 `credentials` 기본값이 `"same-origin"`이다. 다른 origin으로 요청할 때는:

- 요청에 쿠키를 싣지 않는다.
- 응답의 `Set-Cookie`도 저장하지 않는다.

그래서 로드밸런서가 스티키 쿠키를 아무리 줘도 브라우저는 받지도 않고 보내지도 않는다. 로드밸런서 입장에서는 매 요청이 처음 보는 사용자라, 또 아무 서버로나 보낸다.

axios도 마찬가지다. `withCredentials` 기본값이 `false`다.

## 해결 — 세 곳을 같이 바꾼다

### 1. 프런트: 쿠키 전송을 켠다

```js
// fetch
fetch(url, { credentials: "include" });

// axios
axios.defaults.withCredentials = true;
// 또는 인스턴스 생성 시
const api = axios.create({ baseURL, withCredentials: true });
```

상태가 이어져야 하는 요청(여기서는 검사와 다운로드)에는 반드시 넣는다. 전체에 걸어 두는 게 빠뜨릴 일이 없어서 낫다.

### 2. 서버: CORS에서 origin을 명시하고 credentials를 허용한다

`credentials: "include"` 요청에 브라우저가 응답을 넘겨주려면 두 가지가 필요하다.

- `Access-Control-Allow-Credentials: true`
- `Access-Control-Allow-Origin`이 `*`가 아니라 **요청한 origin 그대로**

```ts
// NestJS (main.ts)
const corsOriginList = (process.env.CORS_ORIGIN_LIST ?? "")
  .split(",")
  .map((origin) => origin.trim())
  .filter((origin) => origin.length > 0);
const hasCorsOriginList = corsOriginList.length > 0;

app.enableCors({
  origin: hasCorsOriginList ? corsOriginList : "*",
  credentials: hasCorsOriginList,
});
```

- 허용 origin은 환경변수로 뺐다. 환경마다 화면 주소가 다를 수 있고, 로컬 개발 주소를 dev에만 넣을 수도 있다.
- 배포 환경에서는 이 값이 비어 있으면 부팅 단계(설정 검증)에서 막았다. 조용히 `*`로 뜨면 프런트가 credentials를 켜는 순간 전부 막힌다.

### 3. 로드밸런서: Target Group에 쿠키 기반 스티키를 켠다

이건 원래 하려던 것이다. 위 두 가지가 없으면 이것만 켜서는 효과가 없다.

## 반영 순서가 중요하다

**서버 → 프런트** 순서로 올려야 한다.

| 서버 CORS | 프런트 credentials | 결과 |
|---|---|---|
| `*` | 꺼짐 | 지금 상태. 동작한다. 스티키는 안 된다 |
| origin 명시 + credentials | 꺼짐 | 동작한다. 서버를 먼저 올려도 안전하다 |
| `*` | **켜짐** | **브라우저가 모든 응답을 막는다** |
| origin 명시 + credentials | 켜짐 | 목표 상태 |

프런트가 먼저 나가면 세 번째 줄이 된다. 전체 장애다.

## 화면과 API가 완전히 다른 사이트라면

예를 들어 화면은 `app.foo.com`, API는 `api.bar.com`인 경우다. 이러면 스티키 쿠키는 브라우저 입장에서 **제3자 쿠키**가 된다.

- 쿠키에 `SameSite=None; Secure`가 붙어 있어야 보낸다. LB가 만드는 쿠키의 속성은 보통 우리가 정할 수 없다.
- Safari처럼 제3자 쿠키를 기본으로 막는 브라우저에서는 아예 안 된다.

AWS ALB는 이 문제 때문에 CORS용 쿠키(`AWSALBCORS`, `SameSite=None` 포함)를 따로 하나 더 발급한다. 다른 LB는 문서에 쿠키 속성이 안 나와 있는 경우가 많아서 직접 확인해야 한다.

이 경우에는 스티키 대신 아래 방법을 먼저 보는 게 낫다.

## 스티키가 안 맞을 때의 다른 방법

- **서버를 상태 없이 만든다:** 검사 응답에 파일을 바로 담거나, 서버는 사유 목록만 주고 파일은 브라우저에서 만든다. 스티키가 아예 필요 없다.
- **서버 간 전달:** 토큰에 파일을 가진 서버의 사설 주소를 넣어 서명한다. 다른 서버가 요청을 받으면 내부망으로 그 서버에서 받아다 준다. 프런트와 LB를 건드리지 않는다. 대신 내부 API 보호와 토큰 서명을 챙겨야 한다.
- **외부 저장소(Redis 등):** 가장 흔한 답이다. 저장하면 안 되는 데이터라서 이번에는 제외했다.

## 확인 방법

서버가 한 대면 절대 재현되지 않는다. 반드시 **두 대 이상**에서 확인한다.

1. 브라우저 개발자 도구 → Application → Cookies에 LB 쿠키가 생기는지 본다.
2. Network 탭에서 두 번째 요청의 Request Headers에 그 쿠키가 실리는지 본다.
3. 검사 → 다운로드를 10번 이상 반복해서 404가 한 번도 없는지 본다.

## 정리

- 스티키 세션은 로드밸런서 옵션이지만, **쿠키를 주고받는 건 브라우저**다.
- 화면과 API의 origin이 다르면 세 곳을 같이 바꿔야 한다.
  - 프런트: `credentials: "include"`
  - 서버: CORS origin 명시 + `credentials: true`
  - LB: 스티키 켜기
- 서버를 먼저, 프런트를 나중에 올린다.
- 앞에 CDN이 있으면 IP 기반 스티키는 못 쓴다.
- 증상이 "가끔 404"라서 서버 한 대로는 절대 재현되지 않는다.

## 참고

- [MDN — Using the Fetch API (credentials)](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- [MDN — Access-Control-Allow-Credentials](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Credentials)
- [javascript.info — CORS (한국어)](https://ko.javascript.info/fetch-crossorigin)
- [AWS — Application Load Balancer 고정 세션 (한국어)](https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/application/sticky-sessions.html)
- [Socket.IO — Client options: withCredentials](https://socket.io/docs/v4/client-options/)
- [socket.io-client Issue #1159 — Cookie-based sticky sessions](https://github.com/socketio/socket.io-client/issues/1159)
- [AWS re:Post — AWSALB and AWSALBCORS are 3rd party cookies in web](https://repost.aws/questions/QUIxxI6nhjR9-Y7kTinP6zTQ/awsalb-and-awsalbcors-are-3rd-party-cookies-in-web)
