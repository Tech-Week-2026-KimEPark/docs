# CONTEXT.md: PNU TECH WEEK 2026 Search & Rescue (Webots)

> AI에게: 이 문서는 팀 공용 규칙입니다. 코드를 작성할 때 아래 과제 조건, 좌표 규칙, 파일 구조, 함수 인터페이스를 반드시 지키십시오. 변경이 필요하면 코드보다 먼저 이유를 설명하십시오.

최종 갱신: 2026-09-30. 기준 코드: sar-robot `main` `71dc415`. 사람용 원본 문서는 docs 저장소 `human/reference/sar-과제-구현-기준.md`입니다.

---

## 1. 과제

### 1.1 확정 목표 (대회 공지)

- 빨간 사과 2개를 모두 찾아 각 사과 앞 0.35 m까지 이동한 뒤 시작 지점으로 복귀함
- 정적 장애물, 이동하는 사람과 충돌 금지
- 지도 시각화 권장. 팀은 지도 이미지 저장을 필수 기능으로 정함

| 조건 | 내용 |
|---|---|
| 지도 | 사전 제공 없음 |
| 로봇 현재 위치 | 제공 안 됨 → GPS, Supervisor(`getSelf()`) 사용 금지 |
| 시작 위치·방향 | 제공됨 (config.py에 입력) |
| 대상 | 빨간 사과 2개. 위치 모름 |
| 목적지 | 시작 지점 (별도 목적지 좌표 없음, `GOAL_XY = None`) |

### 1.2 평가 기준과 제출물

| 항목 | 내용 |
|---|---|
| 평가 기준 | 완전성, 창의성, 확장성(코드로 구현, 글로 구현) |
| 창의성 주제 | 특정 자연재해·사고 상황에서 로봇의 행동 |
| 제출물 | 컨트롤러, Python 파일, README |
| README | 평가 환경에서 그대로 시연할 수 있는 전체 절차 포함. 평가 비중이 큼 |

창의성 코드 구현 후보는 위험 구역 회피(`NO_GO_ZONES`)와 구조 보고서 출력입니다. 설정값 1개로 켜고 끌 수 있게 구현해 기본 완주에 영향을 주지 마십시오.

**미확정 (대회 시작 후 확인)**: 제한 시간, 채점 세부(시간 점수, 충돌 감점), 제출 형태(저장소 전체 또는 컨트롤러 폴더), 평가 PC 사양

---

## 2. 개발 환경

- Webots R2025a, Python 3.10 (Ubuntu 22.04 기준, Windows/macOS 가능)
- numpy 1.23.5, opencv-python 4.8.0.74, matplotlib 3.7.5 (`requirements.txt`, Intro 과정 기준)
- **YOLO11n 사용 (필수)**: torch 2.8.0, torchvision 0.23.0, ultralytics 8.4.166, 가중치 `models/YOLO/yolo11n.pt` (컨트롤러 폴더 기준 `../../models/YOLO/yolo11n.pt`)
- `basicTimeStep` 64 ms
- Webots Preferences의 Python command를 저장소 `.venv/bin/python` 절대 경로로 설정. `pip install -e .` 단계 없음
- 팀 컨트롤러: `controllers/sar_main/`. 월드에서 로봇의 `controller` 필드를 `sar_main`으로 지정

**연습 맵 `worlds/apartment.wbt`**

- 크기 약 12.4 × 13.1 m (x: −12.4 ~ 0, y: −13.1 ~ 0), 벽·창문·문으로 된 방 여러 개
- 로봇 시작: (−0.3, −7.5), 방향 π (서쪽), 동쪽 벽 문 앞. 기본 컨트롤러 `tb3_teleop`
- 바닥 사과 7개 (중심 z = 0.05): 빨강 2, 초록 2, 보라 2, 주황 1
- 빨간 사과 위치: (−12.02, −3.02), (−5.34, −10.54), 간격 약 10.1 m. 인식 오차 확인용이며 코드에 직접 입력 금지
- 방해 요소: 식탁 위 Apple 모델 2개·Orange 모델 (z ≈ 0.83), 축구공, 캔·맥주병(바닥), 고양이, 소 모형
- 보행자 1명: 0.2 m/s로 복도·방 사이를 정해진 경로로 왕복

---

## 3. 로봇과 장치 (TurtleBot3 Burger)

| 장치 이름 | 사용법 | 비고 |
|---|---|---|
| `left wheel motor`, `right wheel motor` | `setPosition(float("inf"))` 후 `setVelocity(rad/s)` | 최대 약 6.67 rad/s (0.22 m/s) |
| `left wheel sensor`, `right wheel sensor` | `motor.getPositionSensor()` → `enable(ts)` → `getValue()` | 누적 회전각 [rad] |
| `LDS-01` (라이다) | `enable(ts)` → `getRangeImage()` | 360개, 범위 0.12 ~ 3.5 m, 반사 없으면 `inf` |
| `camera` | `enable(ts)` → `getImage()` | 640 × 480, 수평 화각 1.0472 rad (60°), BGRA 바이트 |
| `compass`, `gyro`, `accelerometer` | `enable(ts)` → `getValues()` | TurtleBot3Burger PROTO(R2025a) 기본 포함 확인. 대회 월드에서도 `getDevice` 결과 확인 |

