# Ver12 - 통합 카메라 시스템

완벽하게 정리된 4-노드 ROS2 카메라 감지 및 붕괴 감지 시스템

## 🎯 핵심 입출력 (User Interface)

### 📤 출력 (발행 토픽)
| 용도 | 토픽 | 타입 | 설명 |
|------|------|------|------|
| **최종영상 출력** | `/output/cam01` | CompressedImage | cam01 감지 결과 시각화 이미지 |
| | `/output/cam02` | CompressedImage | cam02 감지 결과 시각화 이미지 |
| **사람 명수 카운트** | `/detection/cam01/person` | String (JSON) | cam01 사람 감지 결과 + 인원 수 |
| | `/detection/cam02/person` | String (JSON) | cam02 사람 감지 결과 + 인원 수 |
| **붕괴감지 출력** | `/alert/cam01/collapse` | Bool | cam01 붕괴 감지 여부 (True=감지됨) |
| | `/alert/cam02/collapse` | Bool | cam02 붕괴 감지 여부 (True=감지됨) |

### 📥 입력 (구독 토픽)
| 용도 | 토픽 | 타입 | 명령어 |
|------|------|------|--------|
| **사용자 UI에서 초기화** | `/control/alert/reset` | String | `ros2 topic pub /control/alert/reset std_msgs/String "{data: 'reset'}"` |

---

## 🚀 빠른 시작

### 설치
```bash
cd /home/rokey/test_code/cam_pjt/ver12
pip install -r requirements.txt
```

### 실행 (한 번에)
```bash
./run_all.sh
```

### 실행 (개별 터미널)
```bash
# 터미널 1
python3 1_camera_publisher.py

# 터미널 2
python3 2_detection_node.py

# 터미널 3
python3 3_overlay_node.py

# 터미널 4
python3 4_collapse_detector.py
```

---

## 📊 시스템 구조

### 데이터 플로우 다이어그램

```
📹 USB 카메라 (cam01, cam02)
    ↓
🎬 [1] 1_camera_publisher.py
    ├─→ /camera/cam01/raw
    └─→ /camera/cam02/raw
         ↓
⚡ [2] 2_detection_node.py
    ├─→ /detection/cam01/person ————→ 👥 사람 명수 카운트ⓘ
    ├─→ /detection/cam02/person ————→ 👥 사람 명수 카운트ⓘ
    └─→ /detection/cam{01,02}/turtlebot
         ↓↓
    ┌────────────┬──────────────┐
    │            │              │
🖼️ [3]       🚨 [4]           │
overlay_node collapse_detector │
    ├──→ /output/cam01 ————————→ 🎥 최종영상ⓘ
    └──→ /output/cam02 ————────→ 🎥 최종영상ⓘ
                 │
                 ├─→ /alert/cam01/collapse ──→ ⚠️ 붕괴감지ⓘ
                 ├─→ /alert/cam02/collapse ──→ ⚠️ 붕괴감지ⓘ
                 └─→ /alert/cam{01,02}/diff
                      ↑
                      │ /control/alert/reset
                      │ 초기화 신호ⓘ
                      │
                 👤 사용자 UI

ⓘ = 사용자 인터페이스 필수 항목
```

---

## 💻 주요 기능 사용

### 1️⃣ 최종영상 (실시간 감시)
```bash
# cam01 시각화 이미지 확인
ros2 topic echo /output/cam01

# cam02 시각화 이미지 확인
ros2 topic echo /output/cam02
```
**출력**: 사람/로봇 감지 결과가 표시된 이미지

### 2️⃣ 사람 명수 카운트
```bash
# cam01 사람 감지 데이터 조회
ros2 topic echo /detection/cam01/person
```
**출력 예**: `{"person_count": 3, "boxes": [...]}`

### 3️⃣ 붕괴감지 모니터링
```bash
# cam01 붕괴 감지 상태
ros2 topic echo /alert/cam01/collapse
# True = 붕괴 감지, False = 정상

# cam02 붕괴 감지 상태
ros2 topic echo /alert/cam02/collapse
```

