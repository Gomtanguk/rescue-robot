# rescue-robot-workspace

구조 로봇 시나리오를 위한 ROS 2 워크스페이스입니다. 카메라 기반 감지, robot5 사람 탐색, robot6 구조 미션 제어, 음성 입출력, 웹 UI를 하나의 워크스페이스에서 다룹니다.

## Quick Summary

- Domain: 다중 로봇 구조 시나리오 워크스페이스
- Stack: ROS 2 Humble, Python, YOLO, Whisper, Flask
- Key Packages: `camera_system`, `robot5_person_search`, `rescue_bot`

## Project Goal

- 카메라 감지, 탐색 로봇, 구조 로봇, 음성 인터랙션, 웹 UI를 하나의 ROS 2 워크스페이스로 통합합니다.
- “사람을 찾는 단계”와 “구조하러 가는 단계”를 분리해 멀티로봇 흐름을 명확하게 설계합니다.
- 실제 시연과 시나리오 테스트에 맞춰 런타임 계약과 패키지 경계를 정리합니다.

## Problem Statement

- 구조 시나리오는 비전 감지, 위치 계산, 내비게이션, 음성 대화, UI 표시가 서로 강하게 연결됩니다.
- 한 노드에 기능을 몰아넣으면 테스트와 수정이 어려워지고, 시나리오 흐름도 추적하기 힘들어집니다.
- 이 프로젝트는 `camera_system`, `robot5_person_search`, `rescue_bot`으로 역할을 분리해 이벤트 중심 구조로 통합합니다.

## Key Features

- 카메라 기반 객체 탐지와 붕괴 감지
- robot5 사람 탐색 이벤트 발행
- robot6 구조 내비게이션과 세션 제어
- STT/TTS 기반 구조 대화
- Flask 기반 웹 UI 상태 시각화
- 워크스페이스 단위 빌드와 런타임 계약 문서화

## Repository Structure

```text
rescue-robot-workspace/
├── src/
│   ├── camera_system/
│   ├── rescue_bot/
│   └── robot5_person_search/
├── docs/
├── requirements.txt
├── ros_source.bash
└── README.md
```

## Main Packages

- `camera_system`
  - 카메라 입력, 탐지, 오버레이, 붕괴 감지
- `robot5_person_search`
  - robot5 탐색 중 사람 검출 이벤트 발행
- `rescue_bot`
  - 구조 제어, 내비게이션, 음성 상호작용, 웹 UI

## Workspace Boundary

- 이 저장소는 ROS 2 워크스페이스 루트를 기준으로 관리합니다.
- Git에 포함하는 대상은 `src/` 아래 패키지 코드, 런치 파일, 패키지 메타데이터, 문서입니다.
- 모델 가중치, 데이터베이스, 음성 파일, zip 산출물은 기본적으로 로컬 런타임 자산으로 취급하며 Git 추적에서 제외합니다.
- 실험 문서나 검토 자료는 필요 시 `docs/` 또는 패키지별 `docs/`에 정리합니다.

## System Flow

1. `camera_system`이 카메라 스트림을 처리하고 감지 결과를 생성합니다.
2. `robot5_person_search`가 사람 후보를 발견하면 구조 이벤트를 발행합니다.
3. `rescue_nav_node`가 robot6의 목표 위치와 접근 방향을 계산합니다.
4. 도착 후 `rescue_control_node`가 구조 세션을 시작합니다.
5. `rescue_stt_node`가 피해자와의 음성 상호작용을 담당합니다.
6. `rescue_ui`가 웹 기반 상태 표시와 이벤트 시각화를 제공합니다.

## Technical Highlights

### 1. Camera System

- 카메라 스트림을 받아 객체 탐지와 후처리를 수행합니다.
- 감지 결과를 오버레이 노드가 시각화 가능한 형태로 가공합니다.
- 필요 시 자세나 장면 변화를 바탕으로 붕괴 이벤트를 감지합니다.

### 2. Robot5 Person Search

- 탐색 중인 robot5가 사람 후보를 검출하면 이벤트를 발행합니다.
- 검출 위치, 로봇 자세, 피해자 방향 정보를 후속 구조 시스템이 사용할 수 있는 토픽으로 정리합니다.
- 이 단계는 “사람을 찾는 문제”와 “구조하러 가는 문제”를 분리하는 전처리 역할을 합니다.

### 3. Rescue Navigation

- `rescue_nav_node`는 robot5가 제공한 위치 정보를 바탕으로 robot6의 목표 지점을 계산합니다.
- 목표까지의 이동, 도착 판정, 다음 단계 전이를 내비게이션 상태기계로 관리합니다.
- 도착 후에는 구조 세션 시작 이벤트를 상위 제어 노드에 전달합니다.

### 4. Rescue Session Control

- `rescue_control_node`는 구조 시나리오 전체의 오케스트레이터 역할을 합니다.
- 비전 분석 결과, 도착 상태, 음성 대화 완료 상태를 합쳐 현재 구조 세션의 단계를 결정합니다.
- 구조 절차를 단순 노드 집합이 아니라 상태 기반 흐름으로 연결합니다.

### 5. Voice Interaction

- `rescue_stt_node`는 STT와 TTS를 사용해 구조 대화를 수행합니다.
- 구조 대상자의 응답을 받아 추가 판단에 필요한 입력으로 넘깁니다.
- 음성 흐름 종료 신호는 다음 내비게이션 또는 세션 종료 판단에 사용됩니다.

### 6. Web UI and Logging

- `rescue_ui`는 현재 세션 상태와 이벤트를 웹 화면으로 제공합니다.
- 전체 시스템은 이벤트 중심으로 연결되어 있어, 감지, 이동, 구조, 종료까지의 흐름을 순차적으로 추적할 수 있습니다.

## Build

```bash
source /opt/ros/humble/setup.bash
cd <workspace-root>
python3 -m pip install -r requirements.txt
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

## Run

```bash
ros2 launch camera_system camera_system.launch.py
ros2 launch robot5_person_search robot5_person_search.launch.py
ros2 launch rescue_bot rescue_real.launch.py
```

웹 중심 런치:

```bash
ros2 launch rescue_bot rescue_web.launch.py
```

## Environment

- Ubuntu 22.04
- ROS 2 Humble
- Python 3.10 계열
- `colcon`, `rosdep`
- YOLO/Whisper 추론이 가능하면 GPU 환경 권장

## Portfolio Notes

- 통합 시나리오 테스트 절차는 `docs/dry_test_manual.md`에 정리합니다.
- 모델 파일은 패키지 내부 또는 로컬 자산 경로에서 관리하고, Git에는 기본적으로 포함하지 않습니다.
- 데모 영상은 저장소 밖 링크나 별도 배포 채널로 공유하는 것을 권장합니다.
- `colcon`이 없는 환경에서는 전체 워크스페이스 빌드를 직접 검증할 수 없습니다.
