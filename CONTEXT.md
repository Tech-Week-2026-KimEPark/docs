# CONTEXT.md — PNU TECH WEEK 2026 Search & Rescue (Webots)

> AI에게: 이 문서는 팀 공용 규칙입니다. 코드를 작성할 때 아래 좌표 규칙, 파일 구조, 함수 인터페이스를 반드시 지키십시오. 변경이 필요하면 코드보다 먼저 이유를 설명하십시오.

---

## 1. 과제

- 미지의 환경을 탐색하고, 지정된 대상을 찾아 이동한 뒤, 시작 지점으로 복귀하는 자율 로봇 시스템
- 정적 장애물과 이동하는 사람과의 충돌 금지

| 조건 | 내용 |
|---|---|
| 지도 | 사전 제공 없음 |
| 로봇 현재 위치 | 제공 안 됨 → GPS, Supervisor(`getSelf()`) 사용 금지 |
| 시작 위치·방향 | 제공됨 (config.py에 입력) |
| 대상 위치 | 모름 |
| 대상 종류·시각 특징 | 제공됨 (예: 빨간 사과) |

**미확정 (대회 시작 후 확인)**: "목적지"가 대상 위치인지 별도 좌표인지, 대상 개수, 제한 시간, 채점 기준

---

## 2. 개발 환경

- Webots R2025a, Python 3.10 (Ubuntu 22.04 기준, Windows/macOS 가능)
- numpy 1.23.5, opencv-python 4.8.0.74, matplotlib 3.7.5
- **YOLO11n 사용 (필수)**: torch 2.8.0, torchvision 0.23.0, ultralytics, 가중치 `models/YOLO/yolo11n.pt` (컨트롤러 폴더 기준 `../../models/YOLO/yolo11n.pt`)
- `basicTimeStep` 64 ms
- 컨트롤러 위치: `controllers/<이름>/<이름>.py`, 월드에서 로봇의 `controller` 필드에 이름 지정

**연습 맵 `worlds/apartment.wbt`**

- 크기 약 12.4 × 13.1 m (x: −12.4 ~ 0, y: −13.1 ~ 0), 벽·창문·문으로 된 방 여러 개
- 로봇 시작: (−0.3, −7.5), 방향 π (서쪽), 동쪽 벽 문 앞
- 바닥 사과 7개 (z = 0.05): 빨강 2, 초록 2, 보라 2, 주황 1
- 방해 요소: 식탁 위 Apple·Orange 모델 (z ≈ 0.83), 축구공, 캔·맥주병(바닥), 고양이, 소 모형
- 보행자 1명: 0.2 m/s로 복도·방 사이를 정해진 경로로 왕복

---

## 3. 로봇과 장치 (TurtleBot3 Burger)

| 장치 이름 | 사용법 | 비고 |
|---|---|---|
| `left wheel motor`, `right wheel motor` | `setPosition(float("inf"))` 후 `setVelocity(rad/s)` | 최대 약 6.67 rad/s (0.22 m/s) |
| 바퀴 엔코더 | `motor.getPositionSensor()` → `enable(ts)` → `getValue()` | 누적 회전각 [rad] |
| `LDS-01` (라이다) | `enable(ts)` → `getRangeImage()` | 360개, 범위 0.12 ~ 3.5 m, 반사 없으면 `inf` |
| `camera` | `enable(ts)` → `getImage()` | 640 × 480, 수평 화각 1.0472 rad (60°), BGRA 바이트 |
| `compass`, `gyro`, `accelerometer` | `enable(ts)` → `getValues()` | 예제 `tb3_teleop_sensors` 기준. 대회 월드에 있는지 `getDevice` 결과로 확인 |

**상수**

```python
WHEEL_RADIUS = 0.033      # m
WHEEL_SEPARATION = 0.160  # m
ROBOT_RADIUS = 0.105      # m
```

**라이다 인덱스 규칙 (예제 `tb3_lidar.py`로 확인)**

| 인덱스 | 0 | 90 | 180 | 270 |
|---|---|---|---|---|
| 방향 | 뒤 | 왼쪽 | 정면 | 오른쪽 |

로봇 좌표계 각도: `angle_i = π − i · 2π / 360`

**카메라 이미지 변환**

