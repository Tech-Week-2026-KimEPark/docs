# hsv_tuner 기능 설명

이 문서는 sar-robot `controllers/hsv_tuner/`에 구현한 HSV 임계값 튜닝과 인식 성능 측정 컨트롤러의 동작과 검증 결과를 설명합니다. 컨트롤러는 `hsv_tuner.py`, 자동 주행 판단 로직은 `auto_drive.py`, 측정 결과 분석은 `scripts/analyze_measurements.py`에 있습니다.

## 과제와의 관계

- 인지 담당 산출물인 색 임계값 기록과 인식 화면을 만드는 도구임
- [과제와 구현 기준](../../reference/sar-과제-구현-기준.md) 14장의 대회 시작 후 확인 항목 중 색별 사과의 YOLO 클래스·신뢰도, YOLO 추론 시간을 측정함
- 미션 컨트롤러와 별개로 실행하며 상태 머신에는 포함되지 않음

## 동작 원리

컨트롤러는 매 step 다음 순서로 동작합니다.

1. 엔코더 오도메트리로 로봇 pose 갱신. 시작 pose는 `START_X`, `START_Y`, `START_THETA`
2. 카메라 이미지를 받고 트랙바 값으로 HSV 범위 갱신
3. `YOLO_EVERY` step마다 `TargetDetector.detect_all()` 실행
4. 검출 상자와 색 마스크를 한 창에 나란히 표시
5. `LOG_INTERVAL`마다 콘솔에 요약 1줄 출력
6. 자동 주행 중이면 자동 주행 로직이 속도 명령과 측정 요청을 결정
7. 키 입력 처리. OpenCV 창의 밀린 입력을 모두 읽고, Webots 3D 화면의 현재 눌린 키를 `RobotIO.keys()`로 읽음
8. 라이다 안전 필터(`local_control.safety_filter`)를 거쳐 `RobotIO.drive(v, w)`로 구동

검출은 미션과 같은 `TargetDetector`를 사용하므로 튜닝 화면의 결과가 실제 검출 결과와 같습니다. `controllers/sar_main`을 import 경로에 추가해 `sar` 패키지를 사용합니다.

### 수동 조종

조종 키는 누르고 있는 동안만 움직입니다. 입력 경로는 두 가지입니다.

| 입력 창 | 방식 | 이유 |
|---|---|---|
| Webots 3D 화면 | `RobotIO.keys()`가 매 step 현재 눌린 키를 반환. 떼면 즉시 정지 | Webots 키보드는 누르고 있는 키를 매 step 전달함 |
| `hsv_tuner` 창 | 키 입력 1회를 `TUNER_KEY_HOLD` 동안 유지. 키 반복 입력이 유지 시간을 갱신 | OpenCV는 키를 뗀 시점을 알려주지 않음 |

처음 구현은 OpenCV 창의 키 1회 입력으로 속도를 계속 유지했습니다. 한 step에 입력을 1개만 읽어 시뮬레이션이 느리면 입력이 밀렸고, 회전 속도 1.0 rad/s는 화면 지연 동안 사과를 지나쳤습니다. 사용자 확인 결과 사과까지 도달하기 어려운 수준이어서 다음과 같이 바꿨습니다.

- 누르는 동안만 이동. 전진 키와 회전 키를 함께 누르면 곡선 주행
- OpenCV 창의 밀린 입력을 한 step에 모두 처리 (`CV2_MAX_EVENTS`)
- 회전 속도를 0.6 rad/s로 낮추고 Shift(Webots 화면) 또는 대문자(`hsv_tuner` 창) 입력 시 30% 저속
- 방향키 지원, 화면에 현재 속도 명령 표시

Webots 키보드는 `controller` 모듈 장치이므로 `robot_io.py`의 `RobotIO.keys()`로 읽습니다. 키 코드 변환은 Webots 없이 테스트하도록 `decode_keys()`로 분리했습니다.

H 하한이 상한보다 크면 0을 넘는 범위로 보고 두 범위 (0~상한) ∪ (하한~179)로 나눕니다. 빨강 범위를 트랙바 6개로 조정하기 위한 방식입니다.

