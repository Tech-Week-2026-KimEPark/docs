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

`cv2.distanceTransform`의 정확한 유클리드 거리(`DIST_MASK_PRECISE`)로 각 칸에서 가장 가까운 장애물 칸까지의 거리를 계산합니다. 근사 마스크는 거리를 최대 2% 크게 계산해 팽창 영역이 부족해지므로 사용하지 않습니다. scipy를 사용하지 않습니다.

| 레이어 | 조건 | 용도 |
|---|---|---|
| `occ` | $l > l_{\text{th}}$ | 장애물 |
| `blocked` | 장애물 거리 < `INFLATE` (0.175 m) | 로봇 중심 진입 불가 |
| `soft` | `INFLATE` ≤ 장애물 거리 < `INFLATE + WALL_BAND` | 벽 근처 추가 비용 |
| `unknown` | `seen`이 거짓 | 모르는 칸 |

### 버전과 캐시

| 값 | 증가 조건 | 용도 |
|---|---|---|
| `version` | `update()`로 값이 바뀔 때마다 | 지도 갱신 횟수 |
| `plan_version` | 갱신한 칸의 장애물 여부가 바뀌거나 새로 확인한 칸이 있을 때만 | 계획 결과의 유효 상태 식별 |

장애물 거리, 레이어, 프론티어, 계획 영역은 `plan_version`이 같은 동안 캐시를 재사용합니다. log-odds 값만 바뀐 갱신은 분류가 같으므로 다시 계산하지 않습니다. 장애물 거리는 `update()`마다 계산하지 않고 `layers()`나 계획 함수가 요청할 때 계산합니다.

`logodds`나 `seen` 배열을 직접 수정하면 변경을 감지할 수 없습니다. 수정 후 `invalidate()`를 호출하십시오.

### 프론티어

프론티어 칸은 모르는 칸과 4방향으로 맞닿은 아는 빈칸입니다. `cv2.connectedComponents`로 8방향 연결 묶음을 만들고 `MIN_FRONTIER_CELLS`(6칸) 미만 묶음은 센서 잡음으로 제외합니다. 결과는 칸 수 내림차순입니다.

## 인터페이스와 설정값

| 함수 | 입력 | 출력 |
|---|---|---|
| `GridMap(center_x, center_y, size, res)` | 지도 중심 [m], 한 변 [m], 해상도 [m] | 지도 객체 |
| `update(pose, ranges)` | `(x, y, theta)`, 라이다 거리 목록 | 없음. `None`이나 유한하지 않은 pose는 무시 |
| `to_cell(x, y)` / `to_world(row, col)` | 월드 좌표 / 칸 | 칸 / 칸 중심 좌표 |
| `layers()` | 없음 | `(occ, blocked, soft, unknown)` bool 배열 |
| `frontiers()` | 없음 | `[(칸 수, [(row, col), ...]), ...]` |
| `public()` (추가) | 없음 | 공개값 격자 int8 (-1 모름, 0 빈칸, 1 장애물) |
| `clearance()` (추가) | 없음 | 가장 가까운 장애물까지 거리 [m] 배열 |
| `frontier_sizes()` (추가) | 없음 | 칸별 프론티어 묶음 칸 수 int32 배열. 프론티어가 아니면 0 |
| `version`, `plan_version` (추가) | 없음 | 갱신 횟수, 계획용 분류 변경 횟수 |
| `invalidate()` (추가) | 없음 | 없음. 배열을 직접 수정한 뒤 캐시 무효화 |
| `simulate_scan(walls, to_cell, pose)` | 벽 bool 격자, `GridMap.to_cell`, pose | 합성 라이다 거리 목록 (테스트용, numpy 계산) |

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

2026-09-30, macOS(Apple Silicon), Python 3.10.21, sar-robot `perf/planning-optimization` 브랜치에서 확인했습니다.

