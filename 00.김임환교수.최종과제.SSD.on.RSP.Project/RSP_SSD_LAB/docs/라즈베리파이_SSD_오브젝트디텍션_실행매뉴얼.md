# 라즈베리파이 SSD 오브젝트 디텍션 실행 매뉴얼

이 문서는 Raspberry Pi에서 `RPS_PreTrained_SSD.tflite` 모델을 이용해 가위바위보 손 모양을 오브젝트 디텍션 방식으로 인식하는 절차를 정리한 것이다.

기존 `EX_04_Board_RPS_PreTrained_DenseNet.py`는 이미지 전체를 보고 `rock / paper / scissors` 중 하나로 분류하는 코드다. 반면 이 매뉴얼의 SSD 모델은 화면 안에서 손 모양의 위치를 찾고, 바운딩 박스와 함께 `SCISSORS / ROCK / PAPER`를 표시한다.

## 0. `RPS_PreTrained_SSD.tflite` 모델 소개

`RPS_PreTrained_SSD.tflite`는 가위바위보 손 모양을 탐지하기 위해 미리 학습된 SSD 계열 오브젝트 디텍션 모델이다.

모델 이름의 의미는 다음과 같다.

- `RPS`: Rock, Paper, Scissors
- `PreTrained`: 수업 전에 미리 학습된 모델
- `SSD`: Single Shot MultiBox Detector 계열 오브젝트 디텍션 구조
- `TFLite`: TensorFlow Lite 형식으로 변환된 온디바이스 실행용 모델

이 모델은 이미지 전체를 하나의 클래스로 분류하는 CNN 분류 모델과 다르다. SSD 모델은 한 프레임 안에서 물체의 위치와 종류를 동시에 예측한다.

출력 예시는 다음과 같다.

```text
class: ROCK
score: 91%
box: xmin, ymin, xmax, ymax
```

즉, 결과는 단순히 `rock`이라고 끝나는 것이 아니라, 화면 어디에 손이 있는지 바운딩 박스로 함께 표시된다.

### 경량화 모델인가?

이 모델은 Raspberry Pi에서 실행할 수 있도록 `.tflite` 형식으로 변환된 온디바이스용 모델이다. 따라서 일반적인 데스크톱/서버용 TensorFlow 모델보다 배포와 실행이 가볍다.

다만 이번에 실행한 `RPS_PreTrained_SSD.tflite`는 입출력 텐서가 `float32`인 모델이다.

실제 확인된 입력 정보:

```text
input shape: [1, 320, 320, 3]
input dtype: float32
model size: 약 11MB
```

따라서 이 모델은 다음처럼 이해하는 것이 정확하다.

```text
TFLite 변환된 온디바이스 실행용 SSD 모델
```

하지만 다음처럼 표현하면 부정확하다.

```text
완전한 int8 양자화 최경량 SSD 모델
```

수업 자료에는 별도로 `ssd_rps_int8.tflite` 같은 int8 양자화 모델도 존재할 수 있다. 그런 모델은 가중치와 연산을 8비트 정수 중심으로 줄여 모델 크기와 연산량을 더 줄인 버전이다.

정리하면 다음과 같다.

| 모델 유형 | 의미 | 특징 |
|---|---|---|
| `RPS_PreTrained_SSD.tflite` | TFLite 변환 SSD 모델 | Raspberry Pi에서 실행 가능, float32 기반 |
| `ssd_rps_int8.tflite` | int8 양자화 SSD 모델 | 더 작고 빠를 수 있음, 정확도 변화 가능 |
| DenseNet 분류 모델 | 이미지 분류 모델 | 위치 탐지 없이 클래스만 예측 |

이번 매뉴얼에서는 먼저 안정적으로 실행 가능한 `RPS_PreTrained_SSD.tflite`를 사용한다. 이후 속도와 모델 크기를 비교하려면 int8 양자화 모델과 FPS, 모델 크기, 탐지 정확도를 비교하면 된다.

### 이번 실습에서 추가 생성한 양자화 모델

