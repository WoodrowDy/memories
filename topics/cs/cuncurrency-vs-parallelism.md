---
title: 동시성 vs 병렬성
aliases: [동시성 병렬성, concurrency vs parallelism, 동시성, 병렬성, concurrency, parallelism]
created: 2026-06-04
updated: 2026-06-04
tags: [os/concurrency, os/parallelism]
status: growing
---

# 동시성(Concurrency) vs 병렬성(Parallelism)

![concurrency vs parallelism](media/concurrency-vs-parallelism.png)

## 한 줄 요약

```
동시성(Concurrency) = 구조(Structure)
병렬성(Parallelism) = 실행(Execution)
```

개발자 면접에서는 이 한 문장만 기억해도 절반은 먹고 들어간다.

---

## 요리사 비유

### 동시성 (Concurrency)

요리사 1명

```
양파 썰기
↓
물 끓이기
↓
양파 썰기
↓
고기 굽기
↓
물 확인
```

작업을 번갈아 수행한다. 실제로 동시에 하는 것은 아니지만 여러 작업이 진행되는 것처럼 보인다.

### 병렬성 (Parallelism)

요리사 2명

```
요리사 A → 양파 썰기
요리사 B → 물 끓이기
```

실제로 같은 시간에 작업이 수행된다.

---

## CPU 관점

### 동시성

싱글코어에서도 가능

```
CPU 1개
Task A
↓
Task B
↓
Task A
↓
Task C
```

CPU가 매우 빠르게 작업을 교체한다. 이를 `Context Switching`, `Time Slicing` 이라고 한다. 사용자는 동시에 실행되는 것처럼 느낀다.

### 병렬성

멀티코어에서 가능

```
Core1 → Task A
Core2 → Task B
Core3 → Task C
```

실제로 같은 시점에 실행된다.

---

## 핵심 비교표

| 구분 | 동시성 | 병렬성 |
| --- | --- | --- |
| 의미 | 여러 작업을 겹쳐 다루는 구조 | 여러 작업을 실제 동시에 실행 |
| 초점 | 설계(Structure) | 실행(Execution) |
| 개념 | 논리적(Logical) | 물리적(Physical) |
| 싱글코어 | 가능 | 불가능 |
| 멀티코어 | 가능 | 가능 |
| 목적 | 자원 활용 극대화 | 처리속도 향상 |

---

## 개발자 관점

### Node.js

대표적인 동시성 모델

```js
fs.readFile(...)
db.query(...)
api.call(...)
```

Node.js는 기본적으로 싱글 스레드다. 하지만

```
파일 읽기 요청
↓
기다리는 동안 다른 요청 처리
↓
파일 읽기 완료
↓
콜백 실행
```

방식으로 동시성을 구현한다.

### Spring Boot

대표적으로 `Thread Pool` 을 사용한다.

```
요청 1 → Thread 1
요청 2 → Thread 2
요청 3 → Thread 3
```

멀티코어 환경에서는 `동시성 + 병렬성` 을 함께 활용한다.

---

## 면접 단골 질문

### Q. 싱글코어에서 동시성이 왜 필요한가요?

CPU보다 I/O가 훨씬 느리기 때문이다.

```
DB 조회
CPU 사용 : 1ms
DB 응답 대기 : 100ms
```

기다리기만 하면 `CPU 99% 놀고 있음` 상태가 된다. 그래서 DB 응답을 기다리는 동안 다른 요청을 처리한다. 이것이 동시성이다.

### Q. 병렬성이 무조건 빠른가요?

아니다. 오히려 느려질 수도 있다. 병렬 처리에는 비용이 존재한다.

```
스레드 생성
컨텍스트 스위칭
락 경쟁(Lock Contention)
메모리 사용 증가
```

작업이 매우 작다면 `병렬 처리 비용 > 성능 향상` 이 될 수 있다.

```java
// 1 + 1 계산하는데 스레드 100개 생성하면 더 느려진다
```

### Q. 병렬성 없이도 성능 향상이 가능한가요?

가능하다. 대표적으로 `비동기 처리`, `논블로킹 I/O`, `이벤트 루프` 방식이 있다.

```js
Promise.all()
```

---

## 파이썬과 GIL

### Q. 파이썬은 멀티코어인데 왜 쓰레드 병렬처리가 안되나요?

GIL (Global Interpreter Lock)

CPython에는 "한 순간에 하나의 스레드만 파이썬 바이트코드를 실행 가능"이라는 제약이 있다.

```
thread1
↓
thread2
↓
thread3
↓
thread1
```

번갈아 실행. 즉, 멀티코어 환경이어도 CPU 연산은 병렬 처리되지 않는다.

### 그럼 파이썬 멀티쓰레드는 왜 쓰나요?

I/O 작업 때문이다.

```python
API 호출
DB 조회
파일 읽기
웹 크롤링
```

대기시간 동안 `GIL 반환` 하므로 효과적이다.

### CPU 연산은 어떻게 병렬처리 하나요?

멀티프로세스 사용

```python
from multiprocessing import Pool
```

프로세스는 `GIL 공유 안함`

```
Core1 → Process1
Core2 → Process2
Core3 → Process3
```

진짜 병렬 실행 가능

---

## 실무 기준 정리

### CPU 집약 작업

```
이미지 처리 / 영상 인코딩 / 대규모 계산 / AI 모델 학습 / 데이터 분석
```

→ 병렬성 중요 → 멀티코어 활용 → 멀티프로세스 활용

### I/O 집약 작업

```
웹 서버 / API 호출 / DB 조회 / 파일 업로드 / 채팅 서버
```

→ 동시성 중요 → 비동기 처리 활용 → 이벤트 루프 활용

---

## 면접 100점 답안

> 동시성은 여러 작업을 겹쳐 처리할 수 있도록 구성하는 논리적 구조의 개념입니다. 싱글코어에서도 컨텍스트 스위칭을 통해 구현할 수 있으며, Node.js의 이벤트 루프가 대표적인 예입니다.
>
> 병렬성은 여러 작업이 실제로 같은 시점에 실행되는 물리적 개념입니다. 멀티코어 환경이 필요하며, 이미지 렌더링이나 대규모 연산 같은 CPU 집약적 작업에서 주로 활용됩니다.
>
> 즉, 동시성은 구조(Structure), 병렬성은 실행(Execution)이라고 이해하면 됩니다.

---

## 최종 암기 포인트

```
동시성 = 논리적 = 구조 = 싱글코어 가능
병렬성 = 물리적 = 실행 = 멀티코어 필요

Node.js               = 동시성 대표
Spring                = 동시성 + 병렬성
Python Thread         = 동시성
Python Multiprocessing = 병렬성

CPU 작업 = 병렬성
I/O 작업 = 동시성
```
