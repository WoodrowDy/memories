---
title: 블룸 필터
created: 2026-07-29
updated: 2026-07-29
tags: [cs/data-structure]
status: seedling
---

이거 정리해줘 ## 블룸 필터 확률적 자료구조. 원소가 집합에 "없다"는 건 100% 확신할 수 있고, "있다"는 건 틀릴 수 있다. 비트 배열 하나와 해시 함수 k개를 쓴다. 넣을 때 k개 위치를 1로 세우고, 찾을 때 k개가 전부 1이면 "아마 있다". 지울 수 없다. 지우려면 counting bloom filter를 써야 한다.
