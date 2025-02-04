cmake_minimum_required(VERSION 3.5.1)
project(car_gazebo_plugins)

find_package(catkin REQUIRED COMPONENTS
  gazebo_dev
  gazebo_msgs
  roscpp
  rospy
  nodelet
  std_msgs
  geometry_msgs
  urdf
  tf
  tf2_ros
  dynamic_reconfigure
  gazebo_plugins
)

# Through transitive dependencies in the packages above, gazebo_plugins depends
# on Simbody.  There is a bug in the Ubuntu Artful (17.10) version of the
# Simbody package where it includes /usr/lib/libblas.so and
# /usr/lib/liblapack.so in the CMake list of libraries even though neither of
# those two paths exist (they both really live in /usr/lib/<arch>-linux-gnu).
# We remove these two during build-time on artful below; this works because
# they both will get resolved to the proper paths during runtime linking.
find_program(LSB_RELEASE_EXEC lsb_release)
if(NOT LSB_RELEASE_EXEC STREQUAL "LSB_RELEASE_EXEC-NOTFOUND")
  execute_process(COMMAND ${LSB_RELEASE_EXEC} -cs
    OUTPUT_VARIABLE OS_CODENAME
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  if(OS_CODENAME STREQUAL "artful")
    list(FILTER catkin_LIBRARIES EXCLUDE REGEX "/usr/lib/libblas.so")
    list(FILTER catkin_LIBRARIES EXCLUDE REGEX "/usr/lib/liblapack.so")
    list(FILTER GAZEBO_LIBRARIES EXCLUDE REGEX "/usr/lib/libblas.so")
    list(FILTER GAZEBO_LIBRARIES EXCLUDE REGEX "/usr/lib/liblapack.so")
  endif()
endif()

include (FindPkgConfig)
if (PKG_CONFIG_FOUND)
  pkg_check_modules(XML libxml-2.0)
  pkg_check_modules(OGRE OGRE)
  pkg_check_modules(OGRE-Terrain OGRE-Terrain)
  pkg_check_modules(OGRE-Paging OGRE-Paging)
else()
  message(FATAL_ERROR "pkg-config is required; please install it")
endif()

find_package(Boost REQUIRED COMPONENTS thread)
if (CATKIN_ENABLE_TESTING)
  find_package(OpenCV COMPONENTS core imgproc calib3d highgui REQUIRED)
else()
  find_package(OpenCV COMPONENTS core highgui REQUIRED)
endif()

option(ENABLE_PROFILER "Enable Ignition Profiler" FALSE)
if (ENABLE_PROFILER)
  find_package(ignition-common3 QUIET)
  if (ignition-common3_FOUND)
    add_definitions("-DIGN_PROFILER_ENABLE=1" "-DIGN_PROFILER_REMOTERY=1")
    message("Profiler is active")
  else()
    message("Can't find Ignition common3. Profiler will not be actived")
    add_definitions("-DIGN_PROFILER_ENABLE=0" "-DIGN_PROFILER_REMOTERY=0")
  endif()
endif()

execute_process(COMMAND
  pkg-config --variable=plugindir OGRE
  OUTPUT_VARIABLE OGRE_PLUGIN_PATH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

catkin_python_setup()

generate_dynamic_reconfigure_options(
   cfg/Ackermann.cfg
)

include_directories(include
  ${Boost_INCLUDE_DIRS}
  ${catkin_INCLUDE_DIRS}
  ${OGRE_INCLUDE_DIRS}
  ${OGRE-Terrain_INCLUDE_DIRS}
  ${OGRE-Paging_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
)

link_directories(
  ${catkin_LIBRARY_DIRS}
  ${OGRE_LIBRARY_DIRS}
  ${OGRE-Terrain_LIBRARY_DIRS}
  ${OGRE-Paging_LIBRARY_DIRS}
)

catkin_package(
  INCLUDE_DIRS include
  LIBRARIES
  gazebo_ros_wheels_steerable
  
  CATKIN_DEPENDS
  gazebo_dev
  gazebo_msgs
  roscpp
  rospy
  nodelet
  std_msgs
  geometry_msgs
  urdf
  tf
  tf2_ros
  dynamic_reconfigure
  gazebo_plugins
)
add_dependencies(${PROJECT_NAME}_gencfg ${catkin_EXPORTED_TARGETS})

## Executables

add_definitions(-fPIC) # what is this for?

## Plugins

add_library(gazebo_ros_wheels_steerable src/gazebo_ros_wheels_steerable.cpp)
target_link_libraries(gazebo_ros_wheels_steerable ${catkin_LIBRARIES} ${Boost_LIBRARIES})

install(TARGETS
  gazebo_ros_wheels_steerable
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_GLOBAL_BIN_DESTINATION}
  )


install(DIRECTORY include/
   DESTINATION ${CATKIN_GLOBAL_INCLUDE_DESTINATION}
   FILES_MATCHING PATTERN "*.h"
)

catkin_install_python(PROGRAMS src/laserToCluster.py
	DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
catkin_install_python(PROGRAMS src/steer_between_2_objects.py
	DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
