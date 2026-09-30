# perception 기능 설명

이 문서는 sar-robot `controllers/sar_main/sar/perception.py`에 구현한 빨간 사과 검출과 위치 추정의 동작과 검증 결과를 설명합니다.

## 과제와의 관계

- 빨간 사과 2개를 찾는 과제 요구사항의 인식 부분을 담당함
- 상태 머신의 EXPLORE 단계에서 대상 확인, APPROACH 단계에서 대상 위치 갱신에 사용함
- 같은 사과의 재구조를 막는 판정 함수 `is_excluded()`를 제공함. 호출은 `mission` 담당
- 연습 맵의 방해 요소인 다른 색 사과, 식탁 위 과일 모델, 공을 제외함

## 동작 원리

`TargetDetector.detect_all()`은 한 프레임을 다음 순서로 처리합니다.

| 단계 | 처리 | 설정값 |
|---|---|---|
| 1 | YOLO11n으로 apple, orange, sports ball 후보 상자 검출. 클래스 무관 NMS 적용 | `YOLO_CLASSES`, `YOLO_CONF` |
| 2 | 상자 안 대상 색 픽셀 비율이 하한 이상인 후보만 선택 | `HSV_RANGES`, `COLOR_RATIO_MIN` |
| 3 | 색 마스크에서 사과 모양 덩어리 중 2단계 상자 밖에 있는 것을 추가 | `USE_COLOR_FALLBACK`, `MIN_BLOB_AREA`, `MIN_CIRCULARITY`, `MIN_BLOB_FILL`, `BLOB_OPEN_RATIO` |
| 4 | 중심이 화면 가운데선보다 기준 이상 위인 후보를 식탁 위 물체로 제외 | `HORIZON_MARGIN` |
| 5 | 거리와 방위각 계산 후 가까운 순서로 정렬 | `TARGET_DIAMETER`, `CAMERA_FOV` |

YOLO는 사과를 색과 관계없이 같은 클래스로 검출합니다. 따라서 대상 색 판별은 2단계의 HSV 색 비율로 수행합니다.

YOLO의 기본 NMS는 클래스별로 겹친 상자를 제거합니다. Webots 프레임에서 사과 1개가 같은 위치의 sports ball 상자와 apple 상자로 함께 검출되어 대상이 2개로 계산되었습니다. 클래스 무관 NMS(`agnostic_nms=True`)를 적용해 겹친 상자 중 신뢰도가 가장 높은 1개만 남깁니다.

3단계 색 분할은 YOLO가 놓친 먼 거리의 작은 사과와 한 화면에 보이는 두 번째 사과를 보완합니다. YOLO 대상 상자 안에 중심이 있는 덩어리는 같은 사과이므로 추가하지 않습니다. 사과 모양 판정은 다음 순서입니다.

1. 면적이 `MIN_BLOB_AREA` 미만이거나 화면 가장자리에 닿은 덩어리 제외. 잘린 덩어리는 모양과 크기를 판단할 수 없음
2. 덩어리 짧은 변의 `BLOB_OPEN_RATIO`배 크기 원형 커널로 열림 연산해 꼭지 같은 가는 돌출부 제거
3. 본체의 원형도 $4\pi A / P^2$가 `MIN_CIRCULARITY` 이상이고 외접원 채움 비율 $A / \pi r^2$가 `MIN_BLOB_FILL` 이상이면 사과로 판정. 거리는 본체 외접원 지름으로 계산

Webots 프레임에서 화면 왼쪽 끝에 잘린 빨간 소화기 아랫부분이 원형도 0.77로 이전 기준 0.6을 통과해 1.66 m 거리의 사과로 오검출되었습니다. 소화기는 흰 라벨로 빨간 영역이 여러 조각으로 나뉘며, 사각형 조각은 원형도가 약 0.785로 높지만 채움 비율은 0.64입니다. 반면 가까운 사과는 꼭지 때문에 원형도 0.73, 채움 비율 0.69까지 낮아졌습니다. 꼭지를 제거하면 사과는 원형도 0.89 이상, 채움 비율 0.85 이상이 되어 두 분포가 분리됩니다.

이전에는 YOLO 대상이 1개라도 있으면 색 분할을 실행하지 않아 두 번째 사과가 누락되었습니다([sar-robot 병합 내용 분석](../sar-robot-병합-분석.md) 6.1절 8번).

초점거리 $f$, 방위각 $\beta$, 거리 $d$는 다음 식으로 계산합니다. $W$는 이미지 폭, $\phi_h$는 수평 화각, $c_x$는 대상 중심 가로 좌표, $D$는 사과 지름, $w$는 상자 폭과 높이 중 큰 값입니다.

$$
f = \frac{W / 2}{\tan(\phi_h / 2)}, \qquad
\beta = -\arctan\frac{c_x - W/2}{f}, \qquad
d = \frac{f D}{w \cos\beta}
$$

