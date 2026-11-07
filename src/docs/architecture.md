# 구조 그림 (노드 · 토픽 · launch · TF)

[interfaces.md](interfaces.md)의 표와 코드의 발행 · 구독 · 서비스 선언을 맞춰 보고 그린 그림이다 (main `7ea3da1` 기준).
QoS · 주기 · 동작의 자세한 뜻은 `interfaces.md`를 본다. 노드 · 토픽 이름을 바꾸면 이 파일도 함께 고친다.

GitHub, VS Code(Markdown 미리 보기 + Mermaid 확장)에서 그림으로 보인다.
웹 게이트웨이(`lekiwi_web_operator`, 이 워크스페이스 밖)의 연결은 그 저장소 `756c06f`의 코드를 보고 그렸다. 게이트웨이는 그림의 토픽 대부분을 웹 화면이 그 값을 볼 때만 구독한다.

| 모양 | 뜻 |
|------|----|
| 사각형 | ROS 노드 |
| 캡슐 | 토픽 (발행 → 토픽 → 구독) |
| 육각형 | 서비스 (점선: 부르는 쪽 → 서비스 → 서버) |
| 원통 | 장치 · ROS 밖 프로그램 |

## 1. 한눈에 보기

```mermaid
flowchart LR
  subgraph PC
    CLIENT[("lerobot LeKiwiClient")]
    BROWSER[("브라우저 조작 화면")]
    RVIZ["RViz"]
  end
  subgraph PI["Pi"]
    GW["웹 게이트웨이<br/>lekiwi_web_operator"]
    ZMQ["lekiwi_zmq_host_node"]
    TELEOP["조작 도구<br/>lekiwi_teleop"]
    NAV["Nav2"]
    CARTO["Cartographer"]
    BC["lekiwi_base_controller_node"]
    ARM["lekiwi_follower_arm_node"]
    DRV["lekiwi_motor_driver_node"]
    LID["lekiwi_lidar_node"]
    SF["lekiwi_scan_filter_node"]
    IMU["lekiwi_imu_node"]
    CAM["카메라 노드 2개"]
    BUS[("Feetech 버스<br/>/dev/ttyACM0")]
  end
  BROWSER <-->|":8080"| GW
  CLIENT <-->|"ZMQ :5555 · :5556"| ZMQ
  RVIZ -.->|"목표 · 초기 위치"| NAV
  GW & ZMQ & TELEOP & NAV -->|"cmd_vel"| BC
  GW & ZMQ -->|"arm/goal"| ARM
  BC -->|"wheel_command"| DRV
  ARM -->|"arm_command"| DRV
  DRV -->|"motor/states"| BC & ARM
  DRV <--> BUS
  LID -->|"scan"| SF
  SF -->|"scan_filtered"| CARTO & NAV
  IMU -->|"offset_imu"| CARTO
  BC -->|"odom"| CARTO & NAV
  CAM -->|"image_raw/compressed"| ZMQ
  ARM -->|"lerobot/state"| ZMQ
```

`cmd_vel`은 한 번에 발행자 하나만 둔다 (조작 도구 · 게이트웨이 · ZMQ 호스트 · Nav2 중 하나).

## 2. bringup이 켜는 노드

`lekiwi_bringup/launch/bringup.launch.py`. 바퀴 · 팔 토크는 켜지 않는다 (`lekiwi/motor/power`, `lekiwi/motor/arm_torque`로 켠다).

```mermaid
flowchart TD
  L["bringup.launch.py"]
  L --> MODEL["robot_state_publisher<br/>joint_state_publisher"]
  L -->|"use_imu (기본 true)"| IMU["lekiwi_imu_node"]
  L -->|"use_lidar (기본 true)"| LID["lekiwi_lidar_node<br/>lekiwi_scan_filter_node"]
  L -->|"use_camera (기본 false)<br/>lekiwi_camera가 빌드됐을 때"| CAM["front_camera_node<br/>wrist_camera_node"]
  subgraph MOT["use_motors (기본 true)"]
    DRV["lekiwi_motor_driver_node<br/>wheel_write_enabled = use_base_controller<br/>arm_write_enabled = use_arm"]
    BC["lekiwi_base_controller_node"]
    ARM["lekiwi_follower_arm_node<br/>lerobot 0.6.0 venv python으로 실행"]
    ZMQ["lekiwi_zmq_host_node"]
  end
  L --> DRV
  L -->|"use_base_controller (기본 true)"| BC
  L -->|"use_arm (기본 true)<br/>lekiwi_lerobot 빌드 + lerobot_python 있음"| ARM
  L -->|"use_zmq_host (기본 true)<br/>lekiwi_zmq_host가 빌드됐을 때"| ZMQ
```

노드마다 파라미터를 넣는 순서. 오른쪽이 이긴다.