이번 실습에서는 수업 자료에 있는 작은 SSD-Lite 원본 모델을 이용해 양자화 전후 모델을 새로 생성했다.

원본 Keras 모델:

```text
models/ssd_lite_rps.h5
```

생성된 TFLite 모델:

```text
models/rps_ssd_lite_fp32.tflite
models/rps_ssd_lite_int8.tflite
```

주의할 점은, 이 비교용 모델은 `RPS_PreTrained_SSD.tflite`와 완전히 같은 파일을 양자화한 것이 아니라, 수업 자료의 `ssd_lite_rps.h5`에서 새로 변환한 SSD-Lite 모델이라는 점이다.

비교 목적은 다음이다.

```text
같은 SSD-Lite 구조에서 float32 TFLite와 int8 TFLite를 비교한다.
```

실제 생성 결과는 다음과 같다.

| 항목 | FP32 모델 | INT8 양자화 모델 |
|---|---:|---:|
| 파일명 | `rps_ssd_lite_fp32.tflite` | `rps_ssd_lite_int8.tflite` |
| 파일 크기 | 115,876 bytes | 36,768 bytes |
| 입력 크기 | `[1, 64, 64, 3]` | `[1, 64, 64, 3]` |
| 입력 dtype | `float32` | `int8` |
| 출력 크기 | `[1, 256, 7]` | `[1, 256, 7]` |
| 출력 dtype | `float32` | `int8` |
| Raspberry Pi 평균 추론 시간 | 1.89 ms | 1.39 ms |
| `ps` CPU 사용률 | 172.0% | 148.0% |
| `ps` RSS 메모리 | 42.17 MB | 41.97 MB |

비교 결과:

```text
모델 크기 감소: 약 68.3%
평균 추론 속도 향상: 약 1.36배
```

따라서 양자화의 효과는 다음처럼 정리할 수 있다.

- 모델 파일 크기가 크게 줄어든다.
- Raspberry Pi CPU에서 추론 시간이 줄어든다.
- `ps` 기준 RSS 메모리는 두 모델 모두 약 42MB 수준으로 큰 차이가 없었다.
- `ps`의 CPU 사용률은 멀티스레드 실행 시 100%를 넘을 수 있다.
- 단, int8 모델은 입력과 출력도 정수 양자화 값이므로 전처리와 후처리에서 scale, zero point 처리가 필요하다.
- 정확도는 별도 검증 데이터셋으로 비교해야 한다. 크기와 속도가 좋아져도 탐지 정확도가 유지되는지 확인해야 한다.

### RSS 메모리란?

`ps`에서 확인하는 RSS는 Resident Set Size의 약자다. 현재 프로세스가 실제 RAM에 올려 사용 중인 메모리 크기를 의미한다.

확인 명령은 다음과 같다.

```bash
ps -p 프로세스ID -o pid,%cpu,rss,comm
```

여기서 `rss` 값의 단위는 KB다.

예를 들어 `rss=43184`라면 다음과 같이 해석한다.

```text
43184 KB / 1024 = 약 42.17 MB
```

RSS는 모델 파일 크기만 의미하지 않는다. Python으로 TFLite 모델을 실행하면 다음 메모리가 모두 합쳐져 RSS로 보인다.

- Python 인터프리터 자체 메모리
- NumPy 배열 메모리
- OpenCV 로딩 메모리
- TFLite runtime과 interpreter 메모리
- 모델 가중치 메모리
- 입력/출력 텐서 버퍼
- 임시 연산 버퍼
- 공유 라이브러리 로딩 메모리

따라서 모델 파일 크기가 줄어도 RSS가 같은 수준으로 보일 수 있다.

이번 결과가 그 예다.

```text
FP32 모델 파일 크기: 115,876 bytes
INT8 모델 파일 크기: 36,768 bytes

FP32 RSS: 약 42.17 MB
INT8 RSS: 약 41.97 MB
```