**상수**

```python
WHEEL_RADIUS = 0.033      # m
WHEEL_SEPARATION = 0.160  # m
ROBOT_RADIUS = 0.105      # m
```

**카메라 위치**: 로봇 중심 기준 전방 0.02 m, 높이 0.073 m, 수평 방향. 바닥 사과 중심(0.05 m)은 화면 가운데선보다 약간 아래에 보입니다.

**라이다 인덱스 규칙 (예제 `tb3_lidar.py`로 확인)**

| 인덱스 | 0 | 90 | 180 | 270 |
|---|---|---|---|---|
| 방향 | 뒤 | 왼쪽 | 정면 | 오른쪽 |

로봇 좌표계 각도: `angle_i = π − i · 2π / 360`

**카메라 이미지 변환** (`RobotIO.camera_bgr()`가 처리)

```python
frame = np.frombuffer(camera.getImage(), np.uint8).reshape((H, W, 4))
bgr = frame[:, :, :3].copy()   # cv2.cvtColor(frame, cv2.COLOR_BGRA2BGR)와 같음
```

**주의: 라이다 사각지대.** 라이다는 로봇 위쪽의 수평면 하나만 측정합니다. 사과(높이 약 0.1 m), 캔, 맥주병처럼 낮은 물체는 라이다에 검출되지 않습니다. 대상까지의 거리는 카메라로 추정합니다.

**사용 금지**: `Supervisor`, `getSelf()`, GPS. `tb3_ground_truth`는 오도메트리 오차 확인용으로만 사용합니다.

---

## 4. 좌표·단위 규칙 (모든 모듈 공통)

| 항목 | 규칙 |
|---|---|
| 월드 좌표 | Webots 월드 좌표 (x, y) [m]. 시작 위치로 초기화 |
| 방향 θ | [rad], 반시계 +, 범위 [−π, π]. 각도 차이는 반드시 `atan2(sin, cos)`로 정리 (`odometry.wrap()`) |
| pose | 튜플 `(x, y, theta)`. `Pose` 클래스 없음 |
| 로봇 좌표 | +x 정면, +y 왼쪽 |
| 격자 지도 | `grid[row][col]`, **row ↔ y, col ↔ x**, 해상도 0.05 m, 시작점 중심 32 m 정사각형 (640 × 640칸) |
| 격자 값 | 내부는 log-odds (베이지안 점유 갱신, 수치는 9장). 외부 공개값은 −1 모름 / 0 빈칸 / 1 장애물 |
| 속도 명령 | (v [m/s], ω [rad/s]) |
| 바퀴 변환 | `wl = (v − ω·L/2) / R`, `wr = (v + ω·L/2) / R`. 최대값 초과 시 비율 유지하며 축소 (`robot_io.wheel_speeds()`) |
| 시간 | `robot.getTime()` 시뮬 시간. 모든 타임아웃은 시뮬 시간 기준 |
| 카메라 초점거리 | `f = (W/2) / tan(FOV/2) ≈ 554.3 px` |
| 카메라 방위각 | `β = −atan((cx − W/2) / f)` (왼쪽 +) |
| 카메라 거리 | `d = f · D / (w · cos β)` (D: 대상 지름 0.095 m, w: 상자 폭과 높이 중 큰 값 px). 색 분할이면 w = 외접원 지름. `f · D / w`는 광축 방향 깊이이므로 cos β로 나눠 직선거리로 변환 |
| 대상 월드 좌표 | `(x + d·cos(θ+β), y + d·sin(θ+β))` |

---

## 5. 파일 구조와 담당

```
controllers/sar_main/
  sar_main.py            (통합) 진입점: RobotIO 생성, Mission 루프          [기동·로그만]
  sar/config.py          (통합) 모든 설정값                                  [구현]
  sar/robot_io.py        (통합) Webots 장치 래핑. Webots API는 이 파일에서만 호출 [구현]
  sar/mission.py         (통합) 상태 머신                                    [구현]
  sar/grid_map.py        (계획) 점유 격자, 라이다 갱신, 팽창, 프론티어        [구현]
  sar/planner.py         (계획) A*, 다익스트라 프론티어 선택·복귀 거리 지도    [구현]
  sar/perception.py      (인지) YOLO11n 검출 + 색 판별, 거리·방위 추정, 연속 확인 [구현]
  sar/viz.py             (인지) 지도·궤적·구조 위치 그림 저장                  [구현]
  sar/odometry.py        (행동) 엔코더 오도메트리 + 방향 칼만 필터, (선택) 스캔-지도 매칭 [구현]
  sar/local_control.py   (행동) pure pursuit + 라이다 안전 필터               [구현]
controllers/hsv_tuner/
  hsv_tuner.py           (인지) 키보드 조종 + HSV 트랙바 + 프레임 저장         [구현]
tests/test_<모듈>.py      모듈별 pytest
```

| 역할 | 담당자 | 백업 짝 |
|---|---|---|
| 통합 (A) | 김태훈 | 박상원 |
| 계획 (B) | 박상원 | 김태훈 |
| 인지 (C) | 이온규 | 김규민 |
| 행동 (D) | 김규민 | 이온규 |

