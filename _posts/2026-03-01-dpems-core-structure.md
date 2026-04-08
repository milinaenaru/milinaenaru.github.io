---
layout: single
title: "D-PEMS 개발기 #1 — 설계 기반 다지기: Config, Schema, State"
date: 2026-03-01
categories: [프로젝트, D-PEMS]
tags: [D-PEMS, Python, PySide6, 태스크관리]
toc: true
---

## 프로젝트 시작

**D-PEMS (Dynamic Priority Execution Management System)** — 동적 우선순위 실행 관리 시스템.

단순한 할 일 목록이 아니다. 등록한 태스크마다 경과 시간, 중요도, 반복 주기를 바탕으로 **가중치를 실시간으로 계산**해 우선순위를 자동으로 매기는 시스템이다. 가장 급한 것이 항상 맨 위에 올라온다.

## 핵심 설계 결정

### 데이터 구조 분리: Schema vs State

태스크를 두 레이어로 나눴다.

- **TaskSchema** — 변하지 않는 정의. 이름, 주기 타입, 목표 시간, 가중치 곡선 등. `frozen=True`로 불변 처리.
- **TaskState** — 런타임 상태. 현재 가중치, 마지막 실행 시각, 완료 횟수, 토글 ON/OFF.

Schema는 사용자가 설정하는 것, State는 시스템이 계산하는 것이다. 이 둘을 분리하면 저장/로드, 초기화, 직렬화가 훨씬 깔끔해진다.

```python
@dataclass(frozen=True)
class TaskSchema(AdvancedRules):
    name: str = ""
    is_toggle: bool = False
    period_type: str = "DAILY"   # ONCE/MINUTE/DAILY/WEEKLY/MONTHLY
    target_time: int = 480       # 100점 기준점 (분)

@dataclass
class TaskState:
    name: str
    current_weight: float = 0.0
    last_action_at: float = field(default_factory=time.time)
    daily_count: int = 0
    is_on: bool = False
```

### 상수 관리: config.py + styles.py

수치 하드코딩은 나중에 찾기도 힘들고 수정도 귀찮다. 처음부터 분리했다.

- `config.py` — 시스템 수치 (점수 기준, 타이머 간격, 테스트 모드 플래그 등)
- `styles.py` — UI 색상/스타일 문자열 (색상 변수 `C_*` → f-string 참조)

## 파일 구조

```
dpems/
├── main.py         # 진입점
├── engine.py       # 가중치 계산 루프
├── dashboard.py    # GUI
├── components.py   # UI 컴포넌트
├── dialogs.py      # 다이얼로그
├── data_manager.py # JSON 저장/로드
├── schemas.py      # TaskSchema, AdvancedRules
├── state.py        # TaskState
├── styles.py       # 스타일 상수
└── config.py       # 수치 상수
```

아키텍처 원칙은 단순하다: **engine은 계산만, dashboard는 표시만, main은 연결만.**

---

다음 글에서는 PySide6로 실제 데스크탑 앱을 구현한 내용을 다룬다.