모델 파일은 68.3% 줄었지만 RSS는 약 0.2MB 정도만 줄었다. 이유는 이 실습 모델이 매우 작기 때문이다. 모델 자체는 수십 KB에서 100KB 수준인데, Python, OpenCV, NumPy, TFLite runtime이 차지하는 기본 메모리는 수십 MB 수준이다.

즉, 이 실습에서는 전체 RSS를 다음처럼 이해해야 한다.

```text
전체 RSS = 실행 환경 기본 메모리 + 모델 메모리 + 입출력/연산 버퍼
```

작은 모델에서는 실행 환경 기본 메모리가 대부분을 차지한다. 그래서 모델을 양자화해도 RSS 차이가 작게 보인다.

하지만 파일 크기와 추론 시간은 실제로 개선되었다.

```text
파일 크기: 115,876 bytes → 36,768 bytes
추론 시간: 1.89 ms → 1.39 ms
```

정리하면 다음과 같다.

| 비교 항목 | 양자화 효과가 잘 보이는가? | 이유 |
|---|---|---|
| 모델 파일 크기 | 잘 보임 | 가중치가 int8로 줄어듦 |
| 추론 시간 | 어느 정도 보임 | 정수 연산과 작은 모델 구조의 영향 |
| RSS 메모리 | 작게 보임 | Python/OpenCV/TFLite 기본 메모리가 더 큼 |

그래서 경량화 비교에서는 RSS만 보면 안 되고, 모델 크기, 추론 시간, FPS, 정확도를 함께 봐야 한다.

## 1. 목표

- MacBook에 있는 SSD TFLite 모델과 실행 코드를 Raspberry Pi로 전송한다.
- Raspberry Pi에서 모델이 정상 로딩되는지 확인한다.
- USB 카메라를 이용해 실시간 오브젝트 디텍션을 실행한다.
- 실행 중인 프로그램을 안전하게 종료한다.

## 2. 준비물

- Raspberry Pi
- USB 카메라
- Raspberry Pi와 같은 네트워크에 연결된 MacBook
- Raspberry Pi SSH 접속 정보
- 수업 자료에 포함된 두 파일
  - `RPS_PreTrained_SSD.tflite`
  - `EX_03_Board_RPS_PreTrained_SSD.py`

이번 테스트에서 사용한 예시 정보는 다음과 같다.

```text
Raspberry Pi IP: 172.24.133.248
Raspberry Pi user: philip
Raspberry Pi 작업 폴더:
/home/philip/camera_test/cnn/examples/05_Object_Detection_Based_On-Device_AI/
```

각자 실습 환경에서는 IP 주소와 사용자 이름이 다를 수 있다.

## 3. Raspberry Pi 연결 확인

MacBook 터미널에서 Raspberry Pi의 SSH 포트가 열려 있는지 확인한다.

```bash
nc -vz 172.24.133.248 22
```

정상 예시는 다음과 같다.

```text
Connection to 172.24.133.248 port 22 [tcp/ssh] succeeded!
```

SSH로 접속한다.

```bash
ssh philip@172.24.133.248
```

접속 후 Wi-Fi와 IP 상태를 확인한다.

```bash
nmcli device status
ip -4 addr show wlan0
```

`wlan0`가 `Yonsei`에 연결되어 있고 IP 주소가 보이면 정상이다.

## 4. Raspberry Pi에 작업 폴더 만들기

Raspberry Pi에 SSH로 접속한 상태에서 아래 명령을 실행한다.

```bash
mkdir -p ~/camera_test/RSP_SSD_LAB
```

기존 DenseNet 분류 코드가 실행 중이면 카메라를 점유할 수 있으므로 종료한다.

```bash
pkill -f EX_04_Board_RPS_PreTrained_DenseNet.py || true
```

종료 확인:

```bash
pgrep -af 'EX_04_Board_RPS_PreTrained_DenseNet.py' || true
```

아무것도 출력되지 않으면 종료된 것이다.

## 5. MacBook에서 repo 전송

MacBook 터미널에서 repo 폴더로 이동한다.

```bash
cd RSP_SSD_LAB
```