한 파일은 한 사람만 수정합니다. main 병합은 통합 담당만 수행합니다. `hsv_tuner.py`는 `controllers/sar_main`을 `sys.path`에 추가해 `sar` 패키지를 사용합니다.

---

## 6. 모듈 인터페이스

구현된 모듈은 실제 코드 형식입니다. `[미구현]` 모듈은 이 형식으로 구현하십시오.

```python
# robot_io.py
def wheel_speeds(v, w) -> tuple[float, float]      # 바퀴 각속도 [rad/s], 최대값 초과 시 비율 유지 축소
class RobotIO:                                     # controller 모듈은 생성 시 import
    def step(self) -> bool                         # robot.step(ts) != -1
    def time(self) -> float
    def encoders(self) -> tuple[float, float]      # (좌, 우) 누적 rad
    def compass(self) -> tuple | None              # 나침반 원시 벡터. 장치 없으면 None
    def lidar(self) -> list[float] | None          # 360개, 인덱스 규칙은 3장
    def camera_bgr(self) -> "np.ndarray | None"    # (480, 640, 3) uint8
    def drive(self, v: float, w: float) -> None    # 바퀴 속도 변환·제한 포함

# odometry.py
def wrap(angle) -> float                           # atan2(sin, cos)로 [-π, π] 정리
class Odometry:
    def __init__(self, x: float, y: float, theta: float)
    def update(self, enc_l: float, enc_r: float, compass: float | None = None) -> None
    # 첫 호출은 기준값 저장. compass는 INIT_SPIN에서 보정한 방향 [rad]
    # 방향 칼만 필터: 예측(엔코더 dθ, P += Q) → 갱신(나침반 z, K = P/(P+R))
    def pose(self) -> tuple[float, float, float]
    def heading_var(self) -> float                 # 방향 분산 P (디버깅·발표용)
    def correct(self, dx: float, dy: float, dth: float) -> None   # (선택) 스캔 매칭 보정값 적용

# grid_map.py
class GridMap:
    def __init__(self, center_x=config.START_X, center_y=config.START_Y, size=config.MAP_SIZE, res=config.MAP_RES)
    def update(self, pose, ranges) -> None         # 매 step. None이나 유한하지 않은 pose는 무시
    def to_cell(self, x, y) -> tuple[int, int]     # (row, col)
    def to_world(self, row, col) -> tuple[float, float]   # 칸 중심
    def layers(self) -> tuple                      # (occ, blocked, soft, unknown) bool 배열
    def frontiers(self) -> list                    # [(크기, [(row, col), ...]), ...] 크기 내림차순
    def public(self) -> np.ndarray                 # 공개값 격자 int8 (-1/0/1). viz.render_map() 입력
    def clearance(self) -> np.ndarray              # 가장 가까운 장애물까지 거리 [m]
    def frontier_sizes(self) -> np.ndarray         # 칸별 프론티어 묶음 칸 수 (아니면 0)
    def invalidate(self) -> None                   # logodds·seen을 직접 수정한 뒤 호출
    version: int                                   # update()마다 증가
    plan_version: int                              # 장애물·확인 분류가 바뀔 때만 증가. 같으면 계획 캐시 재사용

# planner.py  (목표 1개: A*, 목표 여러 개: 다익스트라)
def plan(grid, start_xy, goal_xy, allow_unknown=False, field=None) -> list[tuple] | None
# [(x, y), ...] PATH_STEP 이하 간격, 첫 점 = start_xy. 경로 없음·NaN·지도 밖이면 None (예외 없음)
# 경로 선분이 지나가거나 닿는 모든 칸은 팽창 영역 밖 (다듬기 뒤에도 같은 규칙)
# 통과 불가 시작·목표: SNAP_RADIUS 안에서 사이에 장애물 칸이 없는 가장 가까운 칸으로 대체
#   목표를 옮기면 끝점은 대체 칸 중심 (path[-1] != goal_xy). 벽 반대편 칸은 선택하지 않음
# field: 목표·지도 상태·영역이 맞는 DistanceField만 A* 휴리스틱으로 사용. 아니면 무시
def choose_frontier(grid, pose, blacklist) -> tuple | None                     # (x, y)
def choose_frontier_path(grid, pose, blacklist) -> tuple | None                # ((x, y), 경로)
# 로봇 기준 다익스트라 1회로 점수 = 묶음 크기 / 경로 비용 최대 칸과 그 경로를 함께 반환
# None이면 도달 가능한 프론티어 없음. 목표는 유지하고 재계획은 plan(grid, pose, target) 사용
def path_length(path) -> float | None              # 실제 경로 길이 [m]. 시간 추정용
class DistanceField:                               # 기준점 다익스트라 1회
    def __init__(self, grid, origin_xy)            # RETURN: origin = 시작점. 사용 직전에 생성
    def distance(self, xy) -> float | None         # 가중 비용 [m] (벽 근처·모르는 칸 배수 포함, 길이 아님)
    def path(self, xy) -> list[tuple] | None       # xy → 기준점 경로. 지도가 바뀌었으면 None
    def is_current(self) -> bool                   # 생성 이후 plan_version 유지 여부
# 복귀: plan(grid, pose[:2], start_xy, allow_unknown=True, field=DistanceField(grid, start_xy))
# pure_pursuit 도착 판정은 GOAL_TOL(0.15 m). 복귀 판정 0.12 m는 미션에서 처리

# local_control.py
def pure_pursuit(pose, path, lookahead=config.LOOKAHEAD) -> tuple[float, float, bool]   # (v, w, reached)
# path가 비었거나 끝점이 GOAL_TOL 이내면 (0, 0, True). 목표 방위 55° 초과 시 제자리 회전
def safety_filter(v, w, ranges) -> tuple[float, float, bool]           # (v, w, blocked)
# 정면 ±25° 안 STOP_DIST 이내면 v = 0. ranges가 None이면 통과

# perception.py
class TargetDetector:
    def __init__(self, color: str, model_path: str | None, device=config.YOLO_DEVICE, ranges=None)
    # YOLO 모델은 생성 시 1회만 로드. 실패하면 load_error 기록 후 색 분할만 사용
    def detect(self, bgr) -> dict | None           # 가장 가까운 대상 1개
    def detect_all(self, bgr) -> list[dict]        # 가까운 순서 전체 목록
    # 항목: {"cx", "cy", "w", "h", "conf", "cls", "color", "dist", "bearing", "source"}
    #   source: "yolo" 또는 "color" (YOLO가 놓쳐 색 분할로 찾은 경우)
    def to_world(self, det, pose) -> tuple[float, float]
class Confirm:
    def update(self, seen: bool, xy, dist: float = None) -> tuple[bool, tuple | None]   # (확정 여부, 위치 추정)
    # CONFIRM_FRAMES회 연속 CONFIRM_MATCH_RADIUS 안에서 검출되면 확정. seen=False 1회에 기록 초기화
    # 위치 추정 = 거리 가중 평균: w = 1 / max(dist, 0.3)**2
    def reset(self) -> None
    def estimate(self) -> tuple[float, float] | None
def is_excluded(xy, found, radius=config.FOUND_EXCLUDE_RADIUS) -> bool   # 구조 완료 위치 주변 여부

# viz.py
def render_map(grid, to_cell, trajectory=(), path=(), rescued=(), start=None, pose=None, title="") -> np.ndarray
# grid는 공개값(-1/0/1), rescued는 (x, y, t) 목록
def save_map(path, image) -> bool

# mission.py
def fit_compass(samples) -> tuple[int, float, float] | None   # (sign, offset, scale). 시작 회전 표본으로 나침반 보정
def compass_heading(vec, sign, offset) -> float                # 보정된 방향 [rad]
class Mission:
    def __init__(self, io, odom, grid, detector, log=None)     # log: 로그 함수. sar_main이 print 전달
    def tick(self) -> None
    # 매 스텝 1회: 센서 → 오도메트리 → 지도 → 인식(YOLO_EVERY step) → 상태 머신 → 안전 필터 → drive
```