```mermaid
flowchart LR
  P1["패키지 기본 설정<br/>config/*.yaml"] --> P2["base_frame.yaml<br/>베이스 컨트롤러만"]
  P2 --> P3["장비 프로파일<br/>robot_profile 인자<br/>→ ~/lekiwi_ws/robot_profile.yaml<br/>→ 패키지 기본값"]
  P3 --> P4["~/lekiwi_ws/arm_profile.yaml<br/>팔로워만, 있으면"]
  P4 --> P5["robot_id 인자<br/>팔로워만, 주면"]
```

드라이버는 장비 프로파일 뒤에 `wheel_write_enabled` · `arm_write_enabled`를, ZMQ 호스트는 `zmq_cmd_port` · `zmq_obs_port` 인자를 얹는다.

## 3. 바퀴 (주행) 경로

명령 (모터 쪽으로 내려가는 흐름)

```mermaid
flowchart LR
  TELEOP["조작 도구<br/>lekiwi_teleop"]
  GW["웹 게이트웨이"]
  ZMQ["lekiwi_zmq_host_node<br/>세션 active일 때만 발행"]
  NAVCM["Nav2 collision_monitor"]
  GUARD["Nav2 감독<br/>nav2_motor_guard"]
  BC["lekiwi_base_controller_node<br/>기구학 · 안전 상태기계"]
  DRV["lekiwi_motor_driver_node"]
  BUS[("Feetech 버스<br/>바퀴 3 · 팔 6")]

  T_CMD(["cmd_vel"])
  T_WC(["lekiwi/motor/wheel_command<br/>50 Hz"])
  S_POWER{{"lekiwi/motor/power"}}
  S_WT{{"lekiwi/motor/wheel_torque"}}
  S_RST{{"lekiwi/motor/reset_odometry"}}

  TELEOP & GW & ZMQ & NAVCM --> T_CMD --> BC
  BC --> T_WC --> DRV --> BUS
  TELEOP -.-> S_POWER
  GUARD -.-> S_POWER
  ZMQ -.-> S_POWER
  GW -.-> S_POWER
  S_POWER -.-> BC
  BC -.->|"베이스 컨트롤러만 부른다"| S_WT -.-> DRV
  GW -.-> S_RST
  S_RST -.-> BC
```

사람 · 도구는 드라이버의 `wheel_torque`가 아니라 `lekiwi/motor/power`를 부른다. 베이스 컨트롤러의 안전 상태기계를 거치게 하기 위해서다.

상태 (모터에서 올라오는 흐름)

```mermaid
flowchart LR
  BUS[("Feetech 버스")]
  DRV["lekiwi_motor_driver_node"]
  BC["lekiwi_base_controller_node<br/>오도메트리"]
  JSP["joint_state_publisher"]
  RSP["robot_state_publisher"]
  USERS["Cartographer · Nav2 · Nav2 감독 · 게이트웨이"]
  WATCH["조작 도구 · Nav2 감독 · ZMQ 호스트 · 게이트웨이"]
  GW["웹 게이트웨이"]

  T_ST(["lekiwi/motor/states<br/>30 Hz"])
  T_ODOM(["odom<br/>50 Hz"])
  T_READY(["lekiwi/motor/ready"])
  T_WJS(["lekiwi/motor/wheel_joint_states"])
  T_JS(["joint_states"])
  T_TF(["tf"])

  BUS --> DRV --> T_ST --> BC
  T_ST --> GW
  BC --> T_ODOM --> USERS
  BC --> T_READY --> WATCH
  BC --> T_WJS --> JSP --> T_JS --> RSP --> T_TF
  BC -->|"odom → base_footprint"| T_TF
```

## 4. 팔 경로

팔 토픽 · 서비스(`arm_command`, `arm_ready`, `arm_registers`, `arm_torque`)는 `use_arm:=true`일 때만 생긴다.

명령

```mermaid
flowchart LR
  CLIENT[("PC lerobot LeKiwiClient")]
  GW["웹 게이트웨이"]
  ZMQ["lekiwi_zmq_host_node"]
  ARM["lekiwi_follower_arm_node<br/>LeRobot 단위 → tick · 보정"]
  DRV["lekiwi_motor_driver_node"]
  BUS[("Feetech 버스")]

  T_EST(["lekiwi/safety/estop_latched"])
  T_GOAL(["lekiwi/arm/goal<br/>LeRobot 단위"])
  T_AC(["lekiwi/motor/arm_command<br/>tick, 약 30 Hz"])
  S_REST{{"lekiwi/arm/rest"}}
  S_AT{{"lekiwi/motor/arm_torque"}}

  CLIENT -->|"ZMQ 명령 :5555"| ZMQ
  GW --> T_EST --> ZMQ
  GW & ZMQ --> T_GOAL --> ARM
  ARM --> T_AC --> DRV --> BUS
  GW -.-> S_REST -.-> ARM
  GW -.-> S_AT
  ZMQ -.-> S_AT
  S_AT -.-> DRV
```

상태

