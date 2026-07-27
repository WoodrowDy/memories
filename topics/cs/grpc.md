---
title: gRPC
aliases: [gRPC, grpc, protobuf, protocol buffers, proto, 프로토버프, RPC, 원격 프로시저 호출]
created: 2026-07-27
updated: 2026-07-27
tags: [cs/grpc, cs/network, cs/architecture]
status: seedling
---

# gRPC

> 채용공고의 "gRPC / Protocol Buffers 기반의 서비스 간 통신 설계 경험"이라는 한 줄에서 시작했다.
> 처음에는 REST 대신 쓰는 API 정도인 줄 알았는데, 철학 자체가 다른 물건이었다.

---

## 핵심

- gRPC는 **다른 서버의 함수를 호출하는 기술**이다. URL이 아니라 메서드가 사고의 기준이다.
- 그 함수의 이름과 인자를 양쪽이 미리 합의한 문서가 `.proto`, 즉 **계약(Contract)**이다.
- 계약은 서비스마다 복사하는 게 아니라 **한 벌만 두고 공유**한다. 중요한 건 파일 위치가 아니라 누가 관리하느냐다.
- REST는 코드가 먼저고 문서가 따라온다. gRPC는 **계약이 먼저고 코드가 따라온다.**
- 그래서 "gRPC 설계 경험"은 `.proto`를 써본 경험이 아니라, 계약을 **버전과 호환성까지 포함해 운영해본** 경험을 묻는 것이다.

---

## 1. REST와는 사고의 기준이 다르다

REST는 URL을 호출한다.

```
GET /users/1
```

기준은 `Resource(URL)`다. "어떤 URL을 호출하지?"로 생각이 흐른다.

gRPC는 메서드를 호출한다.

```ts
userClient.GetUser({ id: 1 });
```

기준은 `Method(Function)`다. "어떤 **서비스**를 호출하지?"로 생각이 흐른다.

이 차이가 사소해 보이지만 설계 단계에서 갈린다. REST는 자원을 먼저 나누고 동사를 HTTP 메서드에 끼워 맞추는 반면, gRPC는 **하고 싶은 동작을 그대로 함수로 적는다.** 자원으로 표현하기 어색한 동작(`RefundPayment`, `RecalculateRanking` 같은 것)에서 특히 차이가 난다.

---

## 2. 다른 서버의 함수를 어떻게 찾는가

가장 먼저 막혔던 지점이다. 다른 서버인데 함수를 어떻게 아는가.

처음에는 함수 이름으로 서버를 찾아가는 줄 알았다. 아니다. 순서가 반대다.

```
IP + Port
    │
    ▼
Service        UserService
    │
    ▼
Method         GetUser()
```

먼저 `IP + Port`로 **어느 서버인지** 찾는다. 그 서버 안에 등록된 서비스 중 `UserService`를 고르고, 그 안의 `GetUser()`를 실행한다. 서버를 찾는 방식 자체는 REST와 다르지 않다 — 달라지는 건 그 다음부터다.

---

## 3. `.proto` — 계약

그러면 어떤 Method가 있는지는 어떻게 아는가.

REST라면 Swagger 문서를 본다. gRPC의 답은 `.proto`다.

```proto
syntax = "proto3";
package user;

service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  int32 id = 1;
}

message GetUserResponse {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

처음에는 DTO 정도라고 생각했다. 그것보다 넓다. `.proto` 하나가 세 가지를 동시에 갖고 있다.

- **Interface** — `service` 블록이 호출 가능한 메서드 목록이다
- **DTO** — `message` 블록이 주고받는 데이터의 모양이다
- **API 명세** — 이 파일 자체가 문서다. 따로 생성할 것이 없다

NestJS로 치면 DTO + Interface + Swagger를 한 파일에 합쳐놓은 느낌이다.

그리고 이건 문서로만 남지 않는다. `.proto`를 컴파일하면 서버 인터페이스와 클라이언트 코드가 **생성된다.** 문서와 구현이 어긋날 수 없는 구조다 — 어긋나면 컴파일이 깨진다.

---

## 4. 계약은 누가 관리하는가

가장 오래 헷갈린 부분이다.

```
user-service/
└── user.proto