```python
frame = np.frombuffer(camera.getImage(), np.uint8).reshape((H, W, 4))
bgr = cv2.cvtColor(frame, cv2.COLOR_BGRA2BGR)
```

**주의: 라이다 사각지대.** 라이다는 로봇 위쪽의 수평면 하나만 측정합니다. 사과(높이 약 0.1 m), 캔, 맥주병처럼 낮은 물체는 라이다에 검출되지 않습니다. 대상까지의 거리는 카메라로 추정합니다.

**사용 금지**: `Supervisor`, `getSelf()`, GPS. `tb3_ground_truth`는 오도메트리 오차 확인용으로만 사용합니다.

---

## 4. 좌표·단위 규칙 (모든 모듈 공통)

| 항목 | 규칙 |
|---|---|
| 월드 좌표 | Webots 월드 좌표 (x, y) [m]. 시작 위치로 초기화 |
| 방향 θ | [rad], 반시계 +, 범위 [−π, π]. 각도 차이는 반드시 `atan2(sin, cos)`로 정리 |
| 로봇 좌표 | +x 정면, +y 왼쪽 |
| 격자 지도 | `grid[row][col]`, **row ↔ y, col ↔ x**, 해상도 0.05 m, 시작점 중심 32 m 정사각형 |
| 격자 값 | 내부는 log-odds (베이지안 점유 갱신, 수치는 9장). 외부 공개값은 −1 모름 / 0 빈칸 / 1 장애물 |
| 속도 명령 | (v [m/s], ω [rad/s]) |
| 바퀴 변환 | `wl = (v − ω·L/2) / R`, `wr = (v + ω·L/2) / R`. 최대값 초과 시 비율 유지하며 축소 |
| 시간 | `robot.getTime()` 시뮬 시간. 모든 타임아웃은 시뮬 시간 기준 |
| 카메라 초점거리 | `f = (W/2) / tan(FOV/2) ≈ 554.3 px` |
| 카메라 방위각 | `β = −atan((cx − W/2) / f)` (왼쪽 +) |
| 카메라 거리 | `d ≈ f · D / w` (D: 대상 지름 약 0.095 m, w: 바운딩 박스 폭 px). 색 분할이면 w = 외접원 지름 |
| 대상 월드 좌표 | `(x + d·cos(θ+β), y + d·sin(θ+β))` |

---

## 5. 파일 구조와 담당

```
controllers/sar_main/
  sar_main.py            (A) 진입점: RobotIO 생성, Mission 루프
  sar/config.py          (A) 모든 설정값
  sar/robot_io.py        (A) Webots 장치 래핑. Webots API는 이 파일에서만 호출
  sar/mission.py         (A) 상태 머신
  sar/grid_map.py        (B) 점유 격자, 라이다 갱신, 팽창, 프론티어
  sar/planner.py         (B) A*, 프론티어 목표 선택
  sar/perception.py      (C) YOLO11n 검출 + 색 판별, 거리·방위 추정, 연속 확인
  sar/odometry.py        (D) 엔코더 오도메트리 + 방향 칼만 필터(나침반 갱신), (선택) 스캔-지도 매칭
  sar/local_control.py   (D) pure pursuit + 라이다 안전 필터
  sar/viz.py             (C) 지도·경로 그림 (디버깅, 발표용)
controllers/hsv_tuner/
  hsv_tuner.py           (C) 키보드 조종 + HSV 트랙바 + 프레임 저장
```

한 파일은 한 사람만 수정합니다. main 병합은 A만 수행합니다.

---

## 6. 모듈 인터페이스