---

## 7. 상태 머신

```
INIT_SPIN → EXPLORE → APPROACH → RESCUE ─┬→ (구조 1개) EXPLORE 또는 저장한 후보로 APPROACH
                                         └→ (구조 2개) RETURN → DONE
EXPLORE에서 프론티어 없음 → RETURN
```

| 상태 | 동작 | 다음 상태로 가는 조건 |
|---|---|---|
| INIT_SPIN | 제자리 360° 회전. 주변 지도 채우기, 나침반 부호·오프셋 보정 | 한 바퀴 완료 |
| EXPLORE | 프론티어 탐색. 저장한 미구조 후보가 있으면 후보로 이동 | 대상 연속 확인 → APPROACH, 프론티어 없음 → RETURN |
| APPROACH | 대상 앞 0.35 m까지 이동 후 정면 정렬 | 도착 → RESCUE |
| RESCUE | 2초 정지, 위치·시각 기록, 지도 이미지 저장 | 구조 수 < 2 → EXPLORE, 구조 수 = 2 → RETURN |
| RETURN | 시작점으로 이동 (모르는 칸 통과 허용) | 0.12 m 이내 → DONE |
| DONE | 정지, 최종 지도 저장, 결과 출력 | — |

GO_GOAL 상태는 사용하지 않습니다(목적지 = 시작 지점).

**대상 2개 처리 규칙**

- 구조 완료 위치에서 `FOUND_EXCLUDE_RADIUS`(0.6 m) 이내로 추정된 검출은 무시 (`perception.is_excluded()`)
- 탐색·접근 중 확인한 두 번째 사과 위치는 후보로 저장. RESCUE 후 후보가 있으면 프론티어 탐색 없이 후보로 이동 (`detect_all()` 사용)
- 한 화면에 빨간 사과가 2개 보이면 가까운 사과부터 접근
- 구조 순서, 위치, 시각을 로그와 지도 이미지에 표시
- `Confirm.update()`는 YOLO를 실행한 step에서만 호출 (4 step마다). 추론하지 않은 step에 `seen=False`를 넘기면 확정되지 않음

**공통 안전 규칙**

