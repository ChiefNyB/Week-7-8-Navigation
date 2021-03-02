[//]: # (Image References)

[image1]: ./assets/gazebo_world.png "Gazebo"
[image2]: ./assets/gazebo_corridor_empty.png "Gazebo"
[image3]: ./assets/gazebo_corridor_features.png "Gazebo"
[image4]: ./assets/gazebo_corridor_robot_1.png "Gazebo"
[image5]: ./assets/gazebo_corridor_robot_2.png "Gazebo"
[image6]: ./assets/tf_tree.png "TF"
[image7]: ./assets/graph_1.png "Graph"
[image8]: ./assets/roswtf.png "roswtf"

# 7. - 8. hét - ROS navigáció

# Hova fogunk eljutni?

<a href="https://youtu.be/8bnjzPTNLfc"><img height="400" src="./assets/youtube1.png"></a>  
<a href="https://youtu.be/-YCcQZmKJtY"><img height="400" src="./assets/youtube2.png"></a>

# Tartalomjegyzék
1. [Kezdőcsomag](#Kezdőcsomag)  
2. [Gazebo világok](#Gazebo-világok)  
3. [Ground truth térkép készítése](#Ground-truth-térkép-készítése)
4. [IMU és odometria szenzorfúzió](#IMU-és-odometria-szenzorfúzió)  
5. [Mapping](#Mapping)  
5.1. [Hector SLAM](#Hector-SLAM)  
5.2. [GMapping](#GMapping)  
5.3. [Map saver](#Map-saver)
6. [Lokalizáció](#Lokalizáció)  
6.1. [AMCL](#AMCL)
7. [Navigáció](#Navigáció)  
7.1. [Recovery akciók](#Recovery-akciók)
8. [Waypoint navigáció](#Waypoint-navigáció)  
8.1. [Grafikusan a follow_waypoints csomaggal](#Grafikusan-a-follow_waypoints-csomaggal)  
8.2. [Waypoint navigáció C++ ROS node-ból](#Waypoint-navigáció-C++-ROS-node-ból)

# Kezdőcsomag
A lecke kezdőcsomagja épít az előző fejezetekre, de egy külön GIT repositoryból dolgozunk, így nem feltétele az előző csomagok megléte.

A kiindulási projekt tartalmazza a Gazebo világ szimulációját, az alap differenciálhajtású MOGI robotunk modelljét és szimulációját, a kamera, IMU és Lidar szimulációját, valamint az alap launchfájlokat és RViz fájlokat.

A kezdőprojekt letöltése:
```console
git clone -b starter-branch https://github.com/MOGI-ROS/Week-7-8-Navigation.git
```

A kezdőprojekt tartalma a következő:
```console
david@DavidsLenovoX1:~/bme_catkin_ws/src/Week-7-8-Navigation/bme_ros_navigation$ tree
.
├── CMakeLists.txt
├── launch
│   ├── check_urdf.launch
│   ├── spawn_robot.launch
│   ├── teleop.launch
│   └── world.launch
├── meshes
│   ├── lidar.dae
│   ├── mogi_bot.dae
│   ├── vlp16.dae
│   └── wheel.dae
├── package.xml
├── rviz
│   ├── check_urdf.rviz
│   └── mogi_world.rviz
├── urdf
│   ├── materials.xacro
│   ├── mogi_bot.gazebo
│   └── mogi_bot.xacro
└── worlds
    └── world_modified.world
```

# Gazebo világok


A fejezet során a már jól megismert `world_modified.world` Gazebo világot fogjuk használni, amit a következő paranccsal tudunk tesztelni:
```console
roslaunch bme_ros_navigation world.launch
```
![alt text][image1]

Emellett azonban szükségünk lesz egy másik világra is, ami egy 20m hosszú folyosóból áll, ezen fogjuk tesztelni a térképezési algoritmusokat. Ennek két verzióját hoztam létre előre, egy üreset és egy olyat, ahol vannak objektumok a folyosón. Ezeket is ki tudjuk próbálni:
```console
roslaunch bme_ros_navigation world.launch world_file:='$(find bme_ros_navigation)/worlds/20m_corridor_empty.world'
```
![alt text][image2]
```console
roslaunch bme_ros_navigation world.launch world_file:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world'
```
![alt text][image3]

A robotunkat is betölthetjük a világba, a korábbiakhoz hasonlóan a `spawn_robot.launch` segítségével. Robot betöltése az alap világba:
```console
roslaunch bme_ros_navigation spawn_robot.launch
```
És a világ megadásával betölthetjük a folyosóra is:
```console
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_empty.world' x:=-7 y:=2
```
![alt text][image4]
Vegyük észre, hogy felülbíráljuk a robot kezdeti pozícióját is ezzel a paranccsal, nézzük meg mi történik nélküle.
![alt text][image5]

A robot ilyekor sem a (0,0) pozícióban indul, mert a `spawn_robot.launch` fájlban ezek az alapértelmezett értékek:
```xml
...
  <arg name="x" default="2.5"/>
  <arg name="y" default="1.5"/>
  <arg name="z" default="0"/>
  <arg name="roll" default="0"/>
  <arg name="pitch" default="0"/>
  <arg name="yaw" default="0"/>
...
```

Ennek megfelelően a másik folyosómodellre is elhelyezhető a robot:
```console
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world' x:=-7 y:=2
```

# Ground truth térkép készítése

Lehetőségünk van a Gezbo világunkból egy úgy nevezett ground truth térképet készíteni. Ehhez a `pgm_map_creator` csomagot használhatjuk.
Ez nem egy hivatalos ROS csomag, így letölthető a tárgy GitHub oldaláról a catkin_workspace-be:
```console
git clone https://github.com/MOGI-ROS/pgm_map_creator
```
A csomag használatához be kell tennünk egy plugint a Gazebo világunkba, ezért csináljunk róla egy másolatot `world_map_creation.world` néven a `worlds/map_creation` mappába.

Tegyük be a plugint a fájl végére a `</world>` tag elé:

```xml
...
    <plugin filename="libcollision_map_creator.so" name="collision_map_creator"/>
  </world>
</sdf>
```

A plugin használatához indítsuk el a világunk szimulációját, ehhez nincs szükség a grafikus frontendre, így elég a gzserver-t használnunk.

```console
gzserver src/Week-7-8-Navigation/bme_ros_navigation/worlds/world_map_creation.world
```

És egy másik terminálból indítsuk el a map creator-t:
```console
roslaunch pgm_map_creator request_publisher.launch
```

A `request_publisher` launch fájlban tudjuk megadni a térképünk méretét és felbontását.
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

Előre elkészítettem a grund truth térképet a `world_modified.world` és a `20m_corridor_empty.world` alapján, így ezeket használhatjuk a kezdőcsomagból.

A `pgm_map_creator` alapértelmezetten a saját csomagjának a maps mappájába menti a térkép fájlokat. A `.pgm` fájlok egyszerű bitmap-ek, többek között megnyithatók a GIMP vagy Inkscape szoftverekkel.

# IMU és odometria szenzorfúzió
Vessünk még egy pillantást a `spawn_robot.launch` fájlra! Ebben ugyanis van pár változás a korábbiakhoz képest. Bekerült egy új node:
```xml
  <!-- Robot pose EKF for sensor fusion -->
  <node pkg="robot_pose_ekf" type="robot_pose_ekf" name="robot_pose_ekf">
    <remap from="imu_data" to="/imu/data"/>
    <param name="output_frame" value="odom"/>
    <param name="base_footprint_frame" value="base_footprint"/>
    <param name="freq" value="30.0"/>
    <param name="sensor_timeout" value="1.0"/>
    <param name="odom_used" value="true"/>
    <param name="imu_used" value="true"/>
    <param name="vo_used" value="false"/>
    <param name="gps_used" value="false"/>
    <param name="debug" value="false"/>
    <param name="self_diagnose" value="false"/>
  </node>
```
És ezzel együtt változott egy picit a `mogi_bot.gazebo` fájl is:
```xml
<publishOdomTF>false</publishOdomTF>
```

Mostantól nem a Gazebo plugin csinálja a transzformációt az odom fix frame és a robot alváza között, hanem a szenzorfúzió. Ez sokkal jobban hasonlít egy valódi robotra, ahol a szenzor adatok (IMU és Odoemtria) megadott topicokba kerülnek, majd ezek alapján a szenzorfúzió hozza létre a transzformációt.
Nézzük is meg a TF tree-t, ami a `rosrun rqt_tf_tree rqt_tf_tree` paranccsal tudok elindítani.
![alt text][image6]

Valamint a node-jaink összekötését is vizsgáljuk meg az `rqt_graph` paranccsal:
![alt text][image7]

Hibakeresés miatt, kapcsoljuk most vissza a `publishOdomTF`-et a Gazebo pluginban, és nézzük meg mi történik!

A robotunk furcsán ugrál az RVizben megjelenítva, de szerencsére ennél nagyobb baj nem történt:
![alt text][image8]

Azokban az esetekben, amikor sejtjük, hogy valami nincs rendben a TF-fel, sokat segít a `roswtf` parancssoros tool, ami ebben az esetben is azonnal észrevette a hibát:
```console
...
Found 1 error(s).

ERROR TF multiple authority contention:
 * node [/robot_pose_ekf] publishing transform [base_footprint] with parent [odom] already published by node [/gazebo]
 * node [/gazebo] publishing transform [base_footprint] with parent [odom] already published by node [/robot_pose_ekf]
```

A végén ne felejtsük el visszaállítani a Gazebo plugint!

# Mapping

Térképezésre általában SLAM (simultaneous localization and mapping) algoritmusokat használunk, amik képesek egyszerre létrehozni a környzet térképét és meghatározni a robot pozícióját és orientációját a térképen (lokalizáció).
Az elmúlt években egyre inkább terjednek a mono, stereo vagy RGBD kamerát használó SLAM algoritmusok, de mi most két Lidart használó algoritmust próbálunk ki.

## Hector SLAM

A Hector SLAM a Darmstadt-i egyetem fejlesztése, nagyon egyszerű használni, akár szabd kézben tartott Lidarral is, ugyanis pusztán csak a Lidar méréseit használja a SLAM probléma megoldása során.
Ez az előnye egyben a hátránya is, mivel nem használja a robot odometriáját, így speciális körülmények között nem használható, látunk is erre példát az üres folyosó esetén.

Előtte azonban próbáljuk ki a Hector SLAM-et a már megszokott Gazebo világunkon.

Hozzuk létre a hector_slam.launch fájlt:
```xml
<?xml version="1.0"?>
<launch>

  <!-- Ground truth map file -->
  <arg name="map_file" default="$(find bme_ros_navigation)/maps/map_hector.yaml"/>

  <node pkg="hector_mapping" type="hector_mapping" name="hector_mapping" output="screen">
    <param name="base_frame" value="base_link" />
    <param name="odom_frame" value="odom"/>
    <param name="output_timing" value="false"/>
    <param name="use_tf_scan_transformation" value="true"/>
    <param name="use_tf_pose_start_estimate" value="false"/>
    <param name="scan_topic" value="scan"/>
    <!-- Map size / start point -->
    <param name="map_resolution" value="0.025"/>
    <param name="map_size" value="2048"/>
    <param name="map_start_x" value="0.5"/>
    <param name="map_start_y" value="0.5" />
    <!--param name="laser_z_min_value" value="-2.5" /-->
    <!--param name="laser_z_max_value" value="3.5" /-->
    <!-- Map update parameters -->
    <param name="update_factor_free" value="0.4"/>
    <param name="update_factor_occupied" value="0.7" />    
    <param name="map_update_distance_thresh" value="0.6"/>
    <param name="map_update_angle_thresh" value="0.9" />
    <param name="pub_map_odom_transform" value="true"/>
    <param name="pub_drawings" value="true"/>
    <param name="pub_debug_output" value="true"/>
  </node>

  <!-- Ground truth map Server -->
  <node name="map_server" pkg="map_server" type="map_server" args="$(arg map_file)">
    <remap from="map" to="map_groundtruth"/>
  </node>

</launch>
```

Majd indítsuk el a világ és a robot szimulációját:

```console
roslaunch bme_ros_navigation spawn_robot.launch
```

A Hector SLAM algoritumust:
```console
roslaunch bme_ros_navigation hector_slam.launch
```

Valamint a távirányítót:
```console
roslaunch bme_ros_navigation teleop.launch
```

### Üres folyosó

Próbáljuk ki a Hector SLAM-et az üres folyosón:
```console
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_empty.world' x:=-7 y:=2 
```

```console
roslaunch bme_ros_navigation hector_slam.launch map_file:='$(find bme_ros_navigation)/maps/corridor_hector.yaml'
```

### Folyosó tárgyakkal

És most nézzük meg tárgyakkal:
```console
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world' x:=-7 y:=2
```

```console
roslaunch bme_ros_navigation hector_slam.launch map_file:='$(find bme_ros_navigation)/maps/corridor_hector.yaml'
```

## GMapping

A másik SLAM algoritmus, amit kipróbálunk a GMapping. A GMapping nem csak a Lidar jeleit használja, hanem a robot odometriáját is!

```xml
<?xml version="1.0"?>
<launch>

  <!-- Ground truth map file -->
  <arg name="map_file" default="$(find bme_ros_navigation)/maps/map.yaml"/>

  <node pkg="gmapping" type="slam_gmapping" name="gmapping" output="screen" >
    <param name="odom_frame" value="odom" />
    <param name="base_frame" value="base_link" />
    <!-- Process 1 out of every this many scans (set it to a higher number to skip more scans)  -->
    <param name="throttle_scans" value="1"/>
    <param name="map_update_interval" value="3.0"/> <!-- default: 5.0 -->

    <!-- The maximum usable range of the laser. A beam is cropped to this value.  -->
    <param name="maxUrange" value="5.0"/>
    <!-- The maximum range of the sensor. If regions with no obstacles within the range of the sensor should appear as free space in the map, set maxUrange < maximum range of the real sensor <= maxRange -->
    <param name="maxRange" value="5.0"/>

    <param name="sigma" value="0.05"/>
    <param name="kernelSize" value="1"/>
    <param name="lstep" value="0.05"/>
    <param name="astep" value="0.05"/>
    <param name="iterations" value="5"/>
    <param name="lsigma" value="0.075"/>
    <param name="ogain" value="3.0"/>
    <param name="minimumScore" value="30.0"/>
    <!-- Number of beams to skip in each scan. -->
    <param name="lskip" value="0"/>
    <param name="srr" value="0.01"/>
    <param name="srt" value="0.02"/>
    <param name="str" value="0.01"/>
    <param name="stt" value="0.02"/>

    <!-- Process a scan each time the robot translates this far  -->
    <param name="linearUpdate" value="0.1"/>

    <!-- Process a scan each time the robot rotates this far  -->
    <param name="angularUpdate" value="0.1"/>
    <param name="temporalUpdate" value="1.0"/>
    <param name="resampleThreshold" value="0.5"/>

    <!-- Number of particles in the filter. default 30        -->
    <param name="particles" value="30"/>

    <!-- Initial map size  -->
    <param name="xmin" value="-10.0"/>
    <param name="ymin" value="-10.0"/>
    <param name="xmax" value="10.0"/>
    <param name="ymax" value="10.0"/>

    <!-- Processing parameters (resolution of the map)  -->
    <param name="delta" value="0.025"/>
    <param name="llsamplerange" value="0.01"/>
    <param name="llsamplestep" value="0.01"/>
    <param name="lasamplerange" value="0.005"/>
    <param name="lasamplestep" value="0.005"/>
  </node> 

  <!-- Ground truth map Server -->
  <node name="map_server" pkg="map_server" type="map_server" args="$(arg map_file)">
    <remap from="map" to="map_groundtruth"/>
  </node>

</launch>
```

```console
roslaunch bme_ros_navigation spawn_robot.launch
```

```console
roslaunch bme_ros_navigation gmapping.launch
```

### Üres folyosó

Próbáljuk ki a GMappinget üres folyosón:
```console
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_empty.world' x:=-7 y:=2
```

```console
roslaunch bme_ros_navigation gmapping.launch map_file:='$(find bme_ros_navigation)/maps/corridor.yaml'
```

### Folyosó tárgyakkal
Ésnézzük meg ezt is tárgyakkal:
```console
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world' x:=-7 y:=2
```

## Map saver

Ha szeretnénk elmenteni a SLAM algoritmus által létrehozott térképet, akkor a map_server csomag map_saver node-ját tudjuk erre a célra használni. A map_server-rel találkoztunk már korábban is, ez adta a ground truth térképünket.

A map_saver abba a mappába menti a térképet, ahonnan indítjuk!
Csináljunk erre egy saved_maps mappát a maps-en belül.

rosrun map_server map_saver -f map

```console
david@DavidsLenovoX1:~/bme_catkin_ws/src/Week-7-8-Navigation/bme_ros_navigation/maps/saved_maps$ rosrun map_server map_saver -f map
[ INFO] [1614518022.653341900]: Waiting for the map
[ INFO] [1614518022.875830700, 563.990000000]: Received a 800 X 800 map @ 0.025 m/pix
[ INFO] [1614518022.875936200, 563.990000000]: Writing map occupancy data to map.pgm
[ INFO] [1614518022.933453800, 564.011000000]: Writing map occupancy data to map.yaml
[ INFO] [1614518022.933801100, 564.011000000]: Done

david@DavidsLenovoX1:~/bme_catkin_ws/src/Week-7-8-Navigation/bme_ros_navigation/maps/saved_maps$ ls
map.pgm  map.yaml
```


# Lokalizáció
## AMCL

```xml
<?xml version="1.0"?>
<launch>

  <!-- Map file for localization -->
  <arg name="map_file" default="$(find bme_ros_navigation)/maps/saved_maps/map.yaml"/>
  <!-- It can be an environmental variable, too -->
  <!--arg name="map_file" default="$(env AMCL_MAP_FILE)"/-->

  <!-- Map Server -->
  <node name="map_server" pkg="map_server" type="map_server" args="$(arg map_file)">
  </node>

  <!-- AMCL Node -->
  <arg name="initial_pose_x"  default="0.0"/>
  <arg name="initial_pose_y"  default="0.0"/>
  <arg name="initial_pose_a"  default="1.57"/>
  <node name="amcl" pkg="amcl" type="amcl" output="screen">
    <param name="odom_frame_id" value="odom"/>
    <param name="odom_model_type" value="diff-corrected"/>
    <param name="base_frame_id" value="base_link"/>
    <param name="global_frame_id" value="map"/>
    <param name="scan_topic" value="scan"/>
    <!-- If you choose to define initial pose here -->
    <param name="initial_pose_x" value="$(arg initial_pose_x)"/>
    <param name="initial_pose_y" value="$(arg initial_pose_y)"/>
    <param name="initial_pose_a" value="$(arg initial_pose_a)"/>
    <!-- Parameters for inital particle distribution -->
    <param name="initial_cov_xx" value="9.0"/>
    <param name="initial_cov_yy" value="9.0"/>
    <param name="initial_cov_aa" value="9.8"/>
    <!-- Dynamically adjusts particles for every iteration -->
    <param name="min_particles" value="500"/>
    <param name="max_particles" value="2000"/>
    <!-- Perform update parameters -->
    <param name="update_min_d" value="0.1"/>
    <param name="update_min_a" value="0.1"/>
    <param name="laser_model_type" value="likelihood_field"/>
    <param name="laser_max_range" value="-1.0"/>
    <param name="odom_alpha1" value="0.1"/>
    <param name="odom_alpha2" value="0.1"/>
    <param name="odom_alpha3" value="0.3"/>
    <param name="odom_alpha4" value="0.1"/>
    <param name="odom_alpha5" value="0.1"/>
    <!-- Transform tolerance needed on slower machines -->
    <param name="transform_tolerance" value="1.0"/>
  </node>

</launch>
```

roslaunch bme_ros_navigation spawn_robot.launch

roslaunch bme_ros_navigation amcl.launch map_file:='$(find bme_ros_navigation)/maps/map.yaml'

roslaunch bme_ros_navigation amcl.launch

roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world' x:=-7 y:=2

roslaunch bme_ros_navigation amcl.launch map_file:='$(find bme_ros_navigation)/maps/saved_maps/corridor.yaml'

# Navigáció

```xml
<?xml version="1.0"?>
<launch>

  <!-- Map file for localization -->
  <arg name="map_file" default="$(find bme_ros_navigation)/maps/saved_maps/map.yaml"/>

  <!-- Launch our Gazebo world -->
  <include file="$(find bme_ros_navigation)/launch/amcl.launch">
    <!-- all vars that included.launch requires must be set -->
    <arg name="map_file" value="$(arg map_file)" />
  </include>

  <!-- Move Base -->
  <node name="move_base" pkg="move_base" type="move_base" respawn="false" output="screen">
    <rosparam file="$(find bme_ros_navigation)/config/move_base_params.yaml" command="load" />
    <rosparam file="$(find bme_ros_navigation)/config/costmap_common_params.yaml" command="load" ns="global_costmap" />
    <rosparam file="$(find bme_ros_navigation)/config/costmap_common_params.yaml" command="load" ns="local_costmap" />
    <rosparam file="$(find bme_ros_navigation)/config/local_costmap_params.yaml" command="load" />
    <rosparam file="$(find bme_ros_navigation)/config/global_costmap_params.yaml" command="load" />
    <rosparam file="$(find bme_ros_navigation)/config/dwa_local_planner_params.yaml" command="load" />
  </node>

</launch>
```

roslaunch bme_ros_navigation spawn_robot.launch
roslaunch bme_ros_navigation navigation.launch

Costmap-ek törlése kézzel:
rosservice call /move_base/clear_costmaps "{}"

roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world' x:=-7 y:=2
roslaunch bme_ros_navigation navigation.launch map_file:='$(find bme_ros_navigation)/maps/saved_maps/corridor.yaml'

## Recovery akciók
ToDo

# Waypoint navigáció



## Grafikusan a follow_waypoints csomaggal

https://github.com/bergercookie/follow_waypoints

```xml
  <!-- Follow waypoints -->
  <param name="wait_duration" value="2.0"/>
  <param name="waypoints_to_follow_topic" value="/waypoint"/>
  <node pkg="follow_waypoints" type="follow_waypoints" name="follow_waypoints" output="screen">
    <param name="goal_frame_id" value="map"/>
  </node>
```

```xml
<remap from="initialpose" to="waypoint" />
```

rosservice call /path_ready {}

### Patrol mode

rosrun rqt_reconfigure rqt_reconfigure

## Waypoint navigáció C++ ROS node-ból

### RViz visual markers

---

# Twist mux

# Velocity smoother

# Exploration