Raspberry Pi로 repo 전체를 전송한다.

```bash
scp -r ./* philip@172.24.133.248:/home/philip/camera_test/RSP_SSD_LAB/
```

비밀번호를 입력하면 전송된다.

전송되는 파일 크기 예시는 다음과 같다.

```text
models/RPS_PreTrained_SSD.tflite 약 11MB
scripts/run_rps_ssd_camera.py    약 4KB
```

## 6. Raspberry Pi에서 파일 확인

Raspberry Pi에 SSH로 접속한 뒤 작업 폴더로 이동한다.

```bash
cd ~/camera_test/cnn/examples/05_Object_Detection_Based_On-Device_AI
ls -lh
```

정상 예시는 다음과 같다.

```text
EX_03_Board_RPS_PreTrained_SSD.py
RPS_PreTrained_SSD.tflite
```

## 7. SSD 모델 로딩 확인

실행 전에 TFLite 모델이 정상적으로 열리는지 확인한다.

```bash
~/camera_test/.venv311/bin/python - <<'PY'
import tflite_runtime.interpreter as tflite

interpreter = tflite.Interpreter(model_path="RPS_PreTrained_SSD.tflite")
interpreter.allocate_tensors()

print("input", interpreter.get_input_details())
print("outputs", [
    (o["index"], o["shape"].tolist(), str(o["dtype"]))
    for o in interpreter.get_output_details()
])
PY
```

정상 출력 예시는 다음과 같다.

```text
INFO: Created TensorFlow Lite XNNPACK delegate for CPU.
input [{'name': 'serving_default_input:0', 'shape': array([1, 320, 320, 3], dtype=int32), ...}]
outputs [(338, [1, 10], ...), (336, [1, 10, 4], ...), (339, [1], ...), (337, [1, 10], ...)]
```

입력 크기가 `[1, 320, 320, 3]`이면 정상이다.

## 8. USB 카메라 확인

카메라 장치가 보이는지 확인한다.

```bash
ls -l /dev/video*
```

`/dev/video0`가 있어야 한다.

카메라 프레임을 한 장 읽어본다.

```bash
~/camera_test/.venv311/bin/python - <<'PY'
import cv2

cap = cv2.VideoCapture(0)
print("opened", cap.isOpened())

ret, frame = cap.read()
print("read", ret, None if frame is None else frame.shape)

cap.release()
PY
```

정상 예시는 다음과 같다.

```text
opened True
read True (480, 640, 3)
```

## 9. Raspberry Pi 화면에서 직접 실행

Raspberry Pi에 모니터와 키보드가 연결되어 있다면 Pi 터미널에서 직접 실행한다.

```bash
cd ~/camera_test/cnn/examples/05_Object_Detection_Based_On-Device_AI
~/camera_test/.venv311/bin/python EX_03_Board_RPS_PreTrained_SSD.py
```

실행되면 OpenCV 창이 뜨고 카메라 영상 위에 바운딩 박스가 표시된다.

탐지 클래스:

```text
SCISSORS
ROCK
PAPER
```

종료하려면 OpenCV 창을 선택한 뒤 `q` 키를 누른다.

## 10. SSH에서 Raspberry Pi 화면으로 실행

MacBook에서 SSH로 접속한 상태에서 Raspberry Pi의 로컬 화면에 창을 띄우려면 `DISPLAY=:0`을 지정한다.

```bash
cd ~/camera_test/cnn/examples/05_Object_Detection_Based_On-Device_AI

nohup env DISPLAY=:0 XAUTHORITY=/home/philip/.Xauthority \
  ~/camera_test/.venv311/bin/python EX_03_Board_RPS_PreTrained_SSD.py \
  > ~/camera_test/rps_ssd_camera.log 2>&1 &
```

실행 프로세스를 확인한다.

```bash
pgrep -af 'EX_03_Board_RPS_PreTrained_SSD.py'
```

정상 예시는 다음과 같다.

```text
3812 /home/philip/camera_test/.venv311/bin/python EX_03_Board_RPS_PreTrained_SSD.py
```

