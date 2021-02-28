# Week-7-8-Navigation



# Ground truth térkép
world_map_creation.world

<plugin filename="libcollision_map_creator.so" name="collision_map_creator"/>

gzserver src/Week-7-8-Navigation/bme_ros_navigation/worlds/world_map_creation.world

roslaunch pgm_map_creator request_publisher.launch

A request publisher:
```xml
<?xml version="1.0" ?>
<launch>
  <arg name="map_name" default="map" />
  <arg name="save_folder" default="$(find pgm_map_creator)/maps" />
  <arg name="xmin" default="-15" />
  <arg name="xmax" default="15" />
  <arg name="ymin" default="-15" />
  <arg name="ymax" default="15" />
  <arg name="scan_height" default="5" />
  <arg name="resolution" default="0.05" />

  <node pkg="pgm_map_creator" type="request_publisher" name="request_publisher" output="screen" args="'($(arg xmin),$(arg ymax))($(arg xmax),$(arg ymax))($(arg xmax),$(arg ymin))($(arg xmin),$(arg ymin))' $(arg scan_height) $(arg resolution) $(arg save_folder)/$(arg map_name)">
  </node>
</launch>
```

 roslaunch bme_ros_navigation world.launch world_file:='$(find bme_ros_navigation)/worlds/20m_corridor_empty.world'

 roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_empty.world'