| 키 | 동작 |
|---|---|
| `w`, `s`, `a`, `d` 또는 방향키 | 누르는 동안 전진, 후진, 좌회전, 우회전 |
| Shift (Webots 화면), 대문자 (`hsv_tuner` 창) | 저속 |
| `space` | 정지, 자동 주행 취소 |
| `m` | 현재 프레임을 다시 검출해 CSV에 측정 행 추가, 프레임 PNG 저장 |
| `g` | 자동 접근 시작 |
| `o` | 회전 측정 시작 |
| `p` | 프레임 PNG만 저장 |
| `r` | 현재 범위를 `HSV_RANGES` 형식으로 출력 |
| `q` 또는 창 닫기 | 로봇 정지 후 종료 |

### 자동 주행

수동 조종으로 거리별 측정 지점에 정확히 세우기는 어렵습니다. 자동 주행은 측정 지점 도달과 측정 기록을 자동으로 수행합니다.

자동 접근(`AutoApproach`)은 다음 단계로 동작합니다.

| 단계 | 동작 | 다음 단계 조건 |
|---|---|---|
| SCAN | `TUNER_SCAN_W`로 제자리 좌회전 | 대상 검출 → ALIGN. `TUNER_SCAN_TURNS`회전 동안 미검출 → 종료 |
| ALIGN | 각속도 $\omega = k \beta$로 제자리 회전. $k$는 `TUNER_ALIGN_GAIN` | $\lvert \beta \rvert$ ≤ `TUNER_ALIGN_TOL` → APPROACH |
| APPROACH | `TUNER_APPROACH_V`로 전진하며 같은 식으로 방향 보정. 방위각이 허용 오차의 4배를 넘으면 정지 후 재정렬 | 추정 거리 ≤ `TUNER_STOP_DIST` → 측정 후 종료 |

- APPROACH 진입 시 현재 추정 거리보다 먼 측정 지점은 건너뜀
- 추정 거리가 측정 지점(`TUNER_MEASURE_DISTS`) 이하가 된 첫 검출 step에 측정 요청
- 대상을 `TUNER_LOST_TIMEOUT` 이상 놓치면 SCAN으로 복귀

회전 측정(`SpinSurvey`)은 제자리에서 1회전하며 누적 회전량이 `TUNER_SURVEY_STEP`의 배수를 지날 때마다 측정합니다. 대상이 없는 방향도 `none` 행으로 기록되므로 방해 요소의 오검출 여부를 방향별로 확인할 수 있습니다.

측정 요청은 YOLO를 새로 실행한 step에서만 발생합니다. 이전 검출 결과로 측정 행을 기록하지 않기 위한 조건입니다. 조종 키를 누르거나 안전 필터가 전진을 막으면 자동 주행을 종료합니다.

측정 CSV의 행 종류는 다음과 같습니다. `note` 열에는 측정 사유(`manual`, `auto 2.0m d=1.99m`, `survey 30deg` 등)를 기록합니다.

| `kind` | 내용 |
|---|---|
| `target` | 최종 대상 검출. 거리, 방위각, 대상 월드 좌표 포함 |
| `yolo_raw` | 색 판별 전 YOLO 상자. 클래스 이름, 신뢰도, 상자 안 색 비율 포함 |
| `none` | 검출 없음 |

콘솔 요약의 실시간 배율 `rtf`는 로그 간격 동안 진행한 시뮬레이션 시간을 실제 시간으로 나눈 값입니다. 1보다 많이 작으면 YOLO 추론이 시뮬레이션을 늦추는 상태입니다.

## 인터페이스와 설정값

컨트롤러 안의 보조 함수는 다음과 같습니다. 다른 모듈에서 호출하지 않습니다.