```python
# robot_io.py
class RobotIO:
    def step(self) -> bool                         # robot.step(ts) != -1
    def time(self) -> float
    def encoders(self) -> tuple[float, float]      # (좌, 우) 누적 rad
    def compass(self) -> tuple | None              # 장치 없으면 None
    def lidar(self) -> list[float] | None          # 360개, 인덱스 규칙은 3장
    def camera_bgr(self) -> "np.ndarray | None"    # (480, 640, 3) uint8
    def drive(self, v: float, w: float) -> None    # 바퀴 속도 변환·제한 포함

# odometry.py
class Odometry:
    def __init__(self, x: float, y: float, theta: float)
    def update(self, enc_l: float, enc_r: float, compass=None) -> None
    # 방향 칼만 필터: 예측(엔코더 dθ, P += Q) → 갱신(나침반 z, K = P/(P+R))
    def pose(self) -> tuple[float, float, float]
    def heading_var(self) -> float                 # 방향 분산 P (디버깅·발표용)
    def correct(self, dx: float, dy: float, dth: float) -> None   # (선택) 스캔 매칭 보정값 적용

# grid_map.py
class GridMap:
    def update(self, pose, ranges) -> None
    def to_cell(self, x, y) -> tuple[int, int]     # (row, col)
    def to_world(self, row, col) -> tuple[float, float]
    def layers(self) -> tuple                      # (occ, blocked, soft, unknown) bool 배열
    def frontiers(self) -> list                    # [(크기, [(row, col), ...]), ...]

# planner.py
def plan(grid, start_xy, goal_xy, allow_unknown=False) -> list[tuple] | None   # [(x, y), ...]
def choose_frontier(grid, pose, blacklist) -> tuple | None                     # (x, y)

# local_control.py
def pure_pursuit(pose, path, lookahead) -> tuple[float, float, bool]   # (v, w, reached)
def safety_filter(v, w, ranges) -> tuple[float, float, bool]           # (v, w, blocked)

# perception.py
class TargetDetector:
    def __init__(self, color: str, model_path: str)   # YOLO 모델은 생성 시 1회만 로드
    def detect(self, bgr) -> dict | None
    # 반환: {"cx", "cy", "w", "h", "conf", "cls", "color", "dist", "bearing", "source"}
    #   source: "yolo" 또는 "color" (YOLO가 놓쳐 색 분할로 찾은 경우)
    def to_world(self, det, pose) -> tuple[float, float]
class Confirm:
    def update(self, seen: bool, xy, dist: float = None) -> tuple[bool, tuple | None]   # (확정 여부, 위치 추정)
    # 위치 추정 = 거리 가중 평균: w = 1 / max(dist, 0.3)**2  (가까이서 본 관측일수록 큰 가중치)

# mission.py
class Mission:
    def tick(self) -> None
    # 매 스텝 1회: 센서 → 오도메트리 → 지도 → 인식 → 상태 머신 → 안전 필터 → drive
```

---

## 7. 상태 머신

```
INIT_SPIN → EXPLORE → APPROACH → RESCUE ─┬→ (대상 남음) EXPLORE
                                         └→ (모두 찾음) GO_GOAL* → RETURN → DONE
* GOAL_XY가 설정된 경우에만
```

| 상태 | 동작 | 다음 상태로 가는 조건 |
|---|---|---|
| INIT_SPIN | 제자리 360° 회전. 주변 지도 채우기, 나침반 부호·오프셋 보정 | 한 바퀴 완료 |
| EXPLORE | 프론티어 탐색 | 대상 연속 확인 → APPROACH, 프론티어 없음 → RETURN |
| APPROACH | 대상 앞 0.35 m까지 이동 후 정면 정렬 | 도착 → RESCUE |
| RESCUE | 2초 정지, 위치 기록 | 개수 충족 여부로 분기 |
| GO_GOAL | 목적지 좌표로 이동 | 도착 → RETURN |
| RETURN | 시작점으로 이동 (모르는 칸 통과 허용) | 0.12 m 이내 → DONE |
| DONE | 정지, 지도 저장, 결과 출력 | — |

**공통 안전 규칙**

- 남은 시간 < `RETURN_RESERVE` → 즉시 RETURN
- 4초 동안 5 cm 미만 이동 → RECOVERY (후진 후 넓은 쪽으로 회전, 재계획)
- 안전 필터가 3초 이상 전진 차단 → 재계획
- 프론티어 목표에 30초 안에 도달 못 함 → 블랙리스트 등록

---

## 8. 설계 결정

