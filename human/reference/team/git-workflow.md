# Git 작업 규칙

sar-robot 저장소의 브랜치, 커밋 메시지, PR, 머지, 태그 규칙입니다. docs 저장소의 문서 작업에도 같은 규칙을 적용합니다.

## 브랜치

`main`은 보호 브랜치입니다. `main`에 직접 push할 수 없으며 PR로만 머지합니다.

작업 브랜치 이름은 `<type>/<내용>` 형식입니다. 내용은 영문 소문자와 하이픈으로 작성하십시오.

| type | 용도 | 예시 |
|---|---|---|
| `feat` | 기능 추가 | `feat/frontier-explorer` |
| `fix` | 오류 수정 | `fix/odometry-turn-sign` |
| `refactor` | 동작 변경 없는 구조 개선 | `refactor/robot-io-devices` |
| `test` | 테스트 추가·수정 | `test/astar-edge-cases` |
| `docs` | 문서 작업 | `docs/team-conventions` |
| `chore` | 설정·의존성·CI | `chore/ci-ruff` |
| `perf` | 성능 개선 | `perf/grid-update-numpy` |

- 브랜치 1개에는 기능 1개만 작업함
- 브랜치 이름에 `claude`, `codex`, `gpt` 등 AI 도구 이름을 넣지 않음
- 역할별 고정 브랜치(`a-integ` 등)는 사용하지 않음. 원본 계획서의 브랜치 규칙을 이 문서로 대체함

작업 시작 순서는 다음과 같습니다.

```bash
git switch main
git pull
git switch -c feat/frontier-explorer
```

## 커밋 메시지

커밋 제목은 `type(scope): 명사형 요약` 형식입니다. type은 브랜치 type과 같습니다. scope는 변경한 모듈 이름입니다.

| scope | 대상 |
|---|---|
| `mission` | `sar/mission.py` |
| `io` | `sar/robot_io.py`, `sar_main.py` |
| `config` | `sar/config.py` |
| `mapping` | `sar/grid_map.py` |
| `planning` | `sar/planner.py` |
| `perception` | `sar/perception.py`, `controllers/hsv_tuner/`, `models/` |
| `control` | `sar/local_control.py` |
| `localization` | `sar/odometry.py` |
| `viz` | `sar/viz.py` |
| `world` | `worlds/`, `protos/` |

`sar_main.py`와 `sar/`로 시작하는 경로는 `controllers/sar_main/` 기준입니다. 파일 구조의 원본은 [과제와 구현 기준](../sar-과제-구현-기준.md) 7.1절입니다.

본문이 필요하면 빈 줄 뒤에 변경 이유를 개조식으로 작성하십시오.

```text
fix(localization): 제자리 회전 시 odometry 위치 누적 오류 수정

- 회전각이 0에 가까울 때 원호 반지름 계산에서 0 나눗셈 발생
- 회전각 1e-9 rad 미만은 직선 이동으로 적분
```

- 제목은 한국어 명사형으로 끝냄. "~했다", "~함" 대신 "~ 수정", "~ 추가"
- 커밋 1개에는 변경 1개만 포함함
- 파일 목록, 이모지, AI 도구 서명(`Co-Authored-By` 등)을 넣지 않음

## PR과 리뷰

| 항목 | 규칙 |
|---|---|
| PR 제목 | 커밋 제목과 같은 형식 |
| PR 본문 | 저장소의 PR 템플릿 항목을 모두 작성 |
| 리뷰어 | 1명 이상. 기본은 백업 짝. 다른 역할의 파일을 수정했으면 해당 담당자 추가 |
| 머지 조건 | 승인 1명, CI 통과(`ruff check`, `ruff format --check`, `pytest`) |
| 머지 방식 | Squash merge. 머지 후 작업 브랜치 삭제 |
| 머지 수행 | 사전 개발 기간은 PR 작성자, 대회 당일은 A만 수행 |

리뷰는 요청 후 30분 안에 시작하십시오. 대회 당일에는 리뷰 대신 A가 단독 테스트 통과 여부만 확인하고 머지합니다.

PR 전에 다음 명령을 저장소 루트에서 실행하십시오.

```bash
.venv/bin/ruff format .
.venv/bin/ruff check .
.venv/bin/python -m pytest -q
```

## 태그

| 태그 | 시점 | 조건 |
|---|---|---|
| `v1` | 첫 완주(대회 1:15 목표) | 탐색 → 대상 → 목적지 → 복귀 1회 완주 |
| `v2` | 기능 동결(대회 2:15) | 이후 오류 수정만 머지 |
| `final` | 제출 직전 | 제출 패키지와 같은 커밋 |

태그는 A가 `main`에서 생성합니다.

```bash
git switch main
git pull
git tag v1
git push origin v1
```

통합 후 동작이 실패하면 마지막 태그의 코드로 되돌리십시오. 기록을 유지하기 위해 `reset` 대신 되돌림 커밋을 만듭니다.

```bash
git switch -c fix/rollback-to-v1
git checkout v1 -- controllers/
git commit -m "fix(mission): v1 태그 코드로 되돌림"
```
