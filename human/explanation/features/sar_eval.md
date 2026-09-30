# sar_eval 기능 설명

이 문서는 sar-robot `controllers/sar_eval/sar_eval.py`에 구현한 설계 근거 측정 컨트롤러와 `scripts/design_eval.py` 분석 스크립트의 동작과 검증 결과를 설명합니다. 측정 결과는 [SAR 로봇 시스템 설계서](../sar-시스템-설계.md) 6장에 있습니다.

## 과제와의 관계

- 제출 코드가 아닌 측정 도구입니다. 팀 코드(`controllers/sar_main/sar/`)는 수정하지 않습니다.
- 설계 선택(위치 추정, 지도, 경로, 인식, 상태 머신)을 같은 기록에서 조건별로 비교합니다.
- 측정 월드 `worlds/sar_apartment_eval.wbt`는 `sar_apartment.wbt`에서 로봇의 `supervisor TRUE`와 `controller "sar_eval"`만 다릅니다.

## 동작 원리

### 측정 컨트롤러

1. `controller.Robot`을 `controller.Supervisor`로 교체한 뒤 `RobotIO()`를 생성합니다. `RobotIO`는 생성 시 `Robot`을 import하므로 팀 코드 변경 없이 Supervisor 인스턴스가 생성됩니다.
2. `sar_main.py`와 같은 순서로 `Odometry`, `GridMap`, `TargetDetector`, `Mission`을 생성합니다.
3. 매 step `Mission.tick()` 전후로 센서 값, 실제 pose(`getSelf()`), 계산 시간, 몸체 접촉점을 기록합니다.
4. `DONE` 1초 후 또는 `TIME_LIMIT` 60초 초과 시 `simulationQuit(0)`으로 Webots를 종료합니다.

| `SAR_EVAL_MODE` | 조건 |
|---|---|
| `mission` (기본) | 현재 코드. 실제 pose는 기록에만 사용 |
| `truth` | `Odometry.update()` 대신 실제 pose 입력. 위치 추정 오차 제거 조건 |
| `explore`, `truth_explore` | 위 조건 + `TARGET_COUNT = 99` |

`SAR_EVAL_SET="CONFIRM_FRAMES=3"`처럼 설정값을 바꿀 수 있습니다. `sar` 모듈 import 전에 적용하므로 함수 기본 인자로 쓰인 설정값(`Confirm(frames=config.CONFIRM_FRAMES)`)에도 반영됩니다.

| 기록 파일 (`controllers/sar_eval/output/<모드>/`) | 내용 |
|---|---|
| `run.npz` | step별 `t`, `enc`, `compass`, `lidar`(360), `gt`, `est`, `state`, `cmd`, `tick_ms`, `yolo_ms`, `contact`, `plan_ms`, `wall`. 구조 위치, 나침반 보정값, 계획 호출 시간 |
| `frames/<step>.jpg` | 인식 step 카메라 화면 (JPG 품질 95) |
| `detections.csv` | 인식 step별 원본 화면 검출 결과 |
| `log.txt` | 미션 로그 |

몸체 접촉은 접촉점 높이 0.02 m 초과로 판정합니다. 바퀴·볼 캐스터와 바닥의 접촉은 제외됩니다.

### 분석 스크립트

`scripts/design_eval.py`는 기록을 다시 계산해 설계서 6장의 실험 번호별 결과를 `results.json`과 `design_*.png`로 저장합니다.

| 실험 | 계산 방식 |
|---|---|
| L (위치 추정) | 기록한 엔코더·나침반으로 `Odometry`를 변형 5종으로 다시 실행. 현재 조건의 재계산 결과는 기록한 추정값과 오차 0 m로 일치 |
| L2 (지도 품질) | 변형별 pose와 기록한 라이다로 `GridMap` 작성 후 실제 pose 지도와 장애물 칸 F1 비교 |
| P1, C3, M1 | 합성 입력으로 모듈 함수 직접 실행 |
| P2~P5, C1, C2, P6 | 실제 pose 지도(기준 지도)에서 `planner`와 `local_control` 실행. 주행은 오차 없는 차동 구동 적분 (64 ms) |
| V (인식) | 저장한 프레임에 검출기 변형 5종 실행. 라벨과 판정은 실제 pose와 월드 파일 물체 좌표 기준 |
| M5 (완주) | 실행별 구조 위치, 실제 복귀 오차, 접촉 수, 계산 시간 요약 |

