[//]: # (Image References)

[image1]: ./assets/gazebo_world.png "Gazebo"
[image2]: ./assets/gazebo_corridor_empty.png "Gazebo"
[image3]: ./assets/gazebo_corridor_features.png "Gazebo"
[image4]: ./assets/gazebo_corridor_robot_1.png "Gazebo"
[image5]: ./assets/gazebo_corridor_robot_2.png "Gazebo"

# 7. - 8. hét - ROS navigáció

# Hova fogunk eljutni?

<a href="https://youtu.be/8bnjzPTNLfc"><img height="400" src="./assets/youtube1.png"></a>
<a href="https://youtu.be/-YCcQZmKJtY"><img height="400" src="./assets/youtube2.png"></a>

# Tartalomjegyzék
1. [Kezdőcsomag](#Kezdőcsomag)  
2. [Szenzorok 1](#Szenzorok-1)  
2.1. [Kamera](#Kamera)  
2.2. [IMU](#IMU)  
2.3. [GPS](#GPS)
3. [GPS waypoint követés](#GPS-waypoint-követés)
4. [Szenzorok 2](#Szenzorok-2)  
4.1. [Lidar](#Lidar)  
4.2. [Velodyne VLP16 lidar](#Velodyne-VLP16-lidar)  
4.3. [RGBD kamera](#RGBD-kamera)
5. [Képfeldolgozás ROS-ban OpenCV-vel](#Képfeldolgozás-ROS-ban-OpenCV-vel)

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

A robot ilyekor nem a (0,0) pozícióban indul, mert a `spawn_robot.launch` fájlban ezek az alapértelmezett értékek:
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

 

 roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_empty.world' x:=-7 y:=2 

# IMU Odometria szenzorfúzió
ToDo

# Hector SLAM
roslaunch bme_ros_navigation spawn_robot.launch
roslaunch bme_ros_navigation hector_slam.launch

## Corridor
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_empty.world' x:=-7 y:=2 
roslaunch bme_ros_navigation hector_slam.launch map_file:='$(find bme_ros_navigation)/maps/corridor_hector.yaml'

## Corridor with features
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world' x:=-7 y:=2

# GMapping
roslaunch bme_ros_navigation spawn_robot.launch
roslaunch bme_ros_navigation gmapping.launch

# Corridor
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_empty.world' x:=-7 y:=2
roslaunch bme_ros_navigation gmapping.launch map_file:='$(find bme_ros_navigation)/maps/corridor.yaml'

# Corridor with features
roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world' x:=-7 y:=2

# Save map

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



# AMCL
roslaunch bme_ros_navigation spawn_robot.launch

roslaunch bme_ros_navigation amcl.launch map_file:='$(find bme_ros_navigation)/maps/map.yaml'

roslaunch bme_ros_navigation amcl.launch

roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world' x:=-7 y:=2

roslaunch bme_ros_navigation amcl.launch map_file:='$(find bme_ros_navigation)/maps/saved_maps/corridor.yaml'

# Navigation
roslaunch bme_ros_navigation spawn_robot.launch
roslaunch bme_ros_navigation navigation.launch

Costmap-ek törlése kézzel:
rosservice call /move_base/clear_costmaps "{}"

roslaunch bme_ros_navigation spawn_robot.launch world:='$(find bme_ros_navigation)/worlds/20m_corridor_features.world' x:=-7 y:=2
roslaunch bme_ros_navigation navigation.launch map_file:='$(find bme_ros_navigation)/maps/saved_maps/corridor.yaml'

## Recovery
ToDo

# Waypoint navigation

https://github.com/bergercookie/follow_waypoints

rosservice call /path_ready {}

## Patrol mode

# Waypoint navigation C++ code

## RViz visual markers

---

# Twist mux

# Velocity smoother

# Exploration