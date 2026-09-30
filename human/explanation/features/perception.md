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
| 1 | YOLO11n으로 apple, orange, sports ball 후보 상자 검출 | `YOLO_CLASSES`, `YOLO_CONF` |
| 2 | 상자 안 대상 색 픽셀 비율이 하한 이상인 후보만 선택 | `HSV_RANGES`, `COLOR_RATIO_MIN` |
| 3 | 2단계 후보가 없으면 색 마스크에서 면적·원형도 조건을 만족하는 덩어리 검출 | `MIN_BLOB_AREA`, `MIN_CIRCULARITY` |
| 4 | 중심이 화면 가운데선보다 기준 이상 위인 후보를 식탁 위 물체로 제외 | `HORIZON_MARGIN` |
| 5 | 거리와 방위각 계산 후 가까운 순서로 정렬 | `TARGET_DIAMETER`, `CAMERA_FOV` |

YOLO는 사과를 색과 관계없이 같은 클래스로 검출합니다. 따라서 대상 색 판별은 2단계의 HSV 색 비율로 수행합니다. YOLO가 놓친 먼 거리의 작은 사과는 3단계 색 분할로 보완합니다.

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

## 인터페이스와 설정값

| 함수 | 입력 | 출력 |
|---|---|---|
| `TargetDetector(color, model_path, device, ranges)` | 대상 색, YOLO 가중치 경로 또는 `None` | 검출기. YOLO는 생성 시 1회 로드 |
| `TargetDetector.detect(bgr)` | BGR 이미지 또는 `None` | 가장 가까운 대상 `dict` 또는 `None` |
| `TargetDetector.detect_all(bgr)` | BGR 이미지 또는 `None` | 대상 목록. 가까운 순서 |
| `TargetDetector.to_world(det, pose)` | 검출 결과, `(x, y, theta)` | 대상 월드 좌표 `(x, y)` |
| `Confirm.update(seen, xy, dist)` | 검출 여부, 대상 월드 좌표, 거리 [m] | `(확정 여부, 위치 추정 또는 None)` |
| `is_excluded(xy, found, radius)` | 대상 월드 좌표, 구조 완료 위치 목록 | 반경 이내 여부 |

검출 결과의 키 정의는 sar-robot [모듈 인터페이스](https://github.com/Tech-Week-2026-KimEPark/sar-robot/blob/main/docs/human/reference/interfaces.md)에 있습니다. 측정·튜닝용으로 마지막 `detect_all()` 호출의 YOLO 원본 상자(`last_yolo`)와 추론 시간(`last_yolo_ms`)을 보관합니다. `last_yolo`에는 색 판별에서 탈락한 상자와 상자 안 색 비율(`ratio`)이 포함됩니다.

| 설정값 | 값 | 의미 |
|---|---|---|
| `YOLO_CLASSES` | `[47, 49, 32]` | COCO apple, orange, sports ball |
| `YOLO_CONF` | 0.20 | YOLO 신뢰도 하한 |
| `YOLO_EVERY` | 4 | YOLO 추론 주기 [step]. 미션에서 적용 |
| `COLOR_RATIO_MIN` | 0.25 | YOLO 상자 안 대상 색 비율 하한 |
| `USE_COLOR_FALLBACK` | `True` | YOLO 대상이 없을 때 색 분할 사용 여부 |
| `MIN_BLOB_AREA` | 60 | 색 분할 덩어리 면적 하한 [px] |
| `MIN_CIRCULARITY` | 0.6 | 색 분할 덩어리 원형도 $4\pi A / P^2$ 하한 |
| `HORIZON_MARGIN` | 20 | 높이 제외 기준 [px] |
| `TARGET_DIAMETER` | 0.095 | 사과 지름 [m] |
| `CONFIRM_FRAMES` | 4 | 확정에 필요한 연속 검출 수 |
| `CONFIRM_MATCH_RADIUS` | 0.5 | 같은 대상으로 보는 위치 차이 상한 [m] |
| `CONFIRM_MIN_DIST` | 0.3 | 가중 평균의 거리 하한 $d_{\min}$ [m] |
| `FOUND_EXCLUDE_RADIUS` | 0.6 | 구조 완료 위치 주변 제외 반경 [m] |

## 검증 결과

2026-09-30 sar-robot `feat/tuner-auto-measure` 브랜치에서 확인했습니다. Python은 3.10.11, ultralytics는 8.4.166입니다.

| 항목 | 명령·조건 | 결과 |
|---|---|---|
| 단위 테스트 | `python -m pytest -q tests/test_perception.py` | 25개 통과 |
| 단독 실행 | `controllers/sar_main`에서 `python -m sar.perception` | 합성 이미지 빨간 원(반지름 12 px) 색 분할 검출, 거리 2.27 m |
| YOLO 경로 | 실제 `yolo11n.pt`, 회색 배경 빨간 원(반지름 30 px) 합성 이미지 | sports ball(32) 신뢰도 0.41로 검출, 색 판별 통과, 거리 0.90 m |
| 거리식 | 위 검출의 상자 폭 59.1 px, 방위각 −0.145 rad | 식 계산값 0.90 m와 일치 |
| YOLO 추론 시간 | 같은 이미지 20회, CPU | 중앙값 54.7 ms, 최소 51.1 ms, 최대 82.9 ms |
| 오판정 방지 | 테스트: 초록·주황 원, 작은 덩어리, 긴 사각형, 가운데선 위 원 | 모두 검출 제외 |
| 실패 처리 | 테스트: 모델 로드 실패, 추론 예외, 이미지 `None` | 예외 없이 색 분할 또는 빈 목록 |
| Webots 실제 검출 | apartment.wbt 조명 | 미확인 |

## 한계와 확인 필요 항목

- Webots 렌더링 이미지에서의 YOLO 클래스·신뢰도, HSV 범위, 거리 오차는 미확인임. 측정 절차는 sar-robot [HSV 임계값 튜닝과 인식 성능 측정](https://github.com/Tech-Week-2026-KimEPark/sar-robot/blob/main/docs/human/how-to/hsv-tuning.md)에 있음
- 사과 지름 설정값과 PROTO 치수 불일치 가능성, YOLO 대상이 있을 때 색 분할 미실행, 미검출 1회에 연속 확인 초기화는 [sar-robot 병합 내용 분석](../sar-robot-병합-분석.md) 6장 7·8·5번 항목임
- `to_world()`는 로봇 중심을 카메라 위치로 사용하므로 약 0.02 m 편향이 있음
- 합성 이미지 결과는 실제 사과 모양과 조명을 반영하지 않음

## 관련 자료

- 구현 PR: sar-robot [#6](https://github.com/Tech-Week-2026-KimEPark/sar-robot/pull/6), YOLO 원본 결과 보관은 sar-robot [#10](https://github.com/Tech-Week-2026-KimEPark/sar-robot/pull/10)
- 원본 인터페이스: [과제와 구현 기준](../../reference/sar-과제-구현-기준.md) 6장 계산식, 7.2절 인터페이스, 9장 설계 결정
- 관련 기능 문서: [viz](viz.md), [hsv_tuner](hsv_tuner.md)