order-service/
└── user.proto
```

이걸 보고 "결국 proto를 서비스마다 정의하는 거 아닌가?"라고 생각했다. 그렇다면 REST에서 DTO를 서로 베껴 쓰는 것과 뭐가 다른가 싶었다.

중요한 것은 파일의 위치가 아니었다. **누가 관리하느냐**였다.

계약은 하나만 존재한다. 실무에서는 공용 계약 저장소를 따로 둔다.

```
grpc-contracts/
├── user.proto
├── payment.proto
└── order.proto
```

각 서비스는 `npm install`이나 Git Submodule로 필요한 proto만 가져온다. 복사해서 각자 관리하는 게 아니라 **하나의 계약을 공유**하는 것이다.

여기서 또 하나 오해했던 게 있다. "명세서를 읽어주는 서버가 따로 도는 건가?" 아니다. `grpc-contracts`는 그냥 **Git 저장소이거나 NPM 패키지**다. NestJS의 `@company/common`과 거의 같은 물건이다.

개발할 때와 실행할 때가 다르다는 걸 구분하면 깔끔해진다.

```
개발 시                     실행 시

Git                         Order Service
 │                              │
 ▼                              ▼
grpc-contracts              User Service
 │                              │
 ▼                              ▼
서비스에 설치                  GetUser()
```

실행 시점에 Git은 전혀 등장하지 않는다. 계약은 이미 코드로 바뀌어 빌드에 들어가 있다.

---

## 5. 통신 4가지 방식

"REST 대신 쓰는 API"라는 첫인상이 진짜로 깨지는 지점이다. gRPC는 요청 하나에 응답 하나만 하는 물건이 아니다.

| 방식 | 모양 | 쓰는 곳 |
|---|---|---|
| Unary | 요청 1 → 응답 1 | 대부분의 조회·명령. REST와 같은 모양 |
| Server streaming | 요청 1 → 응답 N | 대용량 목록, 실시간 알림 |
| Client streaming | 요청 N → 응답 1 | 파일 업로드, 로그 배치 전송 |
| Bidirectional | 요청 N ↔ 응답 N | 채팅, 실시간 동기화 |

```proto
service UserService {
  rpc GetUser      (GetUserRequest)        returns (GetUserResponse);
  rpc ListUsers    (ListUsersRequest)      returns (stream User);
  rpc UploadUsers  (stream User)           returns (UploadResult);
  rpc SyncUsers    (stream User)           returns (stream User);
}
```

`stream` 키워드 하나로 갈린다. 이게 가능한 건 gRPC가 HTTP/2 위에서 돌기 때문이다. HTTP/2는 커넥션 하나에 여러 스트림을 동시에 흘릴 수 있어서, 요청과 응답이 번갈아 오가는 모양이 자연스럽게 표현된다. REST(HTTP/1.1)로는 애초에 표현이 안 되는 형태다.

---

## 6. 버전과 호환성 — "설계 경험"의 실체

채용공고가 묻는 게 결국 여기다. 계약을 바꿔야 할 때 이미 돌고 있는 서비스들을 어떻게 안 깨뜨리느냐.

### 필드 번호가 진짜 식별자다

```proto
message User {
  int32  id    = 1;
  string name  = 2;
  string email = 3;
}
```

`= 1`, `= 2`는 순서가 아니다. **와이어에 실제로 실려 나가는 식별자**다. 필드 이름은 바이너리에 들어가지 않는다. 그래서

- 필드 **이름은 바꿔도 안전하다** (번호가 같으면 같은 필드다)
- 필드 **번호를 바꾸면 즉시 깨진다**
- 필드를 지웠다가 **그 번호를 다른 필드에 재사용하면 가장 위험하다.** 옛 클라이언트가 보낸 데이터가 새 필드로 조용히 잘못 해석된다. 에러도 안 난다

### 그래서 `reserved`로 막는다

```proto
message User {
  reserved 3, 4;
  reserved "phone_number";

  int32  id    = 1;
  string name  = 2;
  string email = 5;
}
```

지운 번호와 이름을 못 쓰게 못박아둔다. "호환성을 어떻게 유지했는지"에 대한 가장 구체적인 답이 이거다.

### 안전한 변경 / 위험한 변경

| 안전 | 위험 |
|---|---|
| 필드 추가 (새 번호로) | 필드 번호 변경·재사용 |
| 필드 이름 변경 | 필드 타입 변경 |
| `service`에 rpc 추가 | rpc 삭제·시그니처 변경 |
| 필드 삭제 + `reserved` 처리 | 필드 삭제만 하고 방치 |

proto3에는 `required`가 없어서 모든 필드가 사실상 선택이다. 그래서 **필드 추가는 거의 항상 안전하다** — 모르는 필드를 받은 옛 코드는 그냥 무시한다. 이게 gRPC 계약이 점진적으로 자랄 수 있는 이유다.

> 자주 쓰는 필드는 1~15번을 준다. 태그가 1바이트로 인코딩돼서 16번부터보다 작다.

### 배포 순서

계약이 바뀌면 **서버를 먼저 올리고 클라이언트를 나중에 올린다.** 반대로 하면 클라이언트가 아직 없는 메서드를 부른다. MSA에서 코드보다 계약을 먼저 고치는 이유가 이 순서 때문이다.

---

## 7. 에러는 어떻게 오는가

gRPC는 HTTP 상태코드를 쓰지 않는다. 자체 status code가 따로 있다.

| 코드 | 언제 |
|---|---|
| `OK` | 성공 |
| `INVALID_ARGUMENT` | 인자가 틀렸다 (REST의 400) |
| `NOT_FOUND` | 대상이 없다 (404) |
| `PERMISSION_DENIED` | 권한 없음 (403) |
| `UNAUTHENTICATED` | 인증 안 됨 (401) |
| `UNIMPLEMENTED` | **계약엔 있는데 서버에 구현이 없다** |
| `UNAVAILABLE` | 서버에 못 닿는다 |
| `DEADLINE_EXCEEDED` | 시한 초과 |

`UNIMPLEMENTED`가 gRPC다운 에러다. REST에는 대응하는 개념이 없다 — 계약과 구현이 따로 존재하기 때문에 생기는 상태다. 계약을 먼저 고치고 서버 구현을 아직 안 했을 때 정확히 이게 뜬다.

`DEADLINE_EXCEEDED`도 눈여겨볼 만하다. gRPC는 **deadline이 프로토콜에 내장**돼 있고, 호출이 서비스를 타고 넘어갈 때 남은 시간이 함께 전파된다. A→B→C로 흐르는 호출에서 A가 준 5초가 C까지 따라간다. REST에서 타임아웃을 각 구간마다 손으로 맞추던 것과 다른 지점이다.

---

## 8. 헷갈리는 지점

### FeignClient ≠ gRPC

Spring의 FeignClient도 원격 호출을 함수처럼 쓰게 해준다. 겉모습이 비슷해서 같은 것으로 보였다.

```
FeignClient                  gRPC

