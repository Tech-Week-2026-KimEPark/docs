# grid_map 기능 설명

이 문서는 sar-robot `controllers/sar_main/sar/grid_map.py`에 구현한 점유 격자 지도, 장애물 팽창, 프론티어 검출의 동작과 검증 결과를 설명합니다.

## 과제와의 관계

- 지도는 사전에 제공되지 않습니다. 라이다로 지도를 작성해 탐색·접근·복귀 경로 계획의 입력으로 사용합니다.
- 상태 머신의 모든 단계에서 매 step `update()`를 호출합니다. INIT_SPIN 단계의 360° 회전으로 주변 지도를 먼저 작성합니다.
- EXPLORE 단계의 목표는 `frontiers()` 결과에서 선택합니다. 프론티어가 없으면 RETURN으로 전환합니다.
- `public()` 결과를 `viz.render_map()`에 전달해 지도 이미지를 저장합니다.

## 동작 원리

### 격자와 좌표

격자는 `grid[row][col]`이고 row는 y, col은 x에 대응합니다. 해상도는 `MAP_RES`(0.05 m), 크기는 시작점 중심 `MAP_SIZE`(32 m) 정사각형으로 640 × 640칸입니다. 격자 (0, 0) 칸의 모서리 좌표를 $(x_0, y_0)$, 해상도를 $r$이라고 하면 칸 번호는 다음과 같습니다.

$$
\text{row} = \left\lfloor \frac{y - y_0}{r} \right\rfloor, \qquad
\text{col} = \left\lfloor \frac{x - x_0}{r} \right\rfloor
$$

`to_world()`는 칸 중심 좌표를 반환합니다.

### log-odds 갱신

각 칸은 log-odds $l$을 저장합니다. 라이다 빔이 지나간 칸에 $L_{\text{free}}$, 빔이 맞은 칸에 $L_{\text{occ}}$를 더하고 $[L_{\min}, L_{\max}]$로 제한합니다.

$$
l \leftarrow \operatorname{clip}\left(l + L_{\text{occ/free}},\ L_{\min},\ L_{\max}\right)
$$

- 인덱스 $i$ 빔의 월드 각도는 $\theta + \pi - 2\pi i / N$입니다. $N$은 빔 개수입니다.
- 모든 빔을 `RAY_STEP`(0.025 m) 간격으로 한 번에 표본화합니다. numpy 배열 연산만 사용하고 Python 반복문을 사용하지 않습니다.
- 스캔 1회에서 같은 칸은 한 번만 갱신합니다. 맞은 칸은 빈칸 갱신에서 제외합니다.
- 반사 없음(`inf`)과 `LIDAR_MAX` 초과 빔은 `LIDAR_MAX`까지 빈칸으로만 갱신합니다. `LIDAR_MIN` 미만과 `NaN`은 무시합니다.
- 로봇 반지름 안의 칸은 빈칸으로 갱신합니다. 시작 칸이 모름으로 남아 경로 계획이 실패하는 경우를 방지합니다.
- 한 번이라도 갱신한 칸은 `seen`에 기록합니다. 모르는 칸 판정은 log-odds 값 대신 `seen`을 사용합니다.

$L_{\max} = 3.5$, 장애물 기준 $l_{\text{th}} = 0.3$, $L_{\text{free}} = -0.4$이므로 사람이 지나간 칸은 8회 갱신 후 빈칸이 됩니다.

### 장애물 팽창과 벽 근처 영역

`cv2.distanceTransform`으로 각 칸에서 가장 가까운 장애물 칸까지의 거리를 계산합니다. scipy를 사용하지 않습니다.

| 레이어 | 조건 | 용도 |
|---|---|---|
| `occ` | $l > l_{\text{th}}$ | 장애물 |
| `blocked` | 장애물 거리 < `INFLATE` (0.175 m) | 로봇 중심 진입 불가 |
| `soft` | `INFLATE` ≤ 장애물 거리 < `INFLATE + WALL_BAND` | 벽 근처 추가 비용 |
| `unknown` | `seen`이 거짓 | 모르는 칸 |

계산 결과는 `update()` 호출 전까지 캐시합니다.

### 프론티어