| 항목 | 명령·조건 | 결과 |
|---|---|---|
| 단독 테스트 | `controllers/sar_main`에서 `../../.venv/bin/python -m sar.grid_map` | 통과 |
| pytest | `.venv/bin/python -m pytest -q tests/test_grid_map.py` | 19개 통과 |
| 라이다 방향 | 인덱스 90 빔 1 m 반사, theta 0 | (0, 1) 칸 장애물 |
| 사람 흔적 제거 | 1 m 거리 20회 반사 후 반사 없음 8회 | 빈칸으로 전환 |
| 팽창 영역 | 무작위 지도 3개에서 모든 장애물 칸까지의 거리를 직접 계산한 결과와 비교 | 일치 |
| 분류 유지 갱신 | 같은 스캔 2회 | `version` 증가, `plan_version` 유지, 레이어 캐시 유지 |
| 분류 변경 갱신 | 새 장애물 반사, 새 칸 확인 | `plan_version` 증가, 캐시 무효화 |
| Webots 주행 | `sar_apartment.wbt`, 1회 | 지도 작성 확인. 가구는 다리만 점으로 표시되고 오도메트리 방향 오차로 지도가 기울어짐. 상세는 [planner 기능 설명](planner.md)의 "Webots 실행 결과" |

성능은 `.venv/bin/python scripts/bench_planner.py`로 측정했습니다. 640 × 640 지도, 빔 360개, 워밍업 후 60회, 단위는 ms입니다. 수정 전 코드는 sar-robot `main` `897fee8`입니다.

| 항목 | 수정 전 중앙값 / p95 / 최대 | 수정 후 중앙값 / p95 / 최대 |
|---|---|---|
| `update()` (분류 변화 없는 스캔) | 1.90 / 2.07 / 2.32 | 1.48 / 1.55 / 1.66 |
| `update()` + `layers()` (분류 변화 없음) | 5.27 / 5.33 / 5.34 | 1.48 / 1.60 / 1.76 |

분류가 바뀌는 갱신(탐색 중 대부분의 스텝) 뒤에는 장애물 거리와 레이어를 다시 계산합니다. 이 경우의 비용은 계획 함수 시간에 포함되며 [planner 기능 설명](planner.md)의 "지도 변경 직후" 항목에 있습니다.

이전 측정(sar-robot #12 시점, 무작위 거리 스캔 50회 평균)은 갱신 2.3 ms, 레이어·프론티어 8 ms였습니다. 조건이 달라 위 표와 직접 비교할 수 없습니다.

## 한계와 확인 필요 항목

- 라이다 장착 위치를 로봇 중심으로 가정합니다. LDS-01의 실제 오프셋은 미확인입니다.
- 의자처럼 라이다 높이에서 다리만 보이는 가구는 몸체가 지도에 표시되지 않습니다. Webots 1회 실행에서 확인했습니다.
- 사과, 캔, 맥주병은 라이다 높이보다 낮아 지도에 표시되지 않습니다. 충돌 방지는 `local_control.safety_filter()`가 담당합니다.
- 유리창처럼 반사가 없는 벽은 빈칸으로 갱신될 수 있습니다. 대회 맵에서 확인이 필요합니다.
- LDS-01은 0.12 m 미만 거리에서 `inf`를 반환합니다(`CONTEXT.md` 8장). 이 경우 해당 빔은 `LIDAR_MAX`까지 빈칸으로 갱신됩니다. 안전 필터가 `STOP_DIST`(0.20 m)를 유지하는 동안에는 발생하지 않습니다.
- 오도메트리 오차가 누적되면 지도의 벽이 두껍게 번집니다. `SCAN_MATCH` 도입 여부는 행동 담당의 오차 측정 결과로 결정합니다.

## 관련 자료

- 구현 PR: sar-robot [#12](https://github.com/Tech-Week-2026-KimEPark/sar-robot/pull/12) (최초 구현). 버전·캐시 보완 PR은 작성 후 추가
- 원본 인터페이스: [과제와 구현 기준](../../reference/sar-과제-구현-기준.md) 6장, 7.2절
- 경로 계획: [planner 기능 설명](planner.md)