$f D / w$는 광축 방향 깊이입니다. 화면 가장자리의 대상은 직선거리와 차이가 커서 $\cos\beta$로 나눕니다. 화면 끝에서 잘린 상자는 한 변만 줄어들므로 긴 변을 지름으로 사용합니다.

대상 월드 좌표는 로봇 pose $(x, y, \theta)$로 계산합니다.

$$
(x_t,\ y_t) = \bigl(x + d\cos(\theta + \beta),\ y + d\sin(\theta + \beta)\bigr)
$$

`Confirm`은 연속 검출을 확인하고 위치를 확정합니다. 관측 $k$의 위치를 $\mathbf{p}_k$, 거리를 $d_k$라고 하면 가까이서 본 관측에 큰 가중치를 줍니다.

$$
w_k = \frac{1}{\max(d_k,\ d_{\min})^2}, \qquad
\hat{\mathbf{p}} = \frac{\sum_k w_k \mathbf{p}_k}{\sum_k w_k}
$$

- 미검출 1회 또는 추정 위치에서 `CONFIRM_MATCH_RADIUS`를 벗어난 관측이 들어오면 기록을 초기화함
- `CONFIRM_FRAMES`회 연속 관측되면 확정함

YOLO 로드나 추론이 실패해도 예외로 종료하지 않습니다. 실패 사유를 `load_error` 또는 `last_error`에 기록하고 색 분할로 계속 동작합니다.

### 이동하는 사람 추적