## 인터페이스와 설정값

| 함수 | 입력 | 출력 |
|---|---|---|
| `sar_eval.true_pose(node)` | Supervisor 노드 | 실제 `(x, y, theta)` |
| `sar_eval.TruthOdometry(node)` | Supervisor 노드 | `update()`가 실제 pose를 사용하는 `Odometry` |
| `design_eval.replay_odometry(run, variant)` | 기록, 변형 이름 | pose 배열 |
| `design_eval.world_objects(path)` | 월드 파일 | 바닥 물체 `(이름, x, y)` 목록 |
| `design_eval.visible(grid, pose, xy)` | 기준 지도, pose, 물체 좌표 | 화각·거리·가시선 조건 충족 여부와 거리·방위각 |

| 설정값 (`sar_eval.py`) | 값 | 의미 |
|---|---|---|
| `EVAL_JPG_QUALITY` | 95 | 프레임 JPG 품질 |
| `EVAL_CONTACT_Z` | 0.02 m | 몸체 접촉 판정 높이 |
| `EVAL_DONE_HOLD` | 1.0 s | DONE 후 기록 유지 시간 |

## 검증 결과

2026-09-30, macOS(Apple Silicon), Webots R2025a, Python 3.10, sar-robot `main` `199349b` 기준입니다.

| 항목 | 명령·조건 | 결과 |
|---|---|---|
| pytest | `.venv/bin/python -m pytest -q tests/test_design_eval.py` | 5개 통과 |
| 재계산 일치 | 현재 조건 `replay_odometry(run, "kf_scale")`와 기록 추정값 비교 | 최대 차이 0 m |
| JPG 영향 | 원본 화면 검출 수와 JPG 프레임 검출 수 비교, 3855장 | 99.92% 일치 |
| 라벨 확인 | 가시 판정 프레임 33장 이미지 확인 | 31장 정확. 가려진 2장은 `MANUAL_NOT_VISIBLE`로 제외 |
| 측정 월드 실행 | `mission`, `truth`, `SAR_EVAL_SET=CONFIRM_FRAMES=3` 2회 | 4회 모두 자동 종료, 기록 저장 |

## 한계와 확인 필요 항목

- 위치 추정 변형 비교는 기록한 궤적의 재계산입니다. 추정 오차에 따라 주행 경로가 달라지는 효과는 포함하지 않습니다.
- 운동학 시뮬레이션은 기준 지도(라이다 높이 단면)와 오차 없는 pose를 사용합니다. 바퀴 미끄러짐, 라이다 높이 아래 물체, 보행자는 포함하지 않습니다.
- 가시선 라벨은 기준 지도 장애물만 반영합니다. 라이다 높이 아래 가구에 가려진 프레임은 이미지 확인으로 제외했습니다.
- 각 조건의 Webots 실행은 1회입니다. Webots 물리 계산은 같은 입력에서 같은 결과를 출력하므로 반복 실행 대신 조건 변경 실행을 사용했습니다.

## 관련 자료

- 구현: sar-robot `controllers/sar_eval/sar_eval.py`, `scripts/design_eval.py`, `tests/test_design_eval.py`, `worlds/sar_apartment_eval.wbt`
- 실행 방법: sar-robot [개발 환경 설정](https://github.com/Tech-Week-2026-KimEPark/sar-robot/blob/main/docs/human/how-to/dev-setup.md) 7장
- 결과: [SAR 로봇 시스템 설계서](../sar-시스템-설계.md)
