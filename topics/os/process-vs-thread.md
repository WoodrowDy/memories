---
title: 프로세스 vs 스레드
aliases: [프로세스 스레드, process vs thread, 멀티스레드, multithread]
created: 2026-06-01
updated: 2026-06-01
tags: [os/process, os/thread]
status: growing
---

# 프로세스 vs 스레드

## 핵심

- **프로세스**: 실행 중인 프로그램. 메모리를 **독립**으로 쓴다 → 자원 할당의 단위.
- **스레드**: 프로세스 내부의 실행 흐름. `Code/Data/Heap`을 **공유**하고 `Stack`만 따로 → 실행 흐름의 단위.
- 그래서 스레드는 Context Switching이 가벼워 더 빠르지만, 메모리 공유 탓에 동시성 문제가 생긴다.

## 내용

### 한 줄 정의

- **프로세스(Process)**: 실행 중인 프로그램 자체
- **스레드(Thread)**: 프로세스 내부에서 실제 작업을 수행하는 실행 흐름

### 비유

- **프로세스 = 집 🏠** — 독립된 집. 옆집이 불나도 안전(메모리 직접 접근 불가), 안정성 높음. OS로부터 메모리·CPU·파일·네트워크를 할당받는 "자원 할당의 단위".
- **스레드 = 집 안 사람 👨‍👩‍👧‍👦** — 거실·주방(공용 자원)은 같이, 각자 방(Stack)은 따로. "실행 흐름의 단위".

### 프로세스 메모리 구조

```
[ Code ]   실행 코드 (읽기 전용)
[ Data ]   전역/static 변수 (프로그램 종료까지 유지)
[ Heap ]   동적 객체 (런타임 할당, GC 대상)
[ Stack ]  함수 호출/지역 변수 (호출 시 frame 생성, 종료 시 제거, 스레드마다 독립)
```

```java
static int count = 0;        // Data
User user = new User();      // Heap
int age = 10;                // Stack
```

### 스레드 메모리 구조

```
공유:  Code / Data / Heap
개별:  Thread A Stack | Thread B Stack | Thread C Stack
```

공용 자원은 함께, 실행 흐름(Stack)은 독립.

### 왜 스레드가 더 빠른가 — Context Switching 비용

CPU가 실행 대상을 바꾸는 과정이 Context Switching.

- **프로세스 전환**: 메모리 주소 공간 변경, CPU 캐시 초기화, PCB 교체, 레지스터 교체 → "집 자체를 이동". 무겁다.
- **스레드 전환**: 같은 주소 공간 유지, 캐시 활용 가능 → "방에서 거실로 이동". 가볍다.

멀티 스레드 장점: 빠른 처리, 메모리 절약, 자원 효율, 응답성. (웹 서버, 게임 엔진, 채팅, 영상 처리)

### 동시성 문제

메모리를 공유하므로 여러 스레드가 같은 데이터에 동시 접근 가능.

**Race Condition** — 동시에 같은 데이터 수정.

```java
count++;
```

```
A: count 읽음 → 1
B: count 읽음 → 1
A: 2 저장
B: 2 저장        // 기대값 3, 실제 2
```

**Deadlock** — 서로의 자원을 기다리며 무한 대기.

```
A: Lock1 점유 → Lock2 대기
B: Lock2 점유 → Lock1 대기   // 둘 다 영원히 멈춤
```

해결: ① Lock 순서 통일 ② synchronized 범위 최소화 ③ Timeout ④ Concurrent 자료구조(`ConcurrentHashMap`, `CopyOnWriteArrayList`)

### 멀티 프로세스 vs 멀티 스레드

| 항목 | 멀티 프로세스 | 멀티 스레드 |
| --- | --- | --- |
| 메모리 | 독립 | 공유 |
| 안정성 | 높음 | 낮음 |
| 속도 | 느림 | 빠름 |
| Context Switching | 무거움 | 가벼움 |
| 데이터 공유 | IPC 필요 | 쉬움 |
| 구현 난이도 | 쉬움 | 어려움 |

- **멀티 프로세스(안정성)**: 크롬 탭, Docker Container, 독립 서비스. 하나 죽어도 영향 적음(단 무겁다).
- **멀티 스레드(성능)**: 웹 서버 요청, 게임 서버, 실시간 채팅, 비동기. 빠르지만 동기화 문제.
- 크롬이 탭별 멀티 프로세스를 쓰는 이유 = 한 탭이 죽어도 전체가 안 죽고, 보안 격리되니까.

### 면접 답변 정리

> 프로세스는 OS로부터 자원을 할당받는 독립 실행 단위이고, 스레드는 프로세스 내부의 실행 흐름입니다. 프로세스는 메모리를 독립적으로 쓰지만, 스레드는 Code·Data·Heap을 공유하고 Stack만 독립적으로 씁니다. 따라서 스레드는 Context Switching 비용이 적어 더 가볍고 빠릅니다.

## 관련 문서

- [컨텍스트 스위칭](context-switching.md) (작성 예정)
- [동시성과 동기화](concurrency.md) (작성 예정)
- [JVM 메모리 구조](../lang/jvm-memory.md) (작성 예정)