userClient.getUser()         userClient.GetUser()
      │                            │
      ▼                            ▼
    HTTP/1.1                    HTTP/2
      │                            │
      ▼                            ▼
     JSON                    Protobuf(Binary)
```

Feign은 **REST를 함수처럼 보이게 감싼 것**이다. 껍데기를 벗기면 여전히 URL과 JSON이 있다. gRPC는 애초에 함수 호출을 위해 설계됐고, 바닥에 URL이 없다.

### 브라우저에서는 gRPC를 그대로 쓸 수 없다

이건 몰랐다가 놀란 부분이다. 브라우저 JavaScript는 HTTP/2 프레임을 직접 다룰 수 없다 — `fetch`나 `XHR`이 그 아래를 감춰버린다. 그래서 브라우저에서 gRPC 서버를 직접 부르는 건 불가능하고, **grpc-web + 프록시(Envoy 등)**를 거쳐야 한다. 그마저도 client streaming과 bidirectional은 지원되지 않는다.

그래서 현실의 구조는 대개 이렇게 갈린다.

```
Browser
   │  REST / GraphQL
   ▼
API Gateway (BFF)
   │  gRPC
   ├──▶ User Service
   ├──▶ Order Service
   └──▶ Payment Service
```

gRPC는 **서비스 사이**에서 쓰고, 클라이언트와 맞닿는 경계는 REST로 남긴다. "gRPC가 REST를 대체한다"가 아니라 **자리가 다르다**가 맞는 이해였다.

### REST는 코드가 먼저, gRPC는 계약이 먼저

```
REST                        gRPC