`PersonTracker`는 라이다로 움직이는 사람(다리)을 추적합니다. 설계는 사람 회피 설계(docs [#21](https://github.com/Tech-Week-2026-KimEPark/docs/pull/21)) 3.1~3.2절이며, 미션 연결은 설계 합의 후 통합 담당이 진행합니다.

| 단계 | 처리 | 설정값 |
|---|---|---|
| 동적 점 | 라이다 끝점 중 지도상 확인된 빈칸(log-odds < 0)이면서 장애물 칸에서 일정 거리 이상 떨어진 점 | `DYN_WALL_GAP` 0.15 m |
| 덩어리 | 인접 빔 점 사이 거리가 기준 미만이면 같은 덩어리. 폭 조건을 만족하는 덩어리 중심을 후보로 사용 | `DYN_CLUSTER_GAP` 0.10 m, `PERSON_MIN_WIDTH`·`PERSON_MAX_WIDTH` 0.05·0.60 m |
| 다리 병합 | 중심 거리가 기준 이내인 후보를 한 사람으로 합침 | `LEG_PAIR_DIST` 0.40 m |
| 연결 | 예측 위치와 가장 가까운 후보를 기준 거리 이내에서 연결 | `TRACK_GATE` 0.50 m |
| 추정 | 알파-베타 필터로 위치·속도 갱신 | `TRACK_ALPHA`·`TRACK_BETA` 0.5·0.1 |
| 확정·삭제 | 연속 연결 횟수 이상이면 확정, 일정 시간 미관측이면 삭제 | `TRACK_CONFIRM` 3회, `TRACK_TIMEOUT` 1.0 s |

`update(t, pose, ranges, grid)`는 `grid.update()` 호출 전에 실행해야 합니다. 갱신 후에는 사람이 찍힌 칸이 장애물로 바뀌어 동적 점이 사라집니다. 반환 항목은 `{"id", "x", "y", "vx", "vy", "age", "moving"}`이며 `moving`은 속도가 `MOVING_SPEED`(0.05 m/s) 이상인지 여부입니다.

## 인터페이스와 설정값

| 함수 | 입력 | 출력 |
|---|---|---|
| `TargetDetector(color, model_path, device, ranges)` | 대상 색, YOLO 가중치 경로 또는 `None` | 검출기. YOLO는 생성 시 1회 로드 |
| `TargetDetector.detect(bgr)` | BGR 이미지 또는 `None` | 가장 가까운 대상 `dict` 또는 `None` |
| `TargetDetector.detect_all(bgr)` | BGR 이미지 또는 `None` | 대상 목록. 가까운 순서 |
| `TargetDetector.to_world(det, pose)` | 검출 결과, `(x, y, theta)` | 대상 월드 좌표 `(x, y)` |
| `Confirm.update(seen, xy, dist)` | 검출 여부, 대상 월드 좌표, 거리 [m] | `(확정 여부, 위치 추정 또는 None)` |
| `is_excluded(xy, found, radius)` | 대상 월드 좌표, 구조 완료 위치 목록 | 반경 이내 여부 |

검출 결과의 키 정의는 sar-robot [모듈 인터페이스](https://github.com/Tech-Week-2026-KimEPark/sar-robot/blob/main/docs/human/reference/interfaces.md)에 있습니다. 측정·튜닝용으로 마지막 `detect_all()` 호출의 YOLO 원본 상자(`last_yolo`)와 추론 시간(`last_yolo_ms`)을 보관합니다. `last_yolo`에는 색 판별에서 탈락한 상자와 상자 안 색 비율(`ratio`)이 포함됩니다. 마지막 검출 결과 목록(`last_detections`)과 누적 호출 횟수(`detect_count`)도 보관합니다. `hsv_tuner` 자동 탐색은 미션이 실행한 검출 결과를 이 두 값으로 재사용합니다.

| 설정값 | 값 | 의미 |
|---|---|---|
| `YOLO_CLASSES` | `[47, 49, 32]` | COCO apple, orange, sports ball |
| `YOLO_CONF` | 0.20 | YOLO 신뢰도 하한 |
| `YOLO_EVERY` | 4 | YOLO 추론 주기 [step]. 미션에서 적용 |
| `COLOR_RATIO_MIN` | 0.25 | YOLO 상자 안 대상 색 비율 하한 |
| `USE_COLOR_FALLBACK` | `True` | YOLO 대상이 없을 때 색 분할 사용 여부 |
| `MIN_BLOB_AREA` | 60 | 색 분할 덩어리 면적 하한 [px] |
| `MIN_CIRCULARITY` | 0.75 | 색 분할 덩어리 원형도 $4\pi A / P^2$ 하한. 꼭지 제거 후 판정 |
| `MIN_BLOB_FILL` | 0.72 | 색 분할 덩어리 면적 / 외접원 면적 하한. 정사각형 0.64, 원 1.0 |
| `BLOB_OPEN_RATIO` | 0.15 | 꼭지 제거 열림 연산 커널 크기 / 덩어리 짧은 변 |
| `HORIZON_MARGIN` | 20 | 높이 제외 기준 [px] |
| `TARGET_DIAMETER` | 0.095 | 사과 지름 [m] |
| `CONFIRM_FRAMES` | 4 | 확정에 필요한 연속 검출 수 |
| `CONFIRM_MATCH_RADIUS` | 0.5 | 같은 대상으로 보는 위치 차이 상한 [m] |
| `CONFIRM_MIN_DIST` | 0.3 | 가중 평균의 거리 하한 $d_{\min}$ [m] |
| `FOUND_EXCLUDE_RADIUS` | 0.6 | 구조 완료 위치 주변 제외 반경 [m] |

## 검증 결과

2026-09-30 sar-robot `feat/tuner-hold-to-drive` 브랜치에서 확인했습니다. Python은 3.10.11, ultralytics는 8.4.166입니다.

| 항목 | 명령·조건 | 결과 |
|---|---|---|
| 단위 테스트 | `python -m pytest -q tests/test_perception.py` | 31개 통과 |
| 빨간 방해 물체 회귀 | 테스트: 화면 끝에 잘린 원, 정사각형, 꼭지 달린 사과 | 앞의 둘은 제외, 사과는 꼭지를 뺀 지름으로 검출. 수정 전 코드에서는 3개 모두 실패 |
| 두 번째 사과 보완 | 테스트: YOLO가 가까운 사과만 검출, 먼 사과는 색 분할 대상 | 두 사과 모두 반환 (`yolo`, `color` 순서). 수정 전 코드에서는 실패 |
| 중복 방지 | 테스트: YOLO 상자 안의 같은 사과 | 색 분할로 다시 추가하지 않음 |
| 단독 실행 | `controllers/sar_main`에서 `python -m sar.perception` | 합성 이미지 빨간 원(반지름 12 px) 색 분할 검출, 거리 2.27 m |
| YOLO 경로 | 실제 `yolo11n.pt`, 회색 배경 빨간 원(반지름 30 px) 합성 이미지 | sports ball(32) 신뢰도 0.41로 검출, 색 판별 통과, 거리 0.90 m |
| 거리식 | 위 검출의 상자 폭 59.1 px, 방위각 −0.145 rad | 식 계산값 0.90 m와 일치 |
| YOLO 추론 시간 | 같은 이미지 20회, CPU | 중앙값 54.7 ms, 최소 51.1 ms, 최대 82.9 ms |
| 오판정 방지 | 테스트: 초록·주황 원, 작은 덩어리, 긴 사각형, 가운데선 위 원 | 모두 검출 제외 |
| 실패 처리 | 테스트: 모델 로드 실패, 추론 예외, 이미지 `None` | 예외 없이 색 분할 또는 빈 목록 |
| Webots 실제 검출 | apartment.wbt, `hsv_tuner` 자동 접근 중 저장 프레임. 벽 모서리 앞 빨간 사과 | sports ball 0.39로 검출, 색 비율 0.71, 추정 거리 1.99 m |
| 클래스 무관 NMS | 위 프레임에 `agnostic_nms` 끄고 켜서 YOLO 실행 | 끔: sports ball 0.39, apple 0.23 두 상자(같은 좌표). 켬: sports ball 0.39 한 상자 |
| Webots 거리·위치 오차 | 같은 측정의 추정 로봇 위치 | 미확인. 엔코더 위치 추정 오차가 커서 실제 거리 계산 불가 |
| Webots 다른 색 사과 | apartment.wbt `hsv_tuner` 회전 측정 | 보라 사과: sports ball 0.90, 빨강 비율 0.00. 초록 사과: sports ball 0.62, 0.00. 주황 사과: orange 0.35, 0.00. 모두 대상에서 제외. 옆의 주황색 고양이는 후보 없음 |
| Webots 빨간 사과 클래스 | 측정 53회의 YOLO 원본 상자 9개, 모두 빨간 사과 | sports ball 6개(신뢰도 0.25~0.83), apple 3개(0.20~0.45). 빨강 비율 0.68~0.78 |
| 빨간 방해 물체 | 저장 프레임 76장에 수정한 `find_color_blobs()` 적용 | 소화기(화면 끝, 이전 코드 오검출)와 진입 금지 표지판 검출 없음. 빨간 사과 9장면은 모두 검출 유지 |
| 작은 사과 | 합성 빨간 원 반지름 5~60 px | 반지름 5 px(약 5.3 m)부터 검출. 4 px는 `MIN_BLOB_AREA` 미만 |

사람 추적은 2026-09-30 sar-robot `feat/person-tracker` 브랜치에서 확인했습니다.

| 항목 | 명령·조건 | 결과 |
|---|---|---|
| 단위 테스트 | `python -m pytest -q tests/test_person_tracker.py` | 2개 통과 |
| 보행자 추적 | 확인된 빈칸 지도, 로봇 정지, 다리 원 2개(반지름 0.06 m, 간격 0.2 m)가 2 m 앞에서 +y로 0.2 m/s 이동, 3초 | 1명 확정, 위치 오차 0.1 m 이내, 속도 (0, 0.2) ± 0.05 m/s |
| 벽 제외 | 장애물 칸에 맞은 라이다 점 | 동적 점 없음, 추적 대상 없음 |
| Webots 보행자 | apartment.wbt 보행자 | 미확인 |

## 한계와 확인 필요 항목

- Webots 렌더링 이미지의 색별 YOLO 클래스·신뢰도, HSV 범위, 거리 오차는 빨간 사과 1장면 외에 미확인임. 측정 절차는 sar-robot [HSV 임계값 튜닝과 인식 성능 측정](https://github.com/Tech-Week-2026-KimEPark/sar-robot/blob/main/docs/human/how-to/hsv-tuning.md)에 있음
- 사과 지름 설정값과 PROTO 치수 불일치 가능성, 미검출 1회에 연속 확인 초기화는 [sar-robot 병합 내용 분석](../sar-robot-병합-분석.md) 6장 7·5번 항목임. 8번(색 분할 미실행)은 해결함
- 색 분할로 추가한 덩어리는 YOLO 확인 없이 대상이 됨. 사과와 크기·모양이 비슷한 빨간 원형 물체는 오검출 가능
- 사과가 YOLO에서 대부분 sports ball로 검출되므로 YOLO 클래스로 빨간 공과 빨간 사과를 구분할 수 없음. 연습 맵의 공은 흰색·검은색이라 색 판별로 제외되지만 대회 맵 방해 물체 색은 당일 확인 필요
- apple 클래스 신뢰도가 최저 0.20으로 `YOLO_CONF` 하한과 같음. 하한을 올리면 먼 사과 누락이 늘어남
- 화면 가장자리에 걸친 사과는 색 분할로 검출하지 않음. 로봇이 회전해 화면 안으로 들어오면 검출됨
- `to_world()`는 로봇 중심을 카메라 위치로 사용하므로 약 0.02 m 편향이 있음
- 합성 이미지 결과는 실제 사과 모양과 조명을 반영하지 않음

## 관련 자료

- 구현 PR: sar-robot [#6](https://github.com/Tech-Week-2026-KimEPark/sar-robot/pull/6), YOLO 원본 결과 보관은 sar-robot [#10](https://github.com/Tech-Week-2026-KimEPark/sar-robot/pull/10), 색 분할 보완·클래스 무관 NMS·방해 물체 오검출 수정은 sar-robot [#17](https://github.com/Tech-Week-2026-KimEPark/sar-robot/pull/17)
- 원본 인터페이스: [과제와 구현 기준](../../reference/sar-과제-구현-기준.md) 6장 계산식, 7.2절 인터페이스, 9장 설계 결정
- 관련 기능 문서: [viz](viz.md), [hsv_tuner](hsv_tuner.md)
