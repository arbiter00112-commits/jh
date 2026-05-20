# Tello Dashboard Display Reference

이 문서는 웹 대시보드가 화면에 띄우는 모든 주요 요소와 각 값의 의미, 데이터 출처를 정리합니다.

대시보드는 `/ws` WebSocket으로 `DashboardSnapshot` JSON을 받아 렌더링합니다. 주요 입력은 `snapshot.tello`, `snapshot.tracking`, `snapshot.history`, `snapshot.events`, `snapshot.hit_count`, `snapshot.shot_count`, `snapshot.jetson_status`, `snapshot.last_received_age`입니다.

## 전체 화면 구조

### Hit Overlay

- 화면 중앙 경고 오버레이입니다.
- 표시 문구: `HIT CONFIRMED`
- 표시 조건:
  - `snapshot.hit_count`가 이전 값보다 증가했을 때
  - 또는 현재 packet의 `tracking.laser.hit_detected`가 `false -> true`로 바뀌었을 때
- 표시 시간: 약 2.6초
- 상세 문구 구성:
  - `Hits <count>`
  - tracking state
  - confidence
  - tracking error px
  - frame id

예:

```text
Hits 3 | LOCKED | conf 0.82 | err 41 px | frame 1234
```

### Top Bar

- 제목: `Tello Dashboard`
- WebSocket 연결 상태:
  - `WebSocket: connecting`
  - `WebSocket: connected`
  - `WebSocket: disconnected, reconnecting`
- Node status:
  - `Jetson`: `snapshot.jetson_status === "CONNECTED"`이면 online
  - `Ultra96`: `tracking.ultra_ps`, `tracking.ultraPs`, `tracking.ultraps` 중 하나가 있고 Jetson이 connected이면 online
  - `RPi`: 항상 online
- Live clock:
  - 브라우저 로컬 시간
  - `LIVE HH:MM:SS`
- Jetson badge:
  - `snapshot.jetson_status`
  - 값: `CONNECTED`, `STALE`, `DISCONNECTED`

## Status Panel

### Key Stats

상단의 큰 요약 지표입니다.

| 표시명 | 의미 | 데이터 출처 / 계산 |
| --- | --- | --- |
| `Lock-on` | 현재 추적 단계 요약 | `displayState(tracking, laser)` |
| `Laser` | 레이저 상태 | `hit_detected=true`이면 `HIT`, 아니면 `armed=true`이면 `ON`, 그 외 `OFF` |
| `Hits` | 실제 명중 횟수 | `snapshot.hit_count`, fallback `latest.history.hit_count`, 기본 `0` |
| `Shots` | Jetson fire 입력/발사 시도 횟수 | `snapshot.shot_count`, fallback `latest.history.shot_count`, 기본 `0` |
| `Error` | 현재 tracking error 크기 | `history.tracking_error_px` 또는 `sqrt(error.x_px^2 + error.y_px^2)` |
| `Latency` | Jetson timestamp와 Pi 수신/표시 시간 차이 | latest history `latency_ms` |

색상/강조 기준:

- `Laser`, `Hits`: hit flash 중이거나 `laser.hit_detected=true`이면 danger/pulse
- `Laser`: `laser.armed=true`이면 danger
- `Error`: `80 px` 이하이면 success, 그 외 warn
- `Latency`: `120 ms` 이하이면 success, 그 외 warn

### State Strip

상태 진행 표시입니다.

표시 단계:

```text
IDLE -> SEARCH -> TRACK -> LOCK -> FIRE
```

`tracking.state`와 `laser` 상태를 다음 규칙으로 압축합니다.

| 표시 상태 | 조건 |
| --- | --- |
| `FIRING` | `laser.hit_detected`, `laser.armed`, 또는 raw state에 `FIR` 포함 |
| `LOCKED` | raw state에 `LOCK` 포함 |
| `TRACKING` | raw state에 `TRACK` 또는 `DETECT` 포함 |
| `SEARCHING` | raw state에 `SEARCH`, `SCAN`, 또는 `LOST` 포함 |
| `IDLE` | 위 조건에 해당하지 않음 |

### Status Meters

막대형 상태 미터입니다.

