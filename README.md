# odom_information

ROS1 기반 차량 상태 생성 패키지입니다. 
대학생 경진대회에 참가하며 여러 패키지에서 차량의 pose, yaw, speed .. 를 각각 받고 있어 코드 수정에 어려움을 해결하기 위해 만든 패키지입니다.
odom 메시지를 활용하여 차량의 상태를 실시간으로 받아 localization, 경로 추종을 수행합니다.

GPS로 부터 차량의 pose를 받아오고 설정한 모드에 따라 Dual Gps 또는 IMU를 이용하여 차량의 yaw를 계산해 odom msg 형식을 활용해 /odom 토픽을 발행합니다. 
또한 MORAI의 /Competition_topic 또는 /cur_speed에서 차량 속도를 받거나 위치 차분을 사용하여 /odom.twist.twist.linear.x에 반영합니다.
필요 시 map -> base_link TF도 함께 발행합니다.

1. 주요기능
   1) GPS back을 기준으로 차량 위치 생성
   2) 두가지 yaw 계산 모드 지원
   3) yaw LPF 적용
   4) Dual GPS baseline 유효 범위 검사
   5) /odom 결과 중복 발행 방지


2. 동작 방식
   동작 방식을 설명하기 이전에 차량의 위치를 받는 /odom_back 토픽은 GPS의 raw data를 utm 좌표계로 바꾼 후의 차량 위치 좌표입니다.
   
   2.1 gps_imu 모드
   
      위치: /odom_back
   
      yaw: /imu
           -> IMU quaternion을 Roll-Pitch-Yaw로 변환한 뒤 yaw 사용
   
      속도: /Competition_topic 우선

   2.2 dual_gps 모드
   
      위치: /odom_back
   
      yaw: /odom_back → /odom_front 방향 벡터
   
      속도: /Competition_topic 우선
   
      /odom 발행 시점: Front/Back GPS가 모두 새 데이터로 갱신되었을 때