### 4️⃣ 사용자 초기화
```bash
# ✅ 추천: 모든 카메라 초기화
ros2 topic pub /control/alert/reset std_msgs/String "{data: 'reset'}"

# 특정 카메라만 초기화
ros2 topic pub /control/alert/reset std_msgs/String "{data: 'reset:cam01'}"
ros2 topic pub /control/alert/reset std_msgs/String "{data: 'reset:cam02'}"

# 파일 신호로 초기화 (외부 프로세스용)
touch /tmp/reset_alert.all      # 모든 카메라
touch /tmp/reset_alert.cam01    # cam01만
touch /tmp/reset_alert.cam02    # cam02만
```

---

## 📋 전체 토픽 맵

**필수 (사용자 인터페이스):**
```
📤 /output/cam01              → 최종영상 1
📤 /output/cam02              → 최종영상 2
📤 /detection/cam01/person    → 사람 명수 1
📤 /detection/cam02/person    → 사람 명수 2
📤 /alert/cam01/collapse      → 붕괴감지 1
📤 /alert/cam02/collapse      → 붕괴감지 2
📥 /control/alert/reset       → 초기화 명령 쓰기
```

**보조 (분석용):**
```
📤 /detection/cam{01,02}/turtlebot
📤 /alert/cam{01,02}/diff
📤 /camera/cam{01,02}/raw
```

---

## ⚙️ 파일 설명

| 파일 | 역할 | 발행 토픽 |
|------|------|----------|
| `1_camera_publisher.py` | USB 카메라 자동 감지 및 이미지 발행 | `/camera/cam{01,02}/raw` |
| `2_detection_node.py` | YOLO 사람/로봇 감지 | `/detection/cam{01,02}/person`<br/>`/detection/cam{01,02}/turtlebot` |
| `3_overlay_node.py` | 감지 결과 시각화 (최종영상 생성) | `/output/cam{01,02}` |
| `4_collapse_detector.py` | 붕괴 감지 + 초기화 수신 | `/alert/cam{01,02}/collapse`<br/>`/alert/cam{01,02}/diff` |

---

## 🔧 성능 조정

### 감도 조정
```python
# 2_detection_node.py - 사람 감지 민감도
class PersonModelConfig:
    conf_threshold: float = 0.15  # 낮을수록 더 민감

# 4_collapse_detector.py - 붕괴 감지 민감도
class CollapseDetectorConfig:
    diff_threshold: int = 20           # 낮을수록 더 민감
    area_ratio_threshold: float = 0.03 # 낮을수록 더 민감
```

### 카메라 성능
```python
# 1_camera_publisher.py
class CameraConfig:
    DEFAULT_FPS = 10         # 프레임 속도
    DEFAULT_WIDTH = 800      # 이미지 폭
    DEFAULT_HEIGHT = 600     # 이미지 높이
    DEFAULT_QUALITY = 80     # JPEG 품질
```

---

## ❓ 문제 해결

### 카메라 미인식
```bash
ls /dev/video*  # 연결된 카메라 확인
```

### 토픽 확인
```bash
ros2 topic list                       # 모든 토픽 목록
ros2 topic hz /output/cam01           # 최종영상 발행 속도
ros2 topic hz /detection/cam01/person # 사람 감지 발행 속도
ros2 topic hz /alert/cam01/collapse   # 붕괴감지 발행 속도
```

### 노드 상태
```bash
ros2 node list  # 실행 중인 모든 노드
```

---

## ⚠️ 주의

1. **실행 순서**: 1→2→3→4 (카메라 먼저 시작)
2. **첫 실행**: 모델 로딩에 30초+ 소요
3. **GPU 없음**: CPU 실행 시 속도 저하
4. **최대 2대**: 자동으로 최대 2개 카메라 감지