| 항목 | 결정 | 이유 |
|---|---|---|
| 위치 추정 | 엔코더 오도메트리 + **방향 1차원 칼만 필터** (예측: 엔코더 dθ, 갱신: 나침반) | 강의 전제: 시작 위치를 알면 오도메트리로 추정 가능(시뮬). 위치 오차의 주원인이 방향 오차이므로 방향부터 필터링 |
| 스캔-지도 매칭 도입 기준 | `tb3_ground_truth` 월드에서 한 바퀴 주행 후 위치 오차 **0.3 m 이상**이면 도입, 미만이면 생략 | 잘못된 매칭은 지도를 손상시킴. 필요성을 수치로 확인 후 결정 |
| 스캔-지도 매칭 방식 (도입 시) | 1~2초마다 현재 스캔을 지도에 대해 작은 범위(±0.1 m, ±3°) 격자 탐색, 점수 향상이 충분할 때만 `correct()` 적용 | ICP보다 구현이 단순하고 실패 시 영향 범위가 작음 |
| 파티클 필터 (MCL/AMCL) | 사용 안 함 | 3시간 안에 구현·튜닝 불가 |
| 대상 위치 추정 | 거리 가중 평균 | 카메라 거리 추정은 가까울수록 정확 |
| 나침반 보정 | 시작 회전 중 엔코더 방향과 비교해 부호·오프셋 자동 추정 | 나침반 값 규약이 예제마다 달라 수동 설정 시 오류 위험 |
| 대상 인식 | YOLO11n으로 후보 검출 → 상자 안 HSV 색 비율로 대상 색 판별 | YOLO 사용이 대회 방침. 연습 맵에 색만 다른 사과가 섞여 있어 YOLO 클래스만으로 구분 불가 |
| YOLO 클래스 | 47 apple, 49 orange, 32 sports ball을 후보로 사용 | 주황·보라·초록 사과가 apple이 아닌 다른 클래스로 잡힐 수 있음 |
| YOLO 보완 | YOLO가 놓치면 같은 프레임에 색 분할(원형도·면적 조건)로 대체 검출 | 먼 거리의 작은 사과, COCO 학습 분포와 다른 색의 사과 |
| YOLO 실행 주기 | 4스텝(약 0.26초)마다 1회, `verbose=False` | CPU 추론 시간만큼 시뮬 진행이 느려짐 |
| 오탐 방지 | 원형도, 최소 면적, 화면 가운데선보다 확실히 위인 덩어리 제외, 4프레임 연속 확인 | 식탁 위 과일, 주황빛 바닥 |
| 대상 거리 | 바운딩 박스 폭(w)으로 추정: `d ≈ f · D / w` | 사과가 라이다에 보이지 않음 |
| 사람 회피 | 전역 경로(A*)는 지도 기준, 매 스텝 라이다 안전 필터가 최종 속도 결정 | 지도에 없는 움직이는 장애물 |
| 지도 | 베이지안 점유 격자 (log-odds) | 강의 Bayesian Filter를 로그 공간 덧셈으로 구현. 상·하한 고정으로 사람이 지나간 자국이 약 8스캔(0.8초) 뒤 삭제 |
| 경로 | A* 8방향, 팽창 반경 = 로봇 반지름 + 0.07 m, 벽 근처 추가 비용 | 통로 가운데 주행 |
| 경로 추종 | pure pursuit, look-ahead 0.35 m, 55° 이상 틀어지면 제자리 회전 | 강의 방식과 동일 |

---

## 9. 기본 설정값 (config.py)

```python
START_X, START_Y, START_THETA = -0.3, -7.5, math.pi   # 대회 당일 입력
TARGET_COLOR = "red"          # red | green | purple | orange | custom
TARGET_COUNT = 1
GOAL_XY = None                # 별도 목적지 좌표가 있으면 (x, y)
TIME_LIMIT = 900.0            # s, 시뮬 시간
RETURN_RESERVE = 150.0        # s

V_MAX = 0.18                  # m/s
V_APPROACH = 0.10
W_MAX = 1.8                   # rad/s
LOOKAHEAD = 0.35              # m
GOAL_TOL = 0.15               # m
SAFETY_MARGIN = 0.06          # m, 몸체 바깥 여유
STOP_DIST = 0.20              # m, 정면 즉시 정지 거리

MAP_RES = 0.05                # m
L_OCC = 0.9                   # 맞은 칸 log-odds 증분 (확률 약 0.71)
L_FREE = -0.4                 # 지나간 칸 log-odds 증분 (확률 약 0.40)
L_MIN, L_MAX = -2.0, 3.5      # 확률 약 0.12 ~ 0.97에서 고정
OCC_THRESHOLD = 0.3           # 이보다 크면 장애물 (확률 약 0.57)

HEADING_Q = 0.01 ** 2         # 방향 예측 잡음 (한 스텝, rad^2)
HEADING_R = 0.05 ** 2         # 나침반 관측 잡음 (rad^2)
SCAN_MATCH = False            # 오차 0.3 m 이상 확인 시 True
SCAN_MATCH_PERIOD = 1.5       # s
LIDAR_MIN, LIDAR_MAX = 0.12, 3.5
INFLATE = ROBOT_RADIUS + 0.07
MIN_FRONTIER_CELLS = 6

CAMERA_FOV = 1.0472
YOLO_MODEL = "../../models/YOLO/yolo11n.pt"
YOLO_DEVICE = "cpu"           # NVIDIA GPU면 "cuda"
YOLO_CLASSES = [47, 49, 32]   # apple, orange, sports ball
YOLO_CONF = 0.20
YOLO_EVERY = 4                # 몇 스텝마다 추론
COLOR_RATIO_MIN = 0.25        # 상자 안 대상 색 픽셀 비율 하한
USE_COLOR_FALLBACK = True     # YOLO가 놓치면 색 분할로 대체
TARGET_DIAMETER = 0.095       # m
MIN_BLOB_AREA = 60            # px
CONFIRM_FRAMES = 4
APPROACH_DIST = 0.35          # m
```