Controller 작성             .proto 작성
      │                          │
      ▼                          ▼
Swagger 자동 생성            Server 구현
                                 │
                                 ▼
                            Client 코드 생성
```

REST에서 문서는 코드의 **결과물**이다. gRPC에서 계약은 코드의 **원인**이다. 그래서 MSA에서는 코드보다 계약을 먼저 수정한다.

---

## 면접 단골 질문

### Q. gRPC가 REST보다 빠른 이유가 뭔가?

두 가지다. 직렬화가 JSON 텍스트가 아니라 Protobuf 바이너리라 크기가 작고 파싱이 싸다. 그리고 HTTP/2를 써서 커넥션 하나를 여러 요청이 나눠 쓰고 헤더도 압축된다.

### Q. `.proto`는 DTO인가?

DTO를 포함하지만 그게 전부가 아니다. `message`가 DTO라면 `service`는 인터페이스이고, 파일 자체가 API 명세다. 그리고 이 파일에서 실제 코드가 생성된다.

### Q. 필드를 지우면 그 번호를 다시 써도 되나?

안 된다. 필드 번호가 와이어의 실제 식별자라, 재사용하면 옛 클라이언트가 보낸 값이 새 필드로 조용히 잘못 해석된다. `reserved`로 막아둬야 한다.

### Q. 계약을 바꿀 때 서버와 클라이언트 중 뭘 먼저 배포하나?

서버 먼저다. 클라이언트를 먼저 올리면 아직 없는 메서드를 부르게 되어 `UNIMPLEMENTED`가 난다.

### Q. 브라우저에서 gRPC 서버를 직접 부를 수 있나?

없다. 브라우저가 HTTP/2 프레임을 직접 다루지 못해서 grpc-web과 프록시가 필요하고, 스트리밍도 일부만 된다. 그래서 보통 클라이언트 경계는 REST로 두고 서비스 간 통신에만 gRPC를 쓴다.

### Q. gRPC 설계 경험이라는 게 정확히 뭘 말하나?

`.proto`를 써본 경험이 아니라 계약을 운영해본 경험이다. 계약을 어떻게 나눴는지, 여러 서비스가 어떻게 공유했는지, 버전을 어떻게 올렸는지, 하위 호환을 어떻게 지켰는지까지 포함한다.

---

## 한 줄 정리

> gRPC는 여러 서버가 `.proto`라는 하나의 계약을 공유하고, 그 계약을 근거로 서로의 함수를 안전하게 호출하는 방식이다.

---

## 더 볼 것

아직 직접 해보지 않아 확인하지 못한 것들.

- `.proto` 하나 써서 NestJS 서버·클라이언트로 실제 호출해보기 — 여기까지 해야 `growing`이다
- `grpc-contracts`를 모노레포에 둘 때와 별도 저장소로 뺄 때의 차이
- 계약 변경을 CI에서 자동 검사하는 방법 (`buf breaking` 같은 도구)
- 인터셉터로 인증·로깅·트레이싱을 어떻게 붙이는가
- HTTP/2 자체 — 멀티플렉싱과 HPACK을 따로 정리하면 5장과 7장이 훨씬 선명해진다

---

## 관련 문서

- [동기 vs 비동기](sync-vs-async.md) — 스트리밍과 deadline을 이해하는 바탕
- [동시성 vs 병렬성](concurrency-vs-parallelism.md)
