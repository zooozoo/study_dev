# Study Dev Index

개발 일반 학습 노트 — **카테고리로 찾는다.** 각 문서는 배경 지식 없이 읽을 수 있게 쓴다(작성 규칙: [AGENTS.md](AGENTS.md)).

> 규모 확장 규칙: 카테고리는 처음엔 한 파일의 **섹션**으로 시작 → 내용이 쌓이면 **독립 파일**로 분리 → 파일이 여럿 되면 **폴더**로 승격. 분리·승격 시 이 인덱스의 링크만 갱신하면 된다.

## 네트워크 · 실시간 통신
- HTTP의 한계와 keep-alive — [2장](realtime-web-networking-cors-infra.md#2-먼저-http부터--왜-실시간-통신이-어려운가) · [부록 A.1~A.2](realtime-web-networking-cors-infra.md#a1-http는-응답이-끝나면-연결을-닫는다)
- WebSocket (Upgrade 핸드셰이크, ws/wss, 하트비트) — [3장](realtime-web-networking-cors-infra.md#3-websocket--한-번-연결해두고-계속-쓰는-방식)
- STOMP (WebSocket 위의 메시징 규약) — [4장](realtime-web-networking-cors-infra.md#4-stomp--websocket-위에-얹는-대화-규칙)
- SockJS 폴백 사다리 (XHR streaming/polling, 폴백의 대가) — [6장](realtime-web-networking-cors-infra.md#6-sockjs--websocket이-막히면-http로-흉내내기)

## 웹 보안
- CORS (동작 원리, 헤더 사전, WebSocket이 안 걸리는 이유) — [7장](realtime-web-networking-cors-infra.md#7-cors--브라우저가-남의-집-응답을-못-읽게-막는-규칙)

## 인프라
- 프록시 · TLS 인터셉션 — 사내망이 WebSocket을 막는 이유 — [5장](realtime-web-networking-cors-infra.md#5-왜-사내망에서-websocket이-막히나)
- 로드밸런서와 sticky session — [9장](realtime-web-networking-cors-infra.md#9-로드밸런서와-세션-고정sticky-session)
- WAF — [10장](realtime-web-networking-cors-infra.md#10-waf--웹-방화벽)

## 전송 계층 기초
- TCP 연결 종료(4-way, TIME_WAIT), 커널/앱 경계, 프록시 절단, 실패 판정 — [부록 A](realtime-web-networking-cors-infra.md#a1-http는-응답이-끝나면-연결을-닫는다)

---
| 문서 | 다루는 카테고리 | 작성/갱신 |
|---|---|---|
| [realtime-web-networking-cors-infra.md](realtime-web-networking-cors-infra.md) | 실시간 통신 · 웹 보안(CORS) · 인프라 · TCP 기초 | 2026-08-18 |