| 표시명 | 범위 | 의미 / 출처 |
| --- | --- | --- |
| `Drone Battery` | `0-100%` | `snapshot.tello.battery` |
| `YOLO Confidence` | `0-1` | `tracking.confidence`, fallback latest history `confidence` |
| `Audio Confidence` | `0-1` | `tracking.audio.confidence`, fallback latest history `audio_confidence` |
| `Signal Quality` | `0-100%` | `100 - last_received_age * 50`, `0-100`으로 clamp |

`Signal Quality`는 Jetson telemetry가 최근에 들어왔는지를 보여주는 단순 품질 지표입니다. `last_received_age`가 없으면 `0%`입니다.

### Detail Rows

작은 key/value 목록입니다.

| 표시명 | 의미 | 데이터 출처 / 표시 |
| --- | --- | --- |
| `Tello connected` | Tello 연결 여부 | `snapshot.tello.connected` -> `yes/no/-` |
| `Airborne` | 이륙 상태 | `snapshot.tello.airborne` -> `yes/no/-` |
| `Speed mode` | Tello speed 설정 | `snapshot.tello.speed` + `cm/s` |
| `Last command` | 마지막 조종 명령 | `snapshot.tello.last_command` |
| `Jetson status` | Jetson telemetry 상태 | `snapshot.jetson_status` |
| `Telemetry rate` | 최근 1초 telemetry 수신 개수 | latest history `telemetry_rate_hz` + `Hz` |
| `Last received` | 마지막 telemetry 이후 경과 시간 | `snapshot.last_received_age` + `sec` |
| `Motor angle` | 모터/팬 방향 | `motorDirectionDeg(tracking)` |
| `Audio bearing` | 오디오 방향 | `tracking.audio.direction_deg` |
| `Target size` | bbox가 frame에서 차지하는 면적 비율 | `(bbox.w * bbox.h) / (frame.width * frame.height)` |

## Motor-Centered Radar

### Radar Canvas

모터 방향 기준 레이더입니다.

표시 요소:

- 배경
- 3단 원형 거리 grid
- 8방향 방사선
- 방위 라벨:
  - `N 0`
  - `W 90`
  - `S 180`
  - `E 270`
- 카메라 FOV wedge
  - 고정 FOV: `110 deg`
  - 반각: `55 deg`
  - 중심: motor direction
- 중심점
  - hit 중이면 빨간색 pulse
- motor marker
  - motor direction 방향으로 선과 점 표시
  - locked/firing/hit 상태면 pulse 표시
  - hit 중이면 marker 근처에 `HIT` 텍스트 표시
- audio bearing
  - `Lock-on` 표시 상태가 `SEARCHING`일 때만 표시
  - `tracking.audio.direction_deg` 방향으로 화살표 표시
  - `tracking.audio.confidence`가 높을수록 더 진하고 두껍게 표시

### Radar Readout

레이더 상단 우측 텍스트입니다.

tracking이 없을 때:

```text
Motor - | FOV 110 deg | Bearing only
```

tracking이 있을 때:

```text
Motor <deg> | FOV 110 deg | Bearing only | Audio <deg or ->
```

### Motor Direction 계산

`motorDirectionDeg(tracking)`은 아래 우선순위로 방향값을 고릅니다.

1. `tracking.ultra_ps.motor_deg`
2. `motor_deg`가 없으면 `tracking.ultra_ps.front_pan`과 `tracking.ultra_ps.pan_tick`으로 fallback 계산
3. `tracking.ultra_ps.motor_direction_deg`
4. `tracking.ultra_ps.fan_deg`
5. `tracking.ultra_ps.heading_deg`
6. `tracking.ultra_ps.direction_deg`
7. `tracking.ptz.pan_deg`

`motor_deg`는 Jetson이 계산해서 보내는 최종 표시 각도이므로 Pi 대시보드는 그대로 표시합니다. tick fallback 계산식은 다음과 같습니다.

```text
motor_deg = ((front_pan - pan_tick) * 360 / 4096) % 360
```

## Live Graphs

최근 최대 300개 history sample을 그립니다.

### Graph Legend / Series