```mermaid
flowchart LR
  BUS[("Feetech 버스")]
  DRV["lekiwi_motor_driver_node"]
  ARM["lekiwi_follower_arm_node<br/>tick → LeRobot 단위"]
  ZMQ["lekiwi_zmq_host_node"]
  GW["웹 게이트웨이"]
  CLIENT[("PC lerobot LeKiwiClient<br/>데이터셋 기록")]

  T_ST(["lekiwi/motor/states"])
  T_AR(["lekiwi/motor/arm_ready"])
  T_REG(["lekiwi/motor/arm_registers"])
  T_AS(["lekiwi/arm/state"])
  T_LS(["lekiwi/lerobot/state<br/>observation.state 9개"])
  T_STAT(["lekiwi/arm/status<br/>JSON"])
  T_ZS(["lekiwi/zmq_host/status<br/>JSON"])

  BUS --> DRV
  DRV --> T_ST & T_REG --> ARM
  T_REG --> GW
  DRV --> T_AR --> ARM & ZMQ & GW
  ARM --> T_AS --> GW
  ARM --> T_LS --> ZMQ
  ARM --> T_STAT --> ZMQ & GW
  ZMQ --> T_ZS --> GW
  ZMQ -->|"ZMQ 관측 :5556"| CLIENT
```

ZMQ 호스트는 세션의 첫 팔 명령 때만 `arm_torque` true를 부른다. 게이트웨이는 bringup을 끄기 전에 `arm/rest` → `rested`를 본 뒤 `arm_torque` false를 부른다.

## 5. 센서 · 지도

```mermaid
flowchart LR
  LIDAR[("YDLIDAR")] --> LID["lekiwi_lidar_node"]
  LID --> T_SCAN(["lekiwi/sensor/scan<br/>10 Hz"])
  T_SCAN --> SF["lekiwi_scan_filter_node<br/>로봇 자기 반사 제거"]
  SF --> T_SF(["lekiwi/sensor/scan_filtered"])
  BMI[("BMI160")] --> IMU["lekiwi_imu_node"]
  IMU --> T_OI(["lekiwi/sensor/offset_imu<br/>자이로 바이어스만 제거"])
  IMU --> T_LPF(["lekiwi/sensor/lpf_imu<br/>필터 + 자세"])
  IMU --> T_KF(["lekiwi/sensor/calibrated_imu<br/>구독자 없음"])
  UVC[("USB 카메라 2대")] --> CAM["front_camera_node<br/>wrist_camera_node"]
  CAM --> T_IMG(["lekiwi/sensor/front_camera/image_raw/compressed<br/>lekiwi/sensor/wrist_camera/image_raw/compressed"])
  T_IMG --> ZMQ["lekiwi_zmq_host_node<br/>클라이언트가 있을 때만 구독"]
  T_ODOM(["odom"])
  T_SF -->|"remap → scan"| CARTO["Cartographer"]
  T_OI -->|"remap → imu"| CARTO
  T_ODOM --> CARTO
  CARTO --> T_MAP(["map"])
  T_SF & T_ODOM --> NAV["Nav2"]
  T_SF & T_ODOM --> GUARD["Nav2 감독<br/>nav2_motor_guard"]
  T_LPF & T_SF & T_ODOM & T_IMG --> GW["웹 게이트웨이<br/>화면이 볼 때만 구독"]
```

## 6. Nav2 명령 경로

`lekiwi_navigation2/launch/navigation.launch.py`. 바퀴로 가는 명령은 `collision_monitor`만 `cmd_vel`로 낸다.

```mermaid
flowchart LR
  CS["controller_server"] & BS["behavior_server"] -->|"remap cmd_vel → cmd_vel_nav"| T_NAV(["cmd_vel_nav"])
  T_NAV --> VS["velocity_smoother"]
  VS --> T_SM(["cmd_vel_smoothed"])
  T_SM --> CM["collision_monitor"]
  CM --> T_CMD(["cmd_vel"])
  T_CMD --> BC["lekiwi_base_controller_node"]
```

## 7. TF

```mermaid
flowchart TD
  MAP["map"] -->|"Cartographer(지도 작성) 또는 AMCL(저장 지도), 하나만"| ODOM["odom"]
  ODOM -->|"lekiwi_base_controller_node, 발행자는 하나뿐"| BF["base_footprint"]
  subgraph RSP["robot_state_publisher (lekiwi_description URDF)"]
    BL["base_link"]
    WHEEL["back · left · right_wheel_mount_link<br/>→ *_wheel_link (바퀴 관절)"]
    FC["front_camera_optical_frame"]
    SB["soarm_base_link"]
    LIDAR["lidar_link"]
    IMUL["imu_link"]
    ARMC["SO-101 팔 체인<br/>shoulder_link … wrist_link"]
    WC["wrist_camera_optical_frame"]
  end
  BF --> BL
  BL --> WHEEL & FC & SB
  SB --> LIDAR & IMUL & ARMC
  ARMC --> WC
```

`lidar_link` · `imu_link`는 bringup이 `use_hardware_sensors:=true`로 URDF를 만들 때만 생긴다.