---

## 10. HSV 초기값 (OpenCV 기준: H 0~179, S·V 0~255)

YOLO 상자 안 색 판별과 색 분할 대체 검출에 공통으로 사용합니다. 현장 조명에서 `hsv_tuner`로 반드시 다시 조정하십시오.

| 색 | 범위 | 참고 |
|---|---|---|
| red | (0,120,60)~(8,255,255) ∪ (170,120,60)~(179,255,255) | baseColor 1 0 0 |
| orange | (12,170,120)~(28,255,255) | baseColor 1 0.73 0. 나무 바닥과 겹칠 수 있어 채도 하한을 높게 |
| purple | (125,90,50)~(160,255,255) | baseColor 0.56 0 1 |
| green | (30,90,50)~(60,255,255) | 텍스처 사과. 현장 확인 필수 |

---

## 11. 코딩 규칙

1. Python 3.10, numpy·opencv·ultralytics만 사용 (scipy 없이 동작)
   - YOLO 모델 로드는 컨트롤러 시작 시 1회. 매 프레임 로드 금지
   - `model.predict(..., verbose=False)`로 로그 출력 억제
2. Webots API는 `robot_io.py`에서만 호출
3. 숫자 상수는 `config.py`에만 정의
4. A*·프론티어 선택 같은 무거운 계산은 필요할 때만 실행 (목표 변경, 경로 차단, 2초 주기)
5. 모든 모듈에 Webots 없이 실행되는 `if __name__ == "__main__":` 단독 테스트 포함
6. 센서 값이나 경로가 None일 때의 동작을 반드시 정의 (예외로 종료 금지)
7. 상태 전환 로그 형식: `[t=12.3s] EXPLORE -> APPROACH (red at -5.34,-10.54)`
8. 모든 while 루프에 반복 상한 또는 시간 제한

---

## 12. 대회 시작 후 확인할 것

- [ ] 대회 맵이 apartment.wbt와 같은지
- [ ] 시작 위치·방향 값
- [ ] 대상 색·개수, "목적지"의 의미 (대상 위치 / 별도 좌표)
- [ ] 제한 시간, 채점 기준 (시간 점수, 충돌 감점)
- [ ] 대회 월드 로봇에 `compass`, `gyro`가 있는지 (`getDevice` 결과 None 여부). 나침반이 없으면 칼만 필터 갱신 단계 생략, 자이로가 있으면 예측을 자이로로
- [ ] (D, 0:40까지) `tb3_ground_truth` 월드에서 오도메트리 위치 오차 측정 → `SCAN_MATCH` 결정
- [ ] 스캔 매칭, 사전 작성 코드 허용 범위
- [ ] YOLO 방침 세부: 사전학습 가중치 그대로 사용인지, 추가 학습 허용인지
- [ ] (C, 첫 20분) `tb3_teleop_yolo`로 색깔별 사과가 어떤 클래스·신뢰도로 잡히는지 기록 (특히 보라·초록·주황 사과, 1 m·2 m·3 m 거리)
- [ ] (C) 추론 1회 시간 측정. 느리면 `YOLO_EVERY` 증가 또는 `imgsz=480`