- 남은 시간 < `RETURN_RESERVE` → 즉시 RETURN
- 4초 동안 5 cm 미만 이동 → RECOVERY (후진 후 넓은 쪽으로 회전, 재계획)
- 안전 필터가 3초 이상 전진 차단 → 재계획
- 프론티어 목표에 30초 안에 도달 못 함 → 블랙리스트 등록

---

## 8. 설계 결정

| 항목 | 결정 | 이유 |
|---|---|---|
| 코드 위치 | 팀 코드를 `controllers/sar_main/sar/`에 둠. 설치 없이 import | 컨트롤러 폴더 1개로 실행·제출. 평가 환경 절차 최소화 |
| 위치 추정 | 엔코더 오도메트리 + **방향 1차원 칼만 필터** (예측: 엔코더 dθ, 갱신: 나침반) | 강의 전제: 시작 위치를 알면 오도메트리로 추정 가능(시뮬). 위치 오차의 주원인이 방향 오차이므로 방향부터 필터링 |
| 스캔-지도 매칭 도입 기준 | `tb3_ground_truth` 월드에서 한 바퀴 주행 후 위치 오차 **0.3 m 이상**이면 도입, 미만이면 생략 | 잘못된 매칭은 지도를 손상시킴. 필요성을 수치로 확인 후 결정 |
| 스캔-지도 매칭 방식 (도입 시) | 1~2초마다 현재 스캔을 지도에 대해 작은 범위(±0.1 m, ±3°) 격자 탐색, 점수 향상이 충분할 때만 `correct()` 적용 | ICP보다 구현이 단순하고 실패 시 영향 범위가 작음 |
| 파티클 필터 (MCL/AMCL) | 사용 안 함 | 3시간 안에 구현·튜닝 불가 |
| 대상 위치 추정 | 거리 가중 평균 | 카메라 거리 추정은 가까울수록 정확 (3 m에서 상자 폭 1 px 오차 = 거리 6% 오차) |
| 나침반 보정 | 시작 회전 중 엔코더 방향과 비교해 부호·오프셋 자동 추정 | 나침반 값 규약이 예제마다 달라 수동 설정 시 오류 위험 |
| 대상 인식 | YOLO11n으로 후보 검출 → 상자 안 HSV 색 비율로 대상 색 판별 | YOLO 사용이 대회 방침. 연습 맵에 색만 다른 사과가 섞여 있어 YOLO 클래스만으로 구분 불가 |
| YOLO 클래스 | 47 apple, 49 orange, 32 sports ball을 후보로 사용 | 주황·보라·초록 사과가 apple이 아닌 다른 클래스로 잡힐 수 있음 |
| YOLO 보완 | YOLO 대상이 없으면 같은 프레임에 색 분할(원형도·면적 조건)로 대체 검출 | 먼 거리의 작은 사과, COCO 학습 분포와 다른 색의 사과 |
| YOLO 실행 주기 | 4스텝(약 0.26초)마다 1회, `verbose=False` | CPU 추론 시간만큼 시뮬 진행이 느려짐 |
| 오탐 방지 | 원형도, 최소 면적, 중심이 화면 가운데선보다 20 px 이상 위인 후보 제외, 4프레임 연속 확인 | 식탁 위 과일(21 m 이내에서 항상 20 px 이상 위), 주황빛 바닥 |
| 대상 거리 | 상자 긴 변으로 추정: `d = f · D / (w · cos β)` | 사과가 라이다에 보이지 않음. 화면 끝에서 잘린 상자는 한 변만 줄어듦 |
| 같은 색 대상 2개 | 구조 완료 위치 주변 검출 제외, 미구조 후보 저장 | 같은 사과 재구조 방지, 두 번째 사과 탐색 시간 단축 |
| 사람 회피 | 전역 경로(A*)는 지도 기준, 매 스텝 라이다 안전 필터가 최종 속도 결정 | 지도에 없는 움직이는 장애물 |
| 지도 | 베이지안 점유 격자 (log-odds) | 강의 Bayesian Filter를 로그 공간 덧셈으로 구현. 상·하한 고정으로 사람이 지나간 칸은 8회 갱신(매 스텝 갱신 시 약 0.5초) 뒤 빈칸 |
| 경로 | A* 8방향, 팽창 반경 = 로봇 반지름 + 0.07 m, 벽 근처 추가 비용 | 통로 가운데 주행 |
| 경로 추종 | pure pursuit, look-ahead 0.35 m, 55° 이상 틀어지면 제자리 회전 | 강의 방식과 동일 |
| 지도 시각화 | `MAP_SAVE_PERIOD`(5초) 주기와 RESCUE·DONE 시점에 PNG 저장. 격자, 궤적, 경로, 구조 위치, 시작점 표시 | 공지 권장 사항. 시연·README 증빙 |
| 라이다 minRange 미만 처리 | 별도 보정 없음. `STOP_DIST`(0.20 m)를 `minRange`(0.12 m)보다 0.08 m 크게 유지 | LDS-01은 0.12 m 미만 거리에서 `inf`를 반환해 "장애물 없음"과 구분 불가. `V_MAX`·64 ms 스텝 기준 스텝당 최대 이동 0.0115 m라 안전 필터가 매 스텝 실행되면 실제로 minRange 미만 구간에 도달하지 않음(전용 진단 월드 `worlds/lidar_range_check.wbt`로 실측 확인) |

