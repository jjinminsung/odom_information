# odom_information

ROS1 기반 차량 상태 생성 패키지입니다.

대학생 경진대회에 참가하며 여러 패키지에서 차량의 pose, yaw, speed를 각각 받고 있어 코드 수정에 어려움이 있었고, 이를 해결하기 위해 만든 패키지입니다. odom 메시지를 활용하여 차량의 상태를 실시간으로 받아 localization, 경로 추종을 수행합니다.

GPS로부터 차량의 pose를 받아오고, 설정한 모드에 따라 Dual GPS 또는 IMU를 이용하여 차량의 yaw를 계산해 `/odom` 토픽을 발행합니다. 또한 MORAI의 `/Competition_topic` 또는 `/cur_speed`에서 차량 속도를 받아 `/odom.twist.twist.linear.x`에 반영합니다. 필요 시 `map → base_link` TF도 함께 발행합니다.

## 1. 주요 기능

1. GPS back을 기준으로 차량 위치 생성
2. 두 가지 yaw 계산 모드 지원 (`dual_gps`, `gps_imu`)
3. yaw LPF 적용
4. Dual GPS baseline 유효 범위 검사
5. `/odom` 결과 중복 발행 방지

## 2. 동작 방식

동작 방식을 설명하기 이전에, 차량의 위치를 받는 `/odom_back` 토픽은 GPS의 raw data를 UTM 좌표계로 바꾼 후의 차량 위치 좌표입니다.

### 2.1 `gps_imu` 모드

- **위치**: `/odom_back`
- **yaw**: `/imu` — IMU quaternion을 Roll-Pitch-Yaw로 변환한 뒤 yaw 사용
- **속도**: `/Competition_topic` 우선
- **`/odom` 발행 시점**: Back GPS가 새 데이터로 갱신되었을 때 (IMU 콜백은 yaw만 갱신)

### 2.2 `dual_gps` 모드

- **위치**: `/odom_back`
- **yaw**: `/odom_back → /odom_front` 방향 벡터의 `atan2(dy, dx)`
- **속도**: `/Competition_topic` 우선
- **`/odom` 발행 시점**: Front/Back GPS가 모두 새 데이터로 갱신되었을 때

baseline 길이가 `min_baseline` ~ `max_baseline` 범위를 벗어나면 GPS 튐으로 간주하고 직전 yaw를 유지합니다.

### 2.3 공통 처리

- **yaw 필터**: 두 모드 모두 각도 wrap(`±π`)을 고려한 저역통과 필터(LPF)로 yaw를 평활화하고, `yaw_offset`으로 센서 장착 오차를 보정합니다.
- **중복 발행 방지**: `header.seq`를 추적해 같은 센서 조합으로 두 번 발행하지 않습니다.
- **타임스탬프**: `/odom`의 `header.stamp`는 `ros::Time::now()`가 아니라 입력 센서의 stamp를 사용합니다. 반면 TF는 시뮬레이터 클럭과 wall time이 어긋날 수 있어 `now()`로 발행합니다.
- **속도 폴백**: `/Competition_topic`의 속도 벡터 크기를 우선 사용하고, stale(`*_stale_sec` 초과)이거나 유효 범위를 벗어나면 `/cur_speed`로, 그것도 없으면 0으로 폴백합니다.

## 3. 토픽

### Subscribe

| 토픽 | 타입 | 설명 |
| --- | --- | --- |
| `/odom_back` | `nav_msgs/Odometry` | 후방 GPS. position의 기준이며 publish 트리거 역할 |
| `/odom_front` | `nav_msgs/Odometry` | 전방 GPS (`dual_gps` 모드에서만 구독) |
| `/imu` | `sensor_msgs/Imu` | IMU orientation (`gps_imu` 모드의 yaw 소스) |
| `/Competition_topic` | `morai_msgs/EgoVehicleStatus` | MORAI ego 속도 (우선순위 높음) |
| `/cur_speed` | `std_msgs/Float32` | 속도 fallback |

### Publish

| 토픽 | 타입 | 설명 |
| --- | --- | --- |
| `/odom` | `nav_msgs/Odometry` | `map` 프레임 기준 pose + `base_link` 기준 전진 속도 |
| `/tf` | `map → base_link` | `publish_tf: true`일 때만 발행 |

## 4. 파라미터

`config/odom_information.yaml`에서 설정합니다.

| 파라미터 | 기본값 | 설명 |
| --- | --- | --- |
| `mode` | `gps_imu` | `dual_gps` 또는 `gps_imu` |
| `publish_tf` | `true` | `map → base_link` TF 발행 여부 |
| `frames/parent`, `frames/child` | `map`, `base_link` | 프레임 ID |
| `topics/*` | 위 표 참조 | 입출력 토픽 이름 |
| `competition_speed_enabled` | `true` | MORAI ego 속도 사용 여부 |
| `competition_speed_scale` | `1.0` | 속도 스케일 보정 |
| `competition_speed_stale_sec` | `0.5` | 이 시간을 넘긴 값은 무시 (초) |
| `competition_speed_max_mps` | `60.0` | 이 값을 넘는 속도는 이상치로 폐기 |
| `cur_speed_is_kmh` | `true` | `/cur_speed`가 km/h 단위인지 |
| `speed_scale`, `speed_stale_sec` | `1.0`, `0.5` | `/cur_speed` 보정 / 유효시간 |
| `<mode>/yaw_offset` | `0.0` | 장착 오차 보정 (rad) |
| `<mode>/yaw_lpf_alpha` | `0.2` | 0~1, 작을수록 부드러움 (1.0 = 필터 없음) |
| `<mode>/print_yaw`, `<mode>/print_period` | `true`, `0.1` | yaw 디버그 로그 |
| `dual_gps/min_baseline` | `0.0` | 유효 baseline 최소 길이 (m) |
| `dual_gps/max_baseline` | `5.0` | 유효 baseline 최대 길이 (m) |

`yaw_offset`, `yaw_lpf_alpha`, `print_yaw`, `print_period`는 `dual_gps:` / `gps_imu:` 네임스페이스 아래에 모드별로 따로 둡니다.

### 튜닝 가이드

- 정지 상태에서 yaw가 흔들릴 때: `yaw_lpf_alpha`를 낮추고(0.2 → 0.1), `dual_gps`라면 `min_baseline`을 실제 GPS 간 거리에 가깝게 올립니다.
- yaw가 실제보다 일정하게 틀어져 있을 때: `yaw_offset`을 라디안으로 조정합니다 (5° = 0.087266).

## 5. 의존성

`roscpp`, `nav_msgs`, `sensor_msgs`, `geometry_msgs`, `std_msgs`, `tf`, `tf2`, `tf2_ros`, `morai_msgs`

`morai_msgs`는 rosdep으로 받을 수 없으므로 같은 워크스페이스의 `src/`에 함께 두어야 합니다.

## 6. 빌드 및 실행

```bash
cd ~/catkin_ws/src
git clone https://github.com/jjinminsung/odom_information.git

cd ~/catkin_ws
catkin_make          # 또는 catkin build
source devel/setup.bash

roslaunch odom_information odom_information.launch
```

launch 파일이 `config/odom_information.yaml`을 자동으로 로드합니다.

### 확인

```bash
rostopic echo /odom
rosrun tf tf_echo map base_link
```
