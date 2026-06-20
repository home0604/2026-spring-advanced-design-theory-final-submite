# 00.김임환교수.최종과제.SSD.on.RSP.Project

## 제출자

- 김임환 교수
- GitHub 계정: `philipdekim-OnD01`

## 최종 발표 자료

아래 링크에서 최종 발표/실습 자료를 볼 수 있습니다.

```text
https://philipdekim-ond01.github.io/obsidian-blog/rsp-ssd-lab-raspberry-pi-object-detection.html
```

## 프로젝트 제목

SSD on Raspberry Pi Object Detection Project

## 프로젝트 요약

Raspberry Pi에서 가위바위보 손 모양을 SSD Object Detection 방식으로 실행하는 실습 프로젝트입니다. TFLite 모델을 사용해 실시간 카메라 추론을 수행하고, float32 모델과 int8 양자화 모델의 크기, 추론 시간, CPU 사용량, RAM 사용량을 비교합니다.

## 업로드한 자료

```text
00.김임환교수.최종과제.SSD.on.RSP.Project/
|-- README.md
|-- project-page/
|   |-- rsp-ssd-lab-raspberry-pi-object-detection.html
|   `-- rsp-ssd-lab-raspberry-pi-object-detection.md
`-- RSP_SSD_LAB/
    |-- README.md
    |-- requirements-mac.txt
    |-- requirements-pi.txt
    |-- assets/
    |-- docs/
    |-- models/
    `-- scripts/
```

## 주요 내용

- Raspberry Pi 실시간 카메라 추론
- SSD 기반 TFLite Object Detection
- 가위, 바위, 보 손 모양 인식
- float32 TFLite 모델과 int8 TFLite 모델 비교
- 모델 크기, 추론 속도, CPU 사용량, 메모리 사용량 벤치마크

## 실행 자료 위치

실습 코드와 모델 파일은 아래 폴더에 있습니다.

```text
RSP_SSD_LAB/
```

자세한 실행 방법은 다음 파일을 확인하세요.

```text
RSP_SSD_LAB/README.md
RSP_SSD_LAB/docs/라즈베리파이_SSD_오브젝트디텍션_실행매뉴얼.md
```
