cmake_minimum_required(VERSION 3.0.2)
project(odom_information)

find_package(catkin REQUIRED COMPONENTS
  geometry_msgs
  morai_msgs
  nav_msgs
  roscpp
  sensor_msgs
  std_msgs
  tf
  tf2
  tf2_ros
)

catkin_package(
)

include_directories(
  ${catkin_INCLUDE_DIRS}
)

add_executable(odom_information_node src/odom_information_node.cpp)
target_link_libraries(odom_information_node ${catkin_LIBRARIES})
add_dependencies(odom_information_node ${catkin_EXPORTED_TARGETS})
