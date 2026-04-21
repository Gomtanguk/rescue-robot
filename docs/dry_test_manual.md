# Rescue Bot Dry Test Manual

이 파일은 통합 시스템의 동작을 확인하기 위한 전체 테스트 시퀀스를 한곳에 모아둔 매뉴얼입니다.

## 1. 환경 설정 및 실행

새 터미널을 열 때마다 먼저 환경을 소스해야 합니다.

```bash
# 워크스페이스 루트에서 실행
source ros_source.bash

# 통합 런치 파일 실행 (Gazebo + RViz + Rescue Nodes)
ros2 launch rescue_bot rescue_sim.launch.py

# 웹 UI 통신 서버 (별도 터미널)
ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# 웹 영상 스트리밍 서버 (별도 터미널)
ros2 run web_video_server web_video_server
```

## 2. 테스트 시나리오: 3개 목표 주행 및 상황별 대응

로봇이 3곳의 목표를 순차적으로 방문하며, 각 지점에서 요구조자의 상태에 따라 다르게 반응하는 시나리오입니다.

### Step 1. robot5 검출 결과를 nav 큐에 쌓기

현재 nav는 `/rescue/victim_pose`를 직접 받지 않고, 아래 두 토픽을 사용합니다.

- `/robot5/robot_pose_at_detection`
  - robot5가 검출 당시 서 있던 위치
  - robot6의 실제 주행 목표 위치
- `/robot5/victim_point`
  - 피해자 추정 위치
  - robot6 도착 yaw 계산용 보조 입력

아래 명령어들을 순서대로 실행하여 robot6에게 갈 곳을 알려줍니다.

```bash
# 목표 1 (지점 1) - Normal 상황
ros2 topic pub --once /robot5/victim_point geometry_msgs/msg/PointStamped "{header: {frame_id: 'map'}, point: {x: 3.0, y: 5.0, z: 0.0}}"
ros2 topic pub --once /robot5/robot_pose_at_detection geometry_msgs/msg/PoseStamped "{header: {frame_id: 'map'}, pose: {position: {x: 2.0, y: 5.0, z: 0.0}, orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}}}"

# 목표 2 (지점 2) - Caution 상황
ros2 topic pub --once /robot5/victim_point geometry_msgs/msg/PointStamped "{header: {frame_id: 'map'}, point: {x: 7.0, y: -1.0, z: 0.0}}"
ros2 topic pub --once /robot5/robot_pose_at_detection geometry_msgs/msg/PoseStamped "{header: {frame_id: 'map'}, pose: {position: {x: 6.0, y: -1.0, z: 0.0}, orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}}}"

# 목표 3 (지점 3) - Critical 상황
ros2 topic pub --once /robot5/victim_point geometry_msgs/msg/PointStamped "{header: {frame_id: 'map'}, point: {x: 1.0, y: -2.0, z: 0.0}}"
ros2 topic pub --once /robot5/robot_pose_at_detection geometry_msgs/msg/PoseStamped "{header: {frame_id: 'map'}, pose: {position: {x: 2.0, y: 2.0, z: 0.0}, orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}}}"
```

`victim_point`는 yaw 계산용이라 먼저 보내고, 바로 이어서 `robot_pose_at_detection`를 보내는 순서를 권장합니다.

### Step 2. 각 지점 도착 시 미션 트리거

로봇이 지점에 도착하여 대기 상태가 되면, 터미널에서 아래 명령어로 상황을 주입합니다.

```bash
# 상황 1: Normal
ros2 topic pub --once /robot6/tts/request std_msgs/String "{data: 'NORMAL'}"

# 상황 2: Caution
ros2 topic pub --once /robot6/tts/request std_msgs/String "{data: 'CAUTION'}"

# 상황 3: Critical
ros2 topic pub --once /robot6/tts/request std_msgs/String "{data: 'CRITICAL'}"
```

- `CAUTION`에서는 로봇의 질문에 마이크로 응답합니다.
- `CRITICAL`에서는 응답하지 않고 사이렌 동작을 확인합니다.

### Step 3. 최종 복귀

모든 미션이 완료되고 큐가 비면 로봇은 자동으로 Dock으로 복귀합니다.

### Step 4. 정지/타임아웃 제어 테스트

```bash
# 즉시 정지 + 세션 리셋 + 도킹 복귀
ros2 topic pub --once /robot/stop std_msgs/msg/Bool "{data: true}"

# 현재 단계 강제 스킵
ros2 topic pub --once /robot6/mission/timeout std_msgs/msg/Bool "{data: true}"
```

## 3. Gazebo 인물 모델 배치 예시

```bash
# actor_normal
ros2 run ros_gz_sim create -world warehouse -file https://fuel.gazebosim.org/1.0/OpenRobotics/models/Walking%20person -name actor_normal -x 3.0 -y 5.0 -z 0.0

# actor_caution
ros2 run ros_gz_sim create -world warehouse -file https://fuel.gazebosim.org/1.0/Mingfei/models/actor_sitting -name actor_caution -x 7.0 -y -1.0 -z 0.0

# actor_critical
ros2 run ros_gz_sim create -world warehouse -file https://fuel.gazebosim.org/1.0/OpenRobotics/models/Walking%20person -name actor_critical -x 1.0 -y -2.0 -z 0.1
```

Fuel 경로는 대소문자를 구분하므로 `OpenRobotics` 표기를 유지해야 합니다.

## 4. 모델 삭제 및 재배치

```bash
ros2 run ros_gz_sim remove -world warehouse -name actor_normal
ros2 run ros_gz_sim remove -world warehouse -name actor_caution
ros2 run ros_gz_sim remove -world warehouse -name actor_critical
```

## 5. 주요 모니터링 토픽

- 실시간 분석 화면: `/robot6/image_result`
- 현재 상태 로그: `/robot6/session/status`
- STT/TTS 완료 신호: `/robot6/tts/done`
- 비전 인식 결과 리포트: `/robot6/session/result`
- robot5 검출 기반 nav 입력: `/robot5/robot_pose_at_detection`
- robot5 victim 보조 좌표: `/robot5/victim_point`
- 긴급 정지/스킵 제어: `/robot/stop`, `/robot6/mission/timeout`

RQT에서 분석 영상을 확인하려면 아래 명령으로 시작할 수 있습니다.

```bash
rqt --clear-config
```

## 6. 맵 및 시각화

- 고정 프레임: `map`
- 맵 데이터: `/robot6/map`
- 맵 메타데이터: `/robot6/map_metadata`
- 현재 추정 위치: `/robot6/amcl_pose`
- 이동 경로: `/robot6/plan`
- 전역 지형도: `/robot6/global_costmap/costmap`
- 지역 지형도: `/robot6/local_costmap/costmap`