| 함수 | 입력 | 출력 |
|---|---|---|
| `initial_values(ranges)` | `HSV_RANGES` 항목 | 트랙바 초기값 6개 |
| `ranges_from_values(values)` | 트랙바 값 6개 | HSV 범위 목록 |
| `measurement_rows(t, pose, detections, yolo_boxes, names, ranges, yolo_ms, note)` | 측정 시점 정보 | CSV 행 목록 |
| `append_csv(path, rows)` | 경로, 행 목록 | 저장 성공 여부. 새 파일이면 머리행 작성 |
| `summary_line(t, pose, detections, yolo_boxes, names, yolo_ms, rtf)` | 주기 로그 정보 | 콘솔 요약 1줄 |
| `AutoApproach(t, pose, dists).update(t, pose, detections, fresh)` | 시각, pose, 검출 목록, 새 검출 여부 | `Command(v, w, measure, message, done)` |
| `SpinSurvey(t, pose).update(t, pose, detections, fresh)` | 같음 | 같음 |
| `cv2_key_name(code)` | `cv2.waitKeyEx` 코드 | `(키 이름, 대문자 여부)` |
| `KeyHold.press(name, now, fine)`, `KeyHold.held(now)` | 키 이름, 실제 시간 [s] | 유지 중인 키 집합과 저속 여부 |
| `drive_from_keys(keys, fine)` | 눌린 조종 키, 저속 여부 | `(v, w)`. 반대 방향 키는 상쇄 |
| `RobotIO.keys()` (`sar/robot_io.py`) | 없음 | Webots 화면의 `(눌린 키 이름 집합, Shift 여부)` |

| 설정값 | 값 | 의미 |
|---|---|---|
| `TUNER_V` | 0.12 | 키보드 전진 속도 [m/s] |
| `TUNER_W` | 0.6 | 키보드 회전 속도 [rad/s] |
| `TUNER_FINE_SCALE` | 0.3 | 저속 입력 시 속도 배율 |
| `TUNER_KEY_HOLD` | 0.6 | `hsv_tuner` 창 키 입력 유지 시간 [s], 실제 시간 |
| `KEYBOARD_MAX_KEYS` | 7 | Webots 키보드 동시 입력 최대 개수 |
| `TUNER_FRAME_DIR` | `"frames"` | 프레임 저장 폴더. 컨트롤러 폴더 기준 |
| `TUNER_LOG_FILE` | `"output/measurements.csv"` | 측정 CSV 경로. 컨트롤러 폴더 기준 |
| `LOG_INTERVAL` | 1.0 | 콘솔 요약 주기 [s] |
| `TUNER_SCAN_W` | 0.4 | 자동 주행 제자리 회전 속도 [rad/s] |
| `TUNER_SCAN_TURNS` | 1.0 | 자동 접근 탐색 한도 [회전] |
| `TUNER_SURVEY_STEP` | 0.5236 | 회전 측정 간격 [rad], 30° |
| `TUNER_ALIGN_TOL` | 0.05 | 정렬 완료 방위각 오차 [rad] |
| `TUNER_ALIGN_GAIN` | 1.5 | 정렬 각속도 이득 [1/s] |
| `TUNER_APPROACH_V` | 0.08 | 자동 접근 전진 속도 [m/s] |
| `TUNER_MEASURE_DISTS` | (3.0, 2.0, 1.0) | 자동 측정 지점 [m] |
| `TUNER_STOP_DIST` | 0.5 | 자동 접근 정지 거리 [m] |
| `TUNER_LOST_TIMEOUT` | 2.0 | 대상 미검출 허용 시간 [s] |

## 검증 결과

2026-09-30 sar-robot `feat/tuner-hold-to-drive` 브랜치에서 확인했습니다. Webots가 사용하는 가상환경(Python 3.10.11, ultralytics 8.4.166)으로 실행했습니다.

