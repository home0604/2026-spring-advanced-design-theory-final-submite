# RSP_SSD_LAB: Raspberry Pi SSD 오브젝트 디텍션 실습

Raspberry Pi에서 가위바위보 손 모양을 SSD Object Detection 방식으로 실행하고, TFLite 양자화 전후의 모델 크기, 추론 시간, CPU/RAM 사용량을 비교하는 실습 자료다.

> 실습 자료 다운로드: [RSP_SSD_LAB.zip](assets/rsp-ssd-lab/RSP_SSD_LAB.zip)

## 실습 목표

- Raspberry Pi에서 `RPS_PreTrained_SSD.tflite`를 이용해 실시간 손 모양 탐지를 실행한다.
- `RPS_PreTrained_SSD.tflite`와 SSD-Lite 양자화 비교용 모델의 차이를 이해한다.
- MacBook에서 SSD-Lite 모델을 float32 TFLite와 int8 TFLite로 변환한다.
- Raspberry Pi에서 양자화 전후 추론 시간, 파일 크기, `ps` CPU/RSS 메모리를 비교한다.

## 포함 파일

```text
RSP_SSD_LAB/
├── README.md
├── requirements-mac.txt
├── requirements-pi.txt
├── assets/
│   └── representative_images/
├── docs/
│   └── 라즈베리파이_SSD_오브젝트디텍션_실행매뉴얼.md
├── models/
│   ├── RPS_PreTrained_SSD.tflite
│   ├── ssd_lite_rps.h5
│   ├── rps_ssd_lite_fp32.tflite
│   └── rps_ssd_lite_int8.tflite
└── scripts/
    ├── run_rps_ssd_camera.py
    ├── EX_03_Board_RPS_PreTrained_SSD.py
    ├── EX_01_Image_Capture.py
    ├── quantize_rps_ssd_lite.py
    └── benchmark_rps_ssd_lite_tflite.py
```

## 모델 설명

`models/RPS_PreTrained_SSD.tflite`는 Raspberry Pi에서 바로 실행하기 위한 SSD 계열 가위바위보 오브젝트 디텍션 모델이다.

| 항목 | 내용 |
|---|---|
| 입력 크기 | `[1, 320, 320, 3]` |
| 입력 dtype | `float32` |
| 모델 크기 | 약 11MB |
| 출력 | 손 위치 bounding box, class, confidence |
| 클래스 | `Scissors`, `Rock`, `Paper` |

이 모델은 TFLite 형식이라 온디바이스 실행에 적합하지만, int8 양자화 최경량 모델은 아니다. 양자화 비교에는 별도의 작은 SSD-Lite 모델을 사용한다.

| 모델 | 설명 |
|---|---|
| `models/rps_ssd_lite_fp32.tflite` | SSD-Lite float32 기준 모델 |
| `models/rps_ssd_lite_int8.tflite` | SSD-Lite int8 양자화 모델 |
| `models/ssd_lite_rps.h5` | 양자화 변환에 사용하는 원본 Keras 모델 |

## 1. 실습 자료 받기

블로그에서 ZIP 파일을 받아 압축을 푼다.

```bash
unzip RSP_SSD_LAB.zip
cd RSP_SSD_LAB
```

GitHub 원격 저장소가 별도로 제공되는 경우에는 같은 폴더 구조를 `git clone`으로 받아도 된다.

## 2. Raspberry Pi 준비

Raspberry Pi에서 필요한 패키지는 다음이다.

```bash
python3 -m venv ~/camera_test/.venv311
~/camera_test/.venv311/bin/pip install -r requirements-pi.txt
```

이미 수업용 이미지에 `~/camera_test/.venv311` 환경이 준비되어 있으면 새로 만들 필요 없다. 아래 명령으로 확인한다.

```bash
~/camera_test/.venv311/bin/python - <<'PY'
import importlib.util
for m in ["cv2", "numpy", "tflite_runtime"]:
    print(m, bool(importlib.util.find_spec(m)))
PY
```

모두 `True`면 준비된 상태다.

## 3. Raspberry Pi로 파일 전송

MacBook에서 실행한다. IP와 사용자 이름은 본인 Raspberry Pi에 맞게 바꾼다.

```bash
PI_USER=philip
PI_IP=172.24.133.248
PI_DIR=/home/$PI_USER/camera_test/RSP_SSD_LAB

ssh $PI_USER@$PI_IP "mkdir -p $PI_DIR"
scp -r ./* $PI_USER@$PI_IP:$PI_DIR/
```