---

## 9. 기본 설정값 (config.py)

sar-robot `controllers/sar_main/sar/config.py`의 현재 값입니다.

```python
import math

# TurtleBot3 Burger
WHEEL_RADIUS = 0.033          # m
WHEEL_SEPARATION = 0.160      # m
ROBOT_RADIUS = 0.105          # m
MAX_WHEEL_SPEED = 6.67        # rad/s

# 장치 이름
LIDAR_NAME = "LDS-01"
CAMERA_NAME = "camera"
COMPASS_NAME = "compass"
LEFT_MOTOR_NAME = "left wheel motor"
RIGHT_MOTOR_NAME = "right wheel motor"

# 과제
START_X, START_Y, START_THETA = -0.3, -7.5, math.pi   # 대회 당일 입력
TARGET_COLOR = "red"          # HSV_RANGES의 키
TARGET_COUNT = 2              # 대회 공지: 빨간 사과 2개
GOAL_XY = None                # 대회 공지: 목적지는 시작 지점
TIME_LIMIT = 900.0            # s, 시뮬 시간. 대회 공지 확인 후 수정
RETURN_RESERVE = 150.0        # s

# 주행
V_MAX = 0.18                  # m/s
V_APPROACH = 0.10             # m/s
W_MAX = 1.8                   # rad/s
LOOKAHEAD = 0.35              # m
GOAL_TOL = 0.15               # m
SAFETY_MARGIN = 0.06          # m, 몸체 바깥 여유
STOP_DIST = 0.20              # m, 정면 즉시 정지 거리

# 지도
MAP_RES = 0.05                # m
L_OCC = 0.9                   # 맞은 칸 log-odds 증분 (확률 약 0.71)
L_FREE = -0.4                 # 지나간 칸 log-odds 증분 (확률 약 0.40)
L_MIN, L_MAX = -2.0, 3.5      # 확률 약 0.12 ~ 0.97에서 고정
OCC_THRESHOLD = 0.3           # 이보다 크면 장애물 (확률 약 0.57)
LIDAR_MIN, LIDAR_MAX = 0.12, 3.5
INFLATE = ROBOT_RADIUS + 0.07
MIN_FRONTIER_CELLS = 6
MAP_SIZE = 32.0               # m, 시작점 중심 정사각형 지도 한 변
RAY_STEP = MAP_RES / 2        # m, 라이다 광선 추적 표본 간격
WALL_BAND = 0.20              # m, 팽창 영역 바깥 벽 근처 추가 비용 폭

# 경로 계획
WALL_COST = 2.0               # 벽 근처 칸 비용 배수 최대 증가량
UNKNOWN_COST = 1.5            # allow_unknown일 때 모르는 칸 비용 배수
PLAN_MARGIN = 1.0             # m, 계획 영역 = 확인 영역 + 여백
SNAP_RADIUS = 0.5             # m, 통과 불가 시작·목표의 대체 칸 탐색 반경
PATH_SMOOTH = True            # 시야선 경로 다듬기. False면 격자 경로
PATH_STEP = 0.10              # m, 반환 경로 점 간격
BLACKLIST_RADIUS = 0.5        # m, 블랙리스트 주변 프론티어 제외 반경
FRONTIER_MIN_DIST = 0.3       # m, 이보다 가까운 프론티어 칸 제외

# 미션 상태 머신
INIT_SPIN_W = 0.8             # rad/s, 시작 제자리 회전 속도
COMPASS_SCALE_TOL = 0.5       # 오도메트리/나침반 회전 비율이 1에서 이만큼 벗어나면 나침반 미사용
REPLAN_PERIOD = 2.0           # s, 프론티어 선택·경로 재계획 주기
FRONTIER_TIMEOUT = 30.0       # s, 같은 프론티어 목표 제한 시간
APPROACH_TIMEOUT = 60.0       # s, 대상 접근 제한 시간
RESCUE_HOLD = 2.0             # s, 구조 정지 시간
RETURN_TOL = 0.12             # m, 시작점 도착 판정
FACE_TOL = 0.1                # rad, 정면 정렬 허용 오차
FACE_GAIN = 1.5               # 1/s, 정렬 각속도 이득
CANDIDATE_DROP_DIST = 0.8     # m, 미확정 후보 삭제 거리
STUCK_TIME, STUCK_DIST = 4.0, 0.05   # s, m. 정체 판정
BLOCKED_TIME = 3.0            # s, 안전 필터 연속 차단 재계획
RECOVERY_BACK_TIME = 1.0      # s, 후진 시간
RECOVERY_TURN_TIME = 1.5      # s, 회전 시간
TRAJ_STEP = 0.10              # m, 궤적 기록 간격

# 위치 추정
HEADING_Q = 0.01 ** 2         # 방향 예측 잡음 (한 스텝, rad^2)
HEADING_R = 0.05 ** 2         # 나침반 관측 잡음 (rad^2)
SCAN_MATCH = False            # 오차 0.3 m 이상 확인 시 True
SCAN_MATCH_PERIOD = 1.5       # s

# 인식
CAMERA_FOV = 1.0472           # rad
YOLO_MODEL = "../../models/YOLO/yolo11n.pt"   # 컨트롤러 폴더 기준
YOLO_DEVICE = "cpu"           # NVIDIA GPU면 "cuda", Apple Silicon이면 "mps"
YOLO_CLASSES = [47, 49, 32]   # apple, orange, sports ball
YOLO_CONF = 0.20
YOLO_EVERY = 4                # 몇 스텝마다 추론
COLOR_RATIO_MIN = 0.25        # 상자 안 대상 색 픽셀 비율 하한
USE_COLOR_FALLBACK = True     # YOLO 미검출 시 색 분할로 대체
TARGET_DIAMETER = 0.095       # m
MIN_BLOB_AREA = 60            # px
MIN_CIRCULARITY = 0.6         # 색 분할 덩어리 원형도(4πA/P²) 하한
MASK_KERNEL_SIZE = 3          # px, 색 마스크 잡음 제거(열림 연산) 커널 크기
HORIZON_MARGIN = 20           # px, 중심이 화면 가운데선보다 이만큼 위인 검출은 식탁 위 물체로 제외
CONFIRM_FRAMES = 4
CONFIRM_MATCH_RADIUS = 0.5    # m, 연속 검출을 같은 대상으로 보는 위치 차이 상한
CONFIRM_MIN_DIST = 0.3        # m, 위치 가중 평균의 거리 하한
APPROACH_DIST = 0.35          # m
FOUND_EXCLUDE_RADIUS = 0.6    # m, 구조 완료 위치 주변 검출 제외 반경

# HSV 범위 (10장)
HSV_RANGES = {
    "red": [((0, 120, 60), (8, 255, 255)), ((170, 120, 60), (179, 255, 255))],
    "orange": [((12, 170, 120), (28, 255, 255))],
    "purple": [((125, 90, 50), (160, 255, 255))],
    "green": [((30, 90, 50), (60, 255, 255))],
}

# 출력
MAP_SAVE_DIR = "output"       # 컨트롤러 폴더 기준
MAP_SAVE_PERIOD = 5.0         # s
MAP_VIEW_MARGIN = 20          # cell, 확인된 영역 바깥 여백
MAP_VIEW_SCALE = 2            # 격자 1칸의 픽셀 수
LOG_INTERVAL = 1.0            # s, 주기 로그 간격

# HSV 튜닝 도구 (controllers/hsv_tuner)
TUNER_V = 0.10                # m/s
TUNER_W = 1.0                 # rad/s
TUNER_FRAME_DIR = "frames"    # 컨트롤러 폴더 기준
```