| 항목 | 명령·조건 | 결과 |
|---|---|---|
| 단위 테스트 | `python -m pytest -q tests/test_hsv_tuner.py tests/test_auto_drive.py tests/test_analyze_measurements.py tests/test_robot_io.py` | 29개 통과 |
| 입력 변환 | 테스트: Webots 키 코드(문자, 방향키, Shift), `cv2.waitKeyEx` 코드, 키 유지·만료·반대 방향 해제, 속도 계산 | 모두 기대값과 일치 |
| 누르는 동안 이동 | 가짜 로봇: Webots 화면에서 5~20번째 step 동안 `w`, 30번째 step에 `hsv_tuner` 창에서 `a` 1회 | 전진은 5~20번째 step에만 출력. 회전은 0.6초 유지 후 정지 |
| 분석 스크립트 | 가상 세계 측정 CSV(회전 측정 후 자동 접근, 초록 사과 방해 요소) | 대상 검출 5건 모두 실제 사과 대응, 역산 지름 중앙값 0.093 m(가상 세계 값 0.095 m), 초록 사과 상자 색 비율 0.00 |
| 자동 주행 상태 전환 | 테스트: 탐색 한도, 정렬 방향, 측정 지점, 재정렬, 새 검출 조건, 대상 놓침, 회전 측정 12회 | 모두 기대값과 일치 |
| 트랙바 변환 | 테스트: `HSV_RANGES` 4색 | 초기값 변환 후 역변환 결과가 원래 범위와 같음 |
| 전체 실행 | 가짜 로봇(빨간 원 이미지, 전진 엔코더), 실제 YOLO, 20번째 반복에 `m`, 40번째에 `q` | 콘솔 요약 출력, CSV 2행(`target`, `yolo_raw`) 기록, 프레임 저장, 정상 종료 |
| 위치 추정 방향 | 위 실행, 시작 방향 180° | 전진 시 x 감소 |
| 창 닫기 | 가짜 로봇, 4번째 반복에 창 제거 | 종료 메시지 출력, 마지막 속도 명령 정지 |
| Webots 실행 | apartment.wbt, 컨트롤러 필드 `hsv_tuner` | 창 표시와 조작 확인. 창 닫기 시 `cv2.error` 종료를 확인해 위 창 닫기 처리 추가 |
| 자동 접근 전체 실행 | 가상 세계: 시작 pose 기준 왼쪽 40°, 2.8 m에 사과. 투영 모델로 빨간 원 렌더링, 속도 명령대로 이동. 실제 YOLO | 탐색·정렬·접근 완료. 3 m 지점 생략, 2 m(실제 1.86 m), 1 m(실제 0.97 m), 정지(실제 0.48 m) 측정. 대상 월드 좌표 오차 0.02 m 이내 |
| 회전 측정 전체 실행 | 같은 가상 세계에서 `o` 키 | 12회 측정, 30°와 60° 측정에서 대상 검출, 1회전 후 종료 |
| Webots 측정 기록, 자동 주행 | apartment.wbt에서 `g`, `o` 키 | CSV 기록과 프레임 저장 확인. 자동 접근 2 m 측정 기록됨 |
| Webots 수동 조종 | 누르는 동안 이동 방식 | 미확인 |

## 한계와 확인 필요 항목

- 위치는 엔코더만으로 추정하므로 오래 주행하면 오차가 누적됨. apartment.wbt에서 약 12 m 수동 주행 후 추정 위치가 소파 안쪽으로 계산되어, 분석 스크립트가 실제 사과 검출을 오검출 후보로 분류함. 위치를 쓰는 측정은 Reset 직후 짧게 이동한 뒤 수행해야 함. 나침반 보정은 미션의 INIT_SPIN 구현 후 적용 가능
- `hsv_tuner` 창 입력은 키를 뗀 뒤 최대 `TUNER_KEY_HOLD` 동안 더 움직임. 정밀 조종은 Webots 화면 입력이 적합함
- `m` 키는 현재 프레임으로 YOLO를 다시 실행하므로 해당 step의 시뮬레이션이 추론 시간만큼 지연됨
- 자동 접근은 가장 가까운 대상을 따라가므로, 두 사과가 함께 보이면 가까운 사과로 접근함
- 자동 주행의 측정 지점은 추정 거리 기준임. 실제 거리와의 차이가 측정 대상이므로 CSV의 로봇 pose와 사과 실제 좌표로 실제 거리를 계산해야 함
- 안전 필터는 진행 방향만 검사하므로 제자리 회전 중 옆 장애물은 막지 않음
- 튜닝 중 월드를 저장하면 `controller` 필드와 시뮬레이션 상태 값이 월드 파일에 기록됨

## 관련 자료

- 구현 PR: sar-robot [#6](https://github.com/Tech-Week-2026-KimEPark/sar-robot/pull/6), 측정 기록과 자동 주행은 sar-robot [#10](https://github.com/Tech-Week-2026-KimEPark/sar-robot/pull/10)
- 사용 절차: sar-robot [HSV 임계값 튜닝과 인식 성능 측정](https://github.com/Tech-Week-2026-KimEPark/sar-robot/blob/main/docs/human/how-to/hsv-tuning.md)
- 관련 기능 문서: [perception](perception.md)