| 라벨 | key | 범위 | 색상 | 의미 |
| --- | --- | --- | --- | --- |
| `YOLO FPS` | `fps` | `0-35` | blue | Jetson tracker FPS |
| `Tracking Error` | `tracking_error_px` | `0-720` | orange | 타겟 중심 오차 크기 |
| `Latency` | `latency_ms` | `0-500` | yellow | Jetson timestamp 기준 지연 |
| `Audio Confidence` | `audio_confidence` | `0-1` | cyan | 오디오 방향 검출 신뢰도 |

그래프는 각 값을 지정 범위로 normalize해서 canvas 높이에 맞춰 그립니다. 숫자가 아닌 sample은 건너뜁니다.

## Events Panel

최근 event log를 표시합니다.

- 표시 개수: 마지막 24개
- 정렬: 최신이 위
- 각 event 표시:
  - local time
  - category
  - message
- 가장 최신 event에는 `fresh` 스타일 적용
- message에 `HIT`가 포함되면 hit event 스타일 적용

### Event Category 분류

| Category | 조건 |
| --- | --- |
| `ERROR` | message에 `ERROR` 또는 `DISCONNECTED` 포함 |
| `WARNING` | message에 `WARN`, `TIMEOUT`, 또는 `LOST` 포함 |
| `AUDIO` | message에 `AUDIO` 포함 |
| `VISION` | message에 `VISION` 또는 `TARGET` 포함 |
| `MOTOR` | message에 `MOTOR` 포함 |
| `LASER` | message에 `LASER` 또는 `HIT` 포함 |
| `SYSTEM` | 위 조건에 해당하지 않음 |

## Snapshot 데이터 출처

대시보드는 서버가 보내는 `DashboardSnapshot.to_dict()` 결과를 사용합니다.

주요 필드:

```json
{
  "timestamp": 1715600000.123,
  "tello": {
    "connected": true,
    "airborne": false,
    "battery": 76,
    "speed": 30,
    "last_command": "takeoff",
    "command_rate": 1.0
  },
  "jetson_status": "CONNECTED",
  "last_received_age": 0.05,
  "tracking": {
    "timestamp": 1715600000.100,
    "target_found": true,
    "state": "LOCKED",
    "fps": 24.8,
    "confidence": 0.87,
    "bbox": {
      "w": 100,
      "h": 80
    },
    "frame": {
      "width": 1280,
      "height": 720
    },
    "error": {
      "x_px": -70,
      "y_px": -60
    },
    "audio": {
      "direction_deg": 35.0,
      "confidence": 0.62
    },
    "ultra_ps": {
      "motor_deg": 90.0,
      "front_pan": 2048,
      "pan_tick": 1024
    },
    "laser": {
      "armed": true,
      "fired": false,
      "shot_count": 7,
      "hit_detected": false
    }
  },
  "history": [],
  "events": [],
  "hit_count": 0,
  "shot_count": 7
}
```

## Count 기준

### Shots

`Shots`는 현재 Pi 대시보드 서버 세션에서 발생한 Jetson fire 입력/발사 시도 횟수입니다.

권장 입력:

- Jetson이 f 입력마다 `laser.shot_count`를 누적 증가
- 또는 발사 순간 `laser.fired=true`를 보내고 다음 packet에서 `false`로 내림

대시보드는 처음 받은 `laser.shot_count` 값을 기준점으로 잡고, 이후 증가분만 `Shots`에 더합니다. 그래서 Jetson이 이전 실행에서 `shot_count=33`을 계속 보내더라도 새 Pi 대시보드 세션은 `Shots=0`에서 시작합니다.

`laser.armed=true`만으로는 증가하지 않습니다. `laser.hit_detected=true`만으로도 증가하지 않습니다.

### Hits

`Hits`는 실제 명중 판정 횟수입니다.

- `laser.hit_detected=false -> true` 상승 엣지에서 증가
- 같은 hit 동안 `true`가 반복되어도 한 번만 증가

## Jetson Status 기준

서버의 `TelemetryStore`가 마지막 Jetson telemetry 수신 시각 기준으로 계산합니다.

| 상태 | 조건 |
| --- | --- |
| `CONNECTED` | 마지막 수신 후 `1.0 sec` 이하 |
| `STALE` | 마지막 수신 후 `1.0 sec` 초과, `3.0 sec` 이하 |
| `DISCONNECTED` | 아직 수신 없음 또는 마지막 수신 후 `3.0 sec` 초과 |
