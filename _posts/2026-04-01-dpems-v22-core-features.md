---
layout: single
title: "D-PEMS 개발기 #3 — v2.2 핵심 기능 완성 + SPEC 동기화"
date: 2026-04-01
categories: [프로젝트, D-PEMS]
tags: [D-PEMS, Python, PySide6, 태스크관리]
toc: true
---

## Phase 1 완료: v2.2

기본 뼈대에서 실제로 쓸 수 있는 수준으로 올라왔다. v2.2에서 추가된 핵심 기능들을 정리한다.

## 주요 추가 기능

### 일과(Work Session) 관리

하루를 세션 단위로 관리하는 기능이다. 일과를 시작하면 엔진이 가동되고, 종료하면 멈춘다. `WORK_AUTO_START`, `WORK_AUTO_START_TIME` 설정으로 특정 시각에 자동 시작도 가능하다.

### chain_to: 태스크 연결

태스크 완료 시 다음 태스크를 자동으로 부스트하는 기능이다. 예를 들어 "운동 완료" → "샤워" 태스크가 즉시 올라오도록 설정할 수 있다.

```python
if schema.chain_to:
    for t in tasks:
        if t['schema'].name == schema.chain_to:
            t['state'].last_action_at = now_ts - (schema.target_time * 60)
```

### triggered_by: 조건부 활성화

특정 태스크가 완료됐을 때만 활성화되는 태스크를 설정할 수 있다. 선행 조건이 있는 작업에 유용하다.

### focus_group: 집중도 레벨

태스크마다 집중도 그룹을 설정한다.

| 그룹 | 동작 |
|------|------|
| 1: 유예 | 완료 후 가중치 천천히 증가 |
| 2: 유지 | 기본 동작 |
| 3: 강화 | 완료 후 빠르게 다시 올라옴 |

### IGNORE (PASS) 기능

지금 당장 할 수 없는 태스크를 전광판에서 임시로 제외한다. 가중치는 그대로 유지되므로 나중에 다시 올라온다. IGNORE는 로그에도 기록된다.

### 로그 시스템

모든 행위는 `dpems_log.json`에 append only로 기록된다. 덮어쓰기는 절대 없다.

```json
{
  "type": "ACTION",
  "task": "운동",
  "timestamp": "2026-04-01 09:30:00",
  "weight": 145.50,
  "memo": ""
}
```

로그 타입: `ACTION`, `MEMO`, `TOGGLE_ON`, `TOGGLE_OFF`, `IGNORE`

## SPEC.md 도입

기능이 많아지면서 코드만으로 전체 구조를 파악하기 어려워졌다. `SPEC.md`를 별도로 관리하기 시작했다. CLAUDE.md와 함께 AI 보조 개발 시 컨텍스트를 명확히 하기 위한 목적도 있다.

코드와 SPEC이 따로 놀지 않도록, 기능을 추가할 때마다 SPEC도 같이 업데이트하는 걸 원칙으로 잡았다.

## 아키텍처 다시 보기

```
DataManager → main.py → Engine
                      → Dashboard
Engine.tick_completed → Dashboard.on_data_updated
Dashboard 버튼 → main.py handlers → Engine.force_tick()
```

handlers는 main.py에서 정의하고 dashboard에 주입한다. dashboard는 버튼 이벤트를 받으면 handler를 호출할 뿐, 저장/계산 로직은 전혀 없다.

---

다음 글에서는 PC-모바일 동기화를 위해 Google Sheets를 데이터 허브로 활용한 설계를 다룬다.