프론티어 칸은 모르는 칸과 4방향으로 맞닿은 아는 빈칸입니다. `cv2.connectedComponents`로 8방향 연결 묶음을 만들고 `MIN_FRONTIER_CELLS`(6칸) 미만 묶음은 센서 잡음으로 제외합니다. 결과는 칸 수 내림차순입니다.

## 인터페이스와 설정값

| 함수 | 입력 | 출력 |
|---|---|---|
| `GridMap(center_x, center_y, size, res)` | 지도 중심 [m], 한 변 [m], 해상도 [m] | 지도 객체 |
| `update(pose, ranges)` | `(x, y, theta)`, 라이다 거리 목록 | 없음. `None`이면 무시 |
| `to_cell(x, y)` / `to_world(row, col)` | 월드 좌표 / 칸 | 칸 / 칸 중심 좌표 |
| `layers()` | 없음 | `(occ, blocked, soft, unknown)` bool 배열 |
| `frontiers()` | 없음 | `[(칸 수, [(row, col), ...]), ...]` |
| `public()` (추가) | 없음 | 공개값 격자 int8 (-1 모름, 0 빈칸, 1 장애물) |
| `clearance()` (추가) | 없음 | 가장 가까운 장애물까지 거리 [m] 배열 |
| `simulate_scan(walls, to_cell, pose)` | 벽 bool 격자, 변환 함수, pose | 합성 라이다 거리 목록 (테스트용) |

| 설정값 | 값 | 의미 |
|---|---|---|
| `MAP_RES` | 0.05 m | 격자 해상도 |
| `MAP_SIZE` | 32.0 m | 지도 한 변 (추가) |
| `RAY_STEP` | 0.025 m | 광선 표본 간격 (추가) |
| `L_OCC`, `L_FREE` | 0.9, −0.4 | log-odds 증분 |
| `L_MIN`, `L_MAX` | −2.0, 3.5 | log-odds 범위 |
| `OCC_THRESHOLD` | 0.3 | 장애물 판정 기준 |
| `INFLATE` | 0.175 m | 팽창 반경 |
| `WALL_BAND` | 0.20 m | 벽 근처 영역 폭 (추가) |
| `MIN_FRONTIER_CELLS` | 6 | 프론티어 묶음 최소 칸 수 |

## 검증 결과

2026-09-30, macOS, Python 3.10.21, sar-robot `feat/grid-map-planner` 브랜치에서 확인했습니다.

| 항목 | 명령·조건 | 결과 |
|---|---|---|
| 단독 테스트 | `controllers/sar_main`에서 `../../.venv/bin/python -m sar.grid_map` | 통과 |
| pytest | `.venv/bin/python -m pytest -q tests/test_grid_map.py` | 11개 통과 |
| 라이다 방향 | 인덱스 90 빔 1 m 반사, theta 0 | (0, 1) 칸 장애물 |
| 사람 흔적 제거 | 1 m 거리 20회 반사 후 반사 없음 8회 | 빈칸으로 전환 |
| 갱신 시간 | 640 × 640 지도, 빔 360개, 50회 평균 | 2.3 ms |
| 레이어·프론티어 시간 | 같은 지도 | 8 ms |
| 탐색 통합 확인 | 12.4 × 13.1 m 합성 지도, 방 9개, 장애물 25개, 순간 이동 주행 | 확인 영역 99.9% |
| Webots 주행 | `sar_dev.wbt`, `apartment.wbt` | 미확인. `mission.py` 연결 후 확인 필요 |

## 한계와 확인 필요 항목

- 라이다 장착 위치를 로봇 중심으로 가정합니다. LDS-01의 실제 오프셋은 미확인입니다.
- 사과, 캔, 맥주병은 라이다 높이보다 낮아 지도에 표시되지 않습니다. 충돌 방지는 `local_control.safety_filter()`가 담당합니다.
- 유리창처럼 반사가 없는 벽은 빈칸으로 갱신될 수 있습니다. 대회 맵에서 확인이 필요합니다.
- 오도메트리 오차가 누적되면 지도의 벽이 두껍게 번집니다. `SCAN_MATCH` 도입 여부는 행동 담당의 오차 측정 결과로 결정합니다.

## 관련 자료

- 구현 PR: 작성 후 추가
- 원본 인터페이스: [과제와 구현 기준](../../reference/sar-과제-구현-기준.md) 6장, 7.2절
- 경로 계획: [planner 기능 설명](planner.md)
