# rescue-robot-workspace

구조 로봇 시나리오를 위한 ROS 2 워크스페이스입니다. 카메라 기반 감지, robot5 사람 탐색, robot6 구조 미션 제어, 음성 입출력, 웹 UI를 하나의 워크스페이스에서 다룹니다.

## 구조

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

## 패키지

- `camera_system`
  - 카메라 입력, 탐지, 오버레이, 붕괴 감지
- `robot5_person_search`
  - robot5 탐색 중 사람 검출 이벤트 발행
- `rescue_bot`
  - 구조 제어, 내비게이션, 음성 상호작용, 웹 UI

## 권장 환경

- Ubuntu 22.04
- ROS 2 Humble
- Python 3.10 계열
- `colcon`, `rosdep`
- YOLO/Whisper 추론이 가능하면 GPU 환경 권장

## 저장소 경계

- 이 저장소는 ROS 2 워크스페이스 루트를 기준으로 관리합니다.
- Git에 포함하는 대상은 `src/` 아래 패키지 코드, 런치 파일, 패키지 메타데이터, 문서입니다.
- 모델 가중치, 데이터베이스, 음성 파일, zip 산출물은 기본적으로 로컬 런타임 자산으로 취급하며 Git 추적에서 제외합니다.
- 실험 문서나 검토 자료는 필요 시 `docs/` 또는 패키지별 `docs/`에 정리합니다.

## 워크스페이스 빌드

```bash
source /opt/ros/humble/setup.bash
cd <workspace-root>
python3 -m pip install -r requirements.txt
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

## 실행 예시

```bash
ros2 launch camera_system camera_system.launch.py
ros2 launch robot5_person_search robot5_person_search.launch.py
ros2 launch rescue_bot rescue_real.launch.py
```

웹 중심 런치:

```bash
ros2 launch rescue_bot rescue_web.launch.py
```

## 동작 흐름

1. `robot5_person_search`가 사람 검출 이벤트를 발행합니다.
2. `rescue_nav_node`가 검출 위치를 바탕으로 이동합니다.
3. 도착 후 `rescue_control_node`가 구조 세션을 시작합니다.
4. `rescue_stt_node`가 음성 상호작용을 처리합니다.
5. `rescue_ui`가 웹 기반 상태 표시를 제공합니다.

## 알고리즘 개요

### 1. camera_system

- 카메라 스트림을 받아 객체 탐지와 후처리를 수행합니다.
- 감지 결과를 오버레이 노드가 시각화 가능한 형태로 가공합니다.
- 필요 시 자세나 장면 변화를 바탕으로 붕괴 이벤트를 감지합니다.

### 2. robot5_person_search

- 탐색 중인 robot5가 사람 후보를 검출하면 이벤트를 발행합니다.
- 검출 위치, 로봇 자세, 피해자 방향 정보를 후속 구조 시스템이 사용할 수 있는 토픽으로 정리합니다.
- 이 단계는 “사람을 찾는 문제”와 “구조하러 가는 문제”를 분리하는 전처리 역할을 합니다.

### 3. rescue_bot 내비게이션

- `rescue_nav_node`는 robot5가 제공한 위치 정보를 바탕으로 robot6의 목표 지점을 계산합니다.
- 목표까지의 이동, 도착 판정, 다음 단계 전이를 내비게이션 상태기계로 관리합니다.
- 도착 후에는 구조 세션 시작 이벤트를 상위 제어 노드에 전달합니다.

### 4. 구조 세션 제어

- `rescue_control_node`는 구조 시나리오 전체의 오케스트레이터 역할을 합니다.
- 비전 분석 결과, 도착 상태, 음성 대화 완료 상태를 합쳐 현재 구조 세션의 단계를 결정합니다.
- 이 노드는 구조 절차를 단순 노드 집합이 아니라 상태 기반 흐름으로 연결합니다.

### 5. 음성 상호작용

- `rescue_stt_node`는 STT와 TTS를 사용해 구조 대화를 수행합니다.
- 구조 대상자의 응답을 받아 추가 판단에 필요한 입력으로 넘깁니다.
- 음성 흐름 종료 신호는 다음 내비게이션 또는 세션 종료 판단에 사용됩니다.

### 6. 웹 UI와 기록

- `rescue_ui`는 현재 세션 상태와 이벤트를 웹 화면으로 제공합니다.
- 전체 시스템은 이벤트 중심으로 연결되어 있어, 감지, 이동, 구조, 종료까지의 흐름을 순차적으로 추적할 수 있습니다.

## 참고

- 이 저장소는 기존 작업 이력이 포함된 상태라 변경분이 비교적 많습니다.
- 모델 파일은 패키지 내부 또는 로컬 자산 경로에서 관리하고, Git에는 기본적으로 포함하지 않습니다.
- 현재 정리 기준에서 핵심 패키지는 `camera_system`, `robot5_person_search`, `rescue_bot`입니다.
- 통합 시나리오 테스트 절차는 `docs/dry_test_manual.md`에 정리합니다.
- `colcon`이 없는 환경에서는 전체 워크스페이스 빌드를 직접 검증할 수 없습니다.
