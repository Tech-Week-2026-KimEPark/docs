# INSTALL.md: 개발 환경 설치 (PNU TECH WEEK 2026 Search & Rescue)

팀원 4명 모두 대회 전에 완료하십시오. 대회장 네트워크가 느릴 수 있으므로 다운로드가 필요한 항목은 사전에 다운로드하십시오.
버전은 주최측 레포 노트북(`TECH-WEEK-26_Physical-AI.ipynb`) 기준입니다.

---

## 요약 체크리스트

- [ ] Python 3.10 (최대 3.11) 설치
- [ ] Webots **R2025a** 설치
- [ ] 파이썬 패키지 설치, numpy 버전 1.23.5 확인
- [ ] 레포 clone, YOLO 가중치 `yolo11n.pt` 다운로드
- [ ] Webots에서 월드 3개 첫 실행 (모델·텍스처 캐시)
- [ ] 동작 확인 3가지 통과
- [ ] 팀 git 저장소 push 권한, AI 계정 로그인 확인

---

## 1. Python 3.10

`numpy==1.23.5`는 Python 3.12 이상에서 설치되지 않습니다. 반드시 3.10(최대 3.11)을 사용하십시오.

| OS | 설치 |
|---|---|
| Ubuntu 22.04 | 기본 Python 3.10 사용 (`python3 --version`으로 확인) |
| Windows | python.org에서 3.10.x 설치, 설치 화면에서 **Add python.exe to PATH** 선택 |
| macOS | python.org 3.10.x 설치 또는 `brew install python@3.10` |

가상환경 생성 (권장):

```bash
# Ubuntu / macOS
python3.10 -m venv .venv
source .venv/bin/activate

# Windows (PowerShell)
py -3.10 -m venv .venv
.venv\Scripts\activate
```

---

## 2. Webots R2025a

월드 파일이 R2025a 기준이므로 버전을 반드시 일치시키십시오.

**Ubuntu**

```bash
cd ~/Downloads
wget https://github.com/cyberbotics/webots/releases/download/R2025a/webots_2025a_amd64.deb
sudo apt install ./webots_2025a_amd64.deb
```

**Windows / macOS**