로그를 확인한다.

```bash
tail -n 40 ~/camera_test/rps_ssd_camera.log
```

정상 실행 중이면 다음과 같은 로그가 보일 수 있다.

```text
INFO: Created TensorFlow Lite XNNPACK delegate for CPU.
QFontDatabase: Cannot find font directory ...
```

`QFontDatabase` 메시지는 OpenCV/Qt 글꼴 경고이며, 카메라 창이 정상적으로 뜬다면 무시해도 된다.

## 11. 실행 중인 SSD 오브젝트 디텍션 종료

OpenCV 창에서 종료할 수 있으면 `q` 키를 누른다.

SSH에서 강제로 종료하려면 다음 명령을 사용한다.

```bash
pkill -f EX_03_Board_RPS_PreTrained_SSD.py
```

종료 확인:

```bash
pgrep -af 'EX_03_Board_RPS_PreTrained_SSD.py' || true
```

아무것도 출력되지 않으면 종료된 것이다.

## 12. DenseNet 분류 코드와 SSD 탐지 코드의 차이

DenseNet 분류 코드:

```text
EX_04_Board_RPS_PreTrained_DenseNet.py
```

- 이미지 전체를 하나의 클래스로 분류한다.
- 출력은 `rock`, `paper`, `scissors` 중 하나다.
- 물체 위치는 표시하지 않는다.

SSD 오브젝트 디텍션 코드:

```text
EX_03_Board_RPS_PreTrained_SSD.py
```

- 화면 안에서 손 모양 위치를 찾는다.
- 바운딩 박스를 그린다.
- 박스 위에 `SCISSORS / ROCK / PAPER`와 confidence를 표시한다.

## 13. 문제 해결

### 카메라 창이 안 뜨는 경우

확인할 항목:

- Raspberry Pi에 모니터가 연결되어 있는가?
- 데스크톱 화면이 켜져 있는가?
- SSH에서 실행했다면 `DISPLAY=:0`을 지정했는가?

확인:

```bash
ls -l /tmp/.X11-unix
```

`X0`가 보이면 로컬 화면 `:0`이 열려 있는 것이다.

### 모델 파일을 못 찾는 경우

스크립트는 현재 폴더에서 `RPS_PreTrained_SSD.tflite`를 찾는다. 반드시 코드와 모델이 있는 폴더로 이동한 뒤 실행한다.

올바른 실행:

```bash
cd ~/camera_test/cnn/examples/05_Object_Detection_Based_On-Device_AI
~/camera_test/.venv311/bin/python EX_03_Board_RPS_PreTrained_SSD.py
```

### 카메라가 열리지 않는 경우

다른 프로그램이 카메라를 사용 중일 수 있다.

```bash
pgrep -af 'EX_04_Board_RPS_PreTrained_DenseNet.py|EX_03_Board_RPS_PreTrained_SSD.py'
```

필요하면 기존 프로세스를 종료한다.

```bash
pkill -f EX_04_Board_RPS_PreTrained_DenseNet.py
pkill -f EX_03_Board_RPS_PreTrained_SSD.py
```

카메라 장치 확인:

```bash
ls -l /dev/video*
```

### 탐지가 잘 안 되는 경우

확인할 항목:

- 손 모양이 화면 중앙에 잘 들어오는가?
- 배경이 너무 복잡하지 않은가?
- 조명이 너무 어둡지 않은가?
- 손이 카메라에서 너무 가깝거나 멀지 않은가?

이 예제의 confidence threshold는 코드 안에서 다음 값으로 설정되어 있다.

```python
threshold = 0.8
```

탐지가 너무 적게 뜨면 실습 목적에 따라 `0.6` 또는 `0.7` 정도로 낮춰볼 수 있다.

## 14. 전체 체크리스트