---

## 10. HSV 초기값 (OpenCV 기준: H 0~179, S·V 0~255)

YOLO 상자 안 색 판별과 색 분할 대체 검출에 공통으로 사용합니다(`config.HSV_RANGES`). 현장 조명에서 `hsv_tuner`로 반드시 다시 조정하십시오.

| 색 | 범위 | 참고 |
|---|---|---|
| red | (0,120,60)~(8,255,255) ∪ (170,120,60)~(179,255,255) | baseColor 1 0 0 |
| orange | (12,170,120)~(28,255,255) | baseColor 1 0.73 0. 나무 바닥과 겹칠 수 있어 채도 하한을 높게 |
| purple | (125,90,50)~(160,255,255) | baseColor 0.56 0 1 |
| green | (30,90,50)~(60,255,255) | 텍스처 사과. 현장 확인 필수 |

---

## 11. 코딩 규칙

1. Python 3.10, numpy·opencv·ultralytics만 사용 (팀 코드는 scipy 없이 동작)
   - YOLO 모델 로드는 컨트롤러 시작 시 1회. 매 프레임 로드 금지
   - `model.predict(..., verbose=False)`로 로그 출력 억제
2. Webots API는 `robot_io.py`에서만 호출. `controller` 모듈은 `RobotIO` 생성 시 import
3. 숫자 상수는 `config.py`에만 정의
4. A*·프론티어 선택 같은 무거운 계산은 필요할 때만 실행 (목표 변경, 경로 차단, 2초 주기)
5. 모든 `sar/` 모듈에 Webots 없이 실행되는 `if __name__ == "__main__":` 단독 테스트 포함. 실행: `controllers/sar_main`에서 `../../.venv/bin/python -m sar.<모듈>`
6. 공개 함수에는 `tests/test_<모듈>.py` pytest 1개 이상
7. 센서 값이나 경로가 None일 때의 동작을 반드시 정의 (예외로 종료 금지)
8. 상태 전환 로그 형식: `[t=12.3s] EXPLORE -> APPROACH (red at -5.34,-10.54)`
9. 모든 while 루프에 반복 상한 또는 시간 제한
10. `print`는 `sar_main.py`에서만 사용. `sar/` 모듈은 값을 반환 (단독 테스트 출력은 예외)
11. PR 전 저장소 루트에서 `ruff format .`, `ruff check .`, `pytest -q` 통과
12. 브랜치 `<type>/<내용>`, 커밋 제목 `type(scope): 명사형 요약`. 브랜치·커밋에 AI 도구 이름과 서명 금지
13. 기능 1개를 구현하면 같은 작업에서 docs 저장소 `human/explanation/features/<모듈 이름>.md`에 기능 문서 작성. docs PR을 코드 PR보다 먼저 병합
14. Webots 동작 검증은 `worlds/sar_apartment.wbt`(apartment.wbt에 `sar_main` 컨트롤러 지정)로만 수행. `sar_dev.wbt` 결과는 검증으로 인정하지 않음
15. 헤드리스로 실행한 Webots는 직접 실행한 프로세스만 PID로 종료. `pkill -f webots` 금지