[R2025a 릴리스 페이지](https://github.com/cyberbotics/webots/releases/tag/R2025a)에서 설치 파일을 받아 설치하십시오.

| OS | 파일 |
|---|---|
| Windows | `webots-R2025a_setup.exe` |
| macOS | `webots-R2025a.dmg` |

---

## 3. 파이썬 패키지

1장의 Python 3.10 환경(가상환경 활성화 상태)에서 실행하십시오.

```bash
pip install numpy==1.23.5 opencv-python==4.8.0.74 scikit-image==0.19.3 matplotlib==3.7.5 scipy
```

**PyTorch (환경에 맞는 한 줄만 실행)**

```bash
# CPU (기본)
pip install torch==2.8.0 torchvision==0.23.0 --index-url https://download.pytorch.org/whl/cpu

# NVIDIA GPU: cu126 부분을 자기 CUDA 버전에 맞게 변경
pip install torch==2.8.0 torchvision==0.23.0 --index-url https://download.pytorch.org/whl/cu126

# macOS (Apple Silicon 포함)
pip install torch==2.8.0 torchvision==0.23.0
```

**YOLO**

```bash
pip install ultralytics
```

**버전 확인**

```bash
python -c "import numpy, cv2, torch, ultralytics; print(numpy.__version__, cv2.__version__, torch.__version__, ultralytics.__version__)"
```

numpy가 `1.23.5`가 아니면 다시 설치하십시오.

```bash
pip install numpy==1.23.5
```

---

## 4. 레포와 YOLO 가중치

```bash
git clone https://github.com/kyu-rae-kim/PNU-TECHWEEK-260930.git
cd PNU-TECHWEEK-260930
python -c "from ultralytics import YOLO; YOLO('models/YOLO/yolo11n.pt')"
```

레포의 `models/YOLO/` 폴더는 비어 있습니다(`.gitkeep`만 존재). 위 명령을 실행하면 `models/YOLO/yolo11n.pt`가 다운로드됩니다.
컨트롤러 안에서는 `../../models/YOLO/yolo11n.pt` 경로로 불러옵니다 (컨트롤러 실행 위치가 `controllers/<이름>/`이기 때문).

---

## 5. Webots 설정과 첫 실행

### 5.1 Python 경로 지정

가상환경을 사용하면 Webots가 가상환경의 Python을 사용하도록 지정하십시오.

1. Webots 메뉴 **Tools → Preferences → General → Python command**
2. 가상환경 Python의 전체 경로 입력

| OS | 예시 |
|---|---|
| Ubuntu / macOS | `/home/<사용자>/PNU-TECHWEEK-260930/.venv/bin/python` |
| Windows | `C:\Users\<사용자>\PNU-TECHWEEK-260930\.venv\Scripts\python.exe` |

### 5.2 월드 첫 실행 (인터넷 필요)

월드 파일은 가구·로봇 모델을 인터넷에서 내려받습니다. 첫 실행 시 다운로드 후 캐시에 저장되므로 아래 3개를 지금 한 번씩 여십시오.

| 월드 | 용도 |
|---|---|
| `worlds/apartment.wbt` | 대회 연습 맵 |
| `worlds/breakroom_teleop_yolo.wbt` | Webots·Python·YOLO 연결 확인 |
| `worlds/breakroom_ground_truth.wbt` | 오도메트리 오차 측정 (D) |

---

## 6. 동작 확인

| 순서 | 월드 | 확인 방법 | 통과 기준 |
|---|---|---|---|
| 1 | `apartment.wbt` | 시뮬 실행, 3D 화면 클릭 후 W/A/S/D | 로봇 이동 |
| 2 | `breakroom_teleop_yolo.wbt` | 시뮬 실행 | 카메라 창에 사과 검출 상자 표시 |
| 3 | `breakroom_ground_truth.wbt` | 시뮬 실행, 로봇 이동 | 화면에 x, y, θ 값 표시 |

2번이 통과하면 Webots, Python, OpenCV, PyTorch, YOLO 연결이 모두 정상입니다.

---

## 7. 역할별 추가 준비

| 담당 | 준비 |
|---|---|
| A | 팀 git 저장소 생성, `CONTEXT.md`·`INSTALL.md` 업로드, 팀원 push 권한 확인 |
| B | `apartment.wbt`에서 라이다 값 출력 확인 (`tb3_lidar` 컨트롤러) |
| C | `breakroom_teleop_yolo.wbt`에서 추론 1회 시간 체감 확인 |
| D | `breakroom_ground_truth.wbt` 실행 확인 |
| 전원 | VS Code, 충전기, 멀티탭, AI 계정 로그인 |

---

## 8. 문제 해결

| 증상 | 원인 | 조치 |
|---|---|---|
| Webots 콘솔에 `No module named 'cv2'` (또는 `ultralytics`) | Webots가 다른 Python 사용 | 5.1의 Python command 경로 확인 |
| `numpy` 설치 실패 | Python 3.12 이상 사용 | Python 3.10 가상환경으로 재설치 |
| import 시 numpy 관련 오류 | 다른 패키지 설치 중 numpy 버전 변경 | `pip install numpy==1.23.5` |
| `yolo11n.pt` 파일 없음 오류 | 가중치 미다운로드 또는 경로 오류 | 4장 명령 재실행, 컨트롤러에서는 `../../models/YOLO/yolo11n.pt` 사용 |
| 월드 로딩 지연, 텍스처 누락 | 첫 실행 시 모델 다운로드 | 인터넷 연결 상태에서 다시 열기 |
| `cv2.imshow` 창 응답 없음 | 이벤트 처리 누락 | 매 루프 `cv2.waitKey(1)` 호출 |
| 터미널에서 실행 시 `No module named 'controller'` | Webots 밖에서 실행 | Webots에서 월드를 열고 시뮬 실행 |
| YOLO 추론으로 시뮬 진행 속도 저하 | CPU 추론 시간 | `YOLO_EVERY` 증가, `imgsz=480`, GPU 사용 가능 시 `device="cuda"` 또는 `"mps"` |
