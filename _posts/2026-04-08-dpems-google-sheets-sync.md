---
layout: single
title: "D-PEMS 개발기 #4 — 서버 없이 PC-모바일 동기화: Google Sheets 이벤트 허브"
date: 2026-04-08
categories: [프로젝트, D-PEMS]
tags: [D-PEMS, Python, Google Sheets, 동기화, 멀티기기]
toc: true
---

## 문제: 서버 없이 멀티 기기 동기화

PC에서 작업하다가 나가면 모바일에서도 같은 태스크 상태를 봐야 한다. 그런데 별도 서버를 운영하기는 번거롭다. 비용도 들고 유지보수도 필요하다.

해결책: **Google Sheets를 데이터 허브로 사용한다.**

모든 기기가 Google Sheets 하나를 공유한다. 서버 코드 없음, 별도 인프라 없음. Google의 인프라를 무료로 쓴다.

## 설계: 이벤트 기반 동기화

단순히 태스크 상태를 시트에 저장하는 게 아니다. **이벤트를 시트에 기록하고 각 기기가 미처리 이벤트를 pull**하는 방식이다.

### 시트 구조

| 시트 이름 | 역할 |
|-----------|------|
| `events_2026` | 이벤트 로그 + 기기별 ack 열 (연도별 영구 보관) |
| `tasks` | 현재 태스크 전체 스냅샷 |
| `settings` | 일과 설정, DEVICE_ID |

### 이벤트 타입

```
ADD      — 태스크 추가
DELETE   — 태스크 삭제
UPDATE   — 태스크 스키마 수정
ACTION   — 태스크 완료
MEMO     — 메모 기록
IGNORE   — PASS 처리
```

### 기기별 ack 열

`events` 시트의 각 행은 이벤트 하나다. 열은 `seq`, `type`, `target`, `data`, `timestamp` + **기기별 ack 열**로 구성된다.

기기가 이벤트를 처리하면 자기 열에 체크한다. 새 기기가 등록되면 자동으로 새 열이 추가된다. 덕분에 어떤 기기가 어떤 이벤트를 처리했는지 시트에서 바로 확인 가능하다.

## 구현: SheetSync 클래스

```python
class SheetSync:
    def connect(self) -> bool:       # 시트 연결 + 기기 열 등록
    def push_event(type, target, data)   # 이벤트 시트에 기록
    def pull_events(tasks) -> int    # 미처리 이벤트 가져와 적용
    def full_sync(tasks) -> bool     # 첫 연결 시 전체 동기화
    def save_tasks_to_sheet(tasks)   # tasks 시트 갱신
    def save_settings_to_sheet()     # settings 시트 갱신
```

### 오프라인 큐

네트워크가 없으면 `_pending_events` 리스트에 이벤트를 쌓아둔다. 온라인 복귀 시 `_flush_pending()`으로 한꺼번에 전송한다. 오프라인 상태에서도 로컬은 정상 동작한다.

### 새 기기 등록

처음 연결하는 기기는 `_is_new_device` 플래그가 세워지고, `tasks` 시트에서 전체 데이터를 내려받아 초기화된다. 기존 기기는 마지막 ack 이후의 이벤트만 pull한다.

## data_manager.py 연동

`data_manager`에 `_sync` 레이어를 추가했다. `main.py`에서 `SheetSync` 인스턴스를 주입하면, 이후 모든 태스크 조작(추가/삭제/수정/완료/PASS)이 자동으로 이벤트를 push한다.

```python
# data_manager.py
def _push(event_type: str, target: str, data: dict = None):
    if _sync is not None:
        _sync.push_event(event_type, target, data)

def add_task(tasks, task):
    tasks.append(task)
    _push("ADD", task["schema"].name, dataclasses.asdict(task["schema"]))
```

동기화 실패는 예외를 잡아 무시한다. 동기화가 안 돼도 로컬 동작에는 영향 없다.

## 앱 시작 흐름

```
1. config 로드 (DEVICE_ID 복원)
2. SheetSync 초기화 + connect()
3. 로컬 tasks 로드
4. full_sync() — 시트의 미처리 이벤트 반영
5. 엔진 시작
```

## 보안: .gitignore

API 키 파일(`*.json` 서비스 계정 키)과 로컬 데이터는 `.gitignore`에 추가했다.

```
dpems-*.json        # Google API 서비스 계정 키
dpems_data.json     # 로컬 태스크 데이터
dpems_log.json      # 로컬 로그
logs/
__pycache__/
```

---

다음 Phase에서는 Android 앱을 만들어 실제 멀티 기기 동기화를 완성할 계획이다.