## 4. SSD 오브젝트 디텍션 실행

Raspberry Pi에 모니터가 연결되어 있으면 Pi 터미널에서 실행한다.

```bash
cd ~/camera_test/RSP_SSD_LAB
~/camera_test/.venv311/bin/python scripts/run_rps_ssd_camera.py
```

실행되면 카메라 영상 위에 `SCISSORS`, `ROCK`, `PAPER` 바운딩 박스가 표시된다. 종료는 OpenCV 창을 선택한 뒤 `q`를 누른다.

SSH에서 Raspberry Pi 화면으로 띄우려면 다음처럼 실행한다.

```bash
cd ~/camera_test/RSP_SSD_LAB
nohup env DISPLAY=:0 XAUTHORITY=/home/philip/.Xauthority \
  ~/camera_test/.venv311/bin/python scripts/run_rps_ssd_camera.py \
  > rps_ssd_camera.log 2>&1 &
```

프로세스 확인과 종료:

```bash
pgrep -af run_rps_ssd_camera.py
pkill -f run_rps_ssd_camera.py
```

## 5. MacBook에서 양자화 모델 생성

MacBook에서 실행한다.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-mac.txt
python scripts/quantize_rps_ssd_lite.py
```

생성 또는 갱신되는 파일은 다음이다.

```text
models/rps_ssd_lite_fp32.tflite
models/rps_ssd_lite_int8.tflite
models/ssd_lite_rps_saved_model/
```

## 6. Raspberry Pi에서 양자화 전후 비교

Raspberry Pi에서 실행한다.

```bash
cd ~/camera_test/RSP_SSD_LAB
~/camera_test/.venv311/bin/python scripts/benchmark_rps_ssd_lite_tflite.py
```

실제 측정 예시는 다음과 같다.

| 항목 | FP32 SSD-Lite | INT8 SSD-Lite |
|---|---:|---:|
| 파일명 | `rps_ssd_lite_fp32.tflite` | `rps_ssd_lite_int8.tflite` |
| 파일 크기 | 115,876 bytes | 36,768 bytes |
| 입력 dtype | `float32` | `int8` |
| 출력 dtype | `float32` | `int8` |
| 평균 추론 시간 | 1.89 ms | 1.39 ms |
| `ps` CPU 사용률 | 172.0% | 148.0% |
| `ps` RSS 메모리 | 42.17 MB | 41.97 MB |

요약하면 모델 크기는 68.3% 감소했고, 평균 추론 속도는 약 1.36배 향상되었다.

## RSS 메모리가 거의 같은 이유

RSS는 Resident Set Size의 약자이며, 현재 프로세스가 실제 RAM에 올라가 사용 중인 메모리 크기다.

```bash
ps -p 프로세스ID -o pid,%cpu,rss,comm
```

`rss` 단위는 KB다. 예를 들어 `43184 KB`는 약 `42.17 MB`다.

이번 실습에서는 모델 파일이 68.3% 줄었지만 RSS는 거의 같았다. 이유는 모델 자체가 수십 KB에서 100KB 수준으로 매우 작고, Python, OpenCV, NumPy, TFLite runtime이 차지하는 기본 메모리가 수십 MB 수준이기 때문이다.

```text
전체 RSS = 실행 환경 기본 메모리 + 모델 메모리 + 입출력/연산 버퍼
```

따라서 경량화 비교에서는 RSS만 보지 말고 모델 크기, 추론 시간, FPS, 정확도를 함께 봐야 한다.

## 자세한 매뉴얼

전체 절차와 문제 해결은 ZIP 안의 아래 문서를 참고한다.

```text
docs/라즈베리파이_SSD_오브젝트디텍션_실행매뉴얼.md
```

## 문제 해결

카메라가 안 보이면 장치 파일을 먼저 확인한다.

```bash
ls -l /dev/video*
```

다른 프로세스가 카메라를 쓰고 있으면 다음 명령으로 확인하고 종료한다.

```bash
pgrep -af 'run_rps_ssd_camera.py|EX_03_Board_RPS_PreTrained_SSD.py'
pkill -f run_rps_ssd_camera.py
```

모델 파일을 못 찾으면 repo 루트에서 실행했는지 확인한다.

```bash
pwd
ls models
```