---

## 12. 알려진 문제 (2026-09-30 갱신)

| 번호 | 위치 | 문제 | 상태·조치 |
|---|---|---|---|
| 1 | `local_control._lookahead_point` | 지나온 경로점을 목표로 선택 | 해결 (sar-robot #11) |
| 2 | `sar_main.py`, `mission` | 나침반 값 미전달 | 해결 (mission INIT_SPIN 보정) |
| 3 | `local_control.safety_filter` | 정면 검사 범위가 로봇 폭보다 좁음 | 해결 (sar-robot #11) |
| 4 | `local_control.safety_filter` | 후진 미검사 | 해결 (sar-robot #11) |
| 5 | `local_control.safety_filter` | 라이다 0.12 m 미만 반환값 | 확인 완료: `inf` 반환, `STOP_DIST` 여유로 안전 (sar-robot #9) |
| 6 | `config.TARGET_DIAMETER` | 0.095 m. RedApple 충돌 구 지름은 0.10 m | 메시 지름 또는 1 m 상자 폭 실측 후 보정 (인지) |
| 7 | `perception.detect_all` | YOLO 대상이 있으면 색 분할 미실행 | YOLO 상자와 겹치지 않는 색 분할 결과 추가 (인지) |
| 8 | `local_control` | `_PIVOT_ANGLE`(55°) 모듈 상수 | `config.py`로 이동 (행동) |
| 9 | `perception` YOLO | apartment 월드 빨간 사과를 0.5 m 거리에서도 YOLO가 검출하지 못함. 확정 검출 3건 모두 `source=color` | 클래스·신뢰도·입력 크기 확인 (인지) |
| 10 | `perception` 색 분할 | 소화기를 빨간 사과로 확정. 원형도 소화기 0.64~0.70, 사과 0.74~0.79로 `MIN_CIRCULARITY`(0.6)로 구분 불가 | 형태 조건 추가 또는 임계값 재측정 (인지). 해결 전에는 두 번째 사과 대신 오탐을 구조할 수 있음 |

상세 근거는 docs 저장소 `human/explanation/sar-robot-병합-분석.md` 6장과 `human/explanation/features/mission.md`입니다.

---

## 13. 대회 시작 후 확인할 것

- [ ] 대회 맵이 apartment.wbt와 같은지
- [ ] 시작 위치·방향 값 → `config.py`
- [ ] 제한 시간, 채점 기준 (시간 점수, 충돌 감점) → `TIME_LIMIT`
- [ ] 평가 기준 괄호 "(코드로 구현, 글로 구현)"의 의미
- [ ] 제출 형태(저장소 전체 또는 컨트롤러 폴더), 평가 PC 사양(OS, GPU, 인터넷)
- [ ] 대회 월드 로봇에 `compass`, `gyro`가 있는지 (`getDevice` 결과 None 여부). 나침반이 없으면 칼만 필터 갱신 단계 생략, 자이로가 있으면 예측을 자이로로
- [x] (행동, 0:40까지) `tb3_ground_truth` 월드에서 오도메트리 위치 오차 측정 → `SCAN_MATCH` 결정. 결과: 위치 오차 0.139 m (< 0.3 m) → `SCAN_MATCH = False` 유지. 짧은 사각형 경로라 방향 오차(~0.59 rad)는 컸음, 참고용
- [ ] 스캔 매칭, 사전 작성 코드 허용 범위
- [ ] YOLO 방침 세부: 사전학습 가중치 그대로 사용인지, 추가 학습 허용인지
- [ ] (인지, 첫 20분) `tb3_teleop_yolo`로 색깔별 사과가 어떤 클래스·신뢰도로 잡히는지 기록 (특히 보라·초록·주황 사과, 1 m·2 m·3 m 거리)
- [ ] (인지) 추론 1회 시간 측정. 느리면 `YOLO_EVERY` 증가 또는 `imgsz=480`
- [ ] (인지) 사과 지름 실측 → `TARGET_DIAMETER`
- [x] (행동) 라이다 0.12 m 미만 반환값 확인. 결과: `inf` 반환 (장애물 없음과 구분 불가). `STOP_DIST`(0.20 m)가 `minRange`(0.12 m)보다 0.08 m 여유가 있고 `V_MAX` 기준 스텝당 최대 이동거리 약 0.0115 m라 안전 필터가 매 스텝 동작하는 한 실제로 이 구간에 도달하지 않음. 8장 설계 결정에 근거 기록

확인 완료: 대상 색·개수(빨간 사과 2개), 목적지(시작 지점), 연습 월드 로봇의 `compass`·`gyro`·`accelerometer` 존재(PROTO 기본 포함), 오도메트리 위치 오차, 라이다 minRange 미만 반환값