- Raspberry Pi가 Wi-Fi에 연결되어 있는가?
- SSH 접속이 되는가?
- SSD 모델과 실행 코드가 Pi에 전송되었는가?
- `RPS_PreTrained_SSD.tflite`가 모델 로딩 테스트를 통과했는가?
- `/dev/video0` 카메라가 보이는가?
- OpenCV 창이 Raspberry Pi 화면에 뜨는가?
- 손 모양에 바운딩 박스가 표시되는가?
- `q` 또는 `pkill`로 정상 종료되는가?

## 15. 양자화 모델 생성 코드

MacBook에서 양자화 모델을 생성하기 위해 아래 스크립트를 사용한다.

파일 위치:

```text
scripts/quantize_rps_ssd_lite.py
```

실행:

```bash
python3 scripts/quantize_rps_ssd_lite.py
```

이 스크립트가 하는 일:

1. `ssd_lite_rps.h5` Keras 모델을 불러온다.
2. Keras 3 호환성을 위해 SavedModel로 export한다.
3. float32 TFLite 모델을 생성한다.
4. 대표 이미지 데이터셋을 이용해 int8 양자화 TFLite 모델을 생성한다.
5. 두 모델의 입력/출력 dtype과 파일 크기를 출력한다.

대표 데이터셋은 다음 폴더의 RPS 이미지를 사용한다.

```text
assets/representative_images/
```

대표 데이터셋은 양자화 과정에서 activation 범위를 추정하는 데 사용된다. 즉, 모델이 실제로 볼 만한 입력 데이터를 몇 장 넣어주어 float 값을 int8 범위로 어떻게 매핑할지 정하는 과정이다.

생성 결과 폴더:

```text
models/
```

생성 파일:

```text
rps_ssd_lite_fp32.tflite
rps_ssd_lite_int8.tflite
ssd_lite_rps_saved_model/
```

## 16. 양자화 전후 벤치마크 코드

Raspberry Pi에서 양자화 전후 추론 시간을 비교하기 위해 아래 스크립트를 사용한다.
이 스크립트는 `ps -p 현재PID -o %cpu=,rss=`로 현재 Python 프로세스의 CPU 사용률과 RSS 메모리도 함께 기록한다.

파일 위치:

```text
scripts/benchmark_rps_ssd_lite_tflite.py
```

MacBook에서 Raspberry Pi로 전송:

```bash
scp models/rps_ssd_lite_fp32.tflite \
    models/rps_ssd_lite_int8.tflite \
    scripts/benchmark_rps_ssd_lite_tflite.py \
    philip@172.24.133.248:/home/philip/camera_test/RSP_SSD_LAB/
```

Raspberry Pi에서 실행:

```bash
cd /home/philip/camera_test/RSP_SSD_LAB
/home/philip/camera_test/.venv311/bin/python scripts/benchmark_rps_ssd_lite_tflite.py
```

실제 측정 결과:

```text
rps_ssd_lite_fp32.tflite
- size: 115,876 bytes
- input dtype: float32
- output dtype: float32
- mean inference time: 1.89 ms
- ps CPU: 172.0%
- ps RSS: 42.17 MB

rps_ssd_lite_int8.tflite
- size: 36,768 bytes
- input dtype: int8
- output dtype: int8
- mean inference time: 1.39 ms
- ps CPU: 148.0%
- ps RSS: 41.97 MB

size reduction: 68.3%
mean speedup: 1.36x
```

해석:

- 이 결과는 Raspberry Pi에서 같은 입력 크기 `[1, 64, 64, 3]`의 SSD-Lite 모델을 기준으로 측정한 것이다.
- int8 양자화는 모델 크기를 크게 줄였다.
- 속도도 개선되었지만, 실제 카메라 FPS는 전처리, 후처리, OpenCV 화면 출력 시간까지 포함되므로 순수 모델 추론 시간과 다를 수 있다.
- RSS 메모리는 모델 파일 크기만큼 선형적으로 줄지 않았다. Python, OpenCV, TFLite interpreter가 차지하는 기본 메모리가 함께 포함되기 때문이다.
- 최종 실습에서는 모델 크기, 추론 시간, 실제 탐지 안정성을 함께 비교해야 한다.
