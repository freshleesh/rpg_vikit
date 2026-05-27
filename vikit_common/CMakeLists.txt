CMAKE_MINIMUM_REQUIRED (VERSION 3.5)
SET(PROJECT_NAME vikit_common)
PROJECT(${PROJECT_NAME})
SET(CMAKE_BUILD_TYPE Release) # Release, RelWithDebInfo
SET(CMAKE_VERBOSE_MAKEFILE OFF)
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/CMakeModules/")
SET(USE_ROS False) # Set False if you want to build this package without Catkin

message("Current CPU archtecture: ${CMAKE_SYSTEM_PROCESSOR}")
# Force conda env includes into CXX_FLAGS to bypass CMake's implicit-directory filtering
# (conda compilers list $CONDA_PREFIX/include as implicit, so INCLUDE_DIRECTORIES() omits
# the -I flag; but -isysroot in the conda toolchain then prevents the compiler from finding
# headers there. Adding explicitly to CMAKE_CXX_FLAGS bypasses this.)
if(DEFINED ENV{CONDA_PREFIX})
  set(_conda_inc "-I$ENV{CONDA_PREFIX}/include")
else()
  set(_conda_inc "")
endif()
SET(CMAKE_CXX_FLAGS "-Wall -D_REENTRANT -Wno-unused-variable -Wno-unknown-pragmas ${_conda_inc}")
if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native -mmmx -msse -msse2 -msse3 -mssse3")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "(aarch64)|(arm64)|(ARM64)")
  # Apple Silicon / 64-bit ARM: no SSE, no armv8-a march override needed
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
else()
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
endif()

SET(CMAKE_CXX_STANDARD 17)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS} -O3 -fomit-frame-pointer -fno-signed-zeros -fno-math-errno -funroll-loops")

# Add plain cmake packages
FIND_PACKAGE(OpenCV REQUIRED)
FIND_PACKAGE(Eigen3 REQUIRED)
FIND_PACKAGE(Sophus REQUIRED)
FIND_PACKAGE(fmt REQUIRED)

# Resolve Sophus include directory
# Modern Sophus (1.x) only exports Sophus::Sophus target without Sophus_INCLUDE_DIRS variable.
# Use CONDA_PREFIX as fallback since this package is built inside a conda env.
if(TARGET Sophus::Sophus)
  get_target_property(_sophus_inc Sophus::Sophus INTERFACE_INCLUDE_DIRECTORIES)
endif()
if(NOT _sophus_inc)
  if(DEFINED ENV{CONDA_PREFIX})
    set(_sophus_inc "$ENV{CONDA_PREFIX}/include")
  elseif(Sophus_INCLUDE_DIR)
    set(_sophus_inc "${Sophus_INCLUDE_DIR}")
  endif()
endif()
message(STATUS "Sophus include dir: ${_sophus_inc}")

# Include dirs
INCLUDE_DIRECTORIES(
  include
  ${EIGEN3_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
  ${_sophus_inc}
)

IF(USE_ROS)
  FIND_PACKAGE(catkin REQUIRED COMPONENTS roscpp cmake_modules)
  LIST(APPEND INCLUDE_DIRECTORIES ${catkin_INCLUDE_DIRS})
  catkin_package(
    DEPENDS Eigen OpenCV Sophus
    CATKIN_DEPENDS roscpp
    INCLUDE_DIRS include
    LIBRARIES ${PROJECT_NAME}
  )
ELSE()
  SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
  SET(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
ENDIF()

# Set Sourcefiles
LIST(APPEND SOURCEFILES src/atan_camera.cpp
                        src/omni_camera.cpp
                        src/math_utils.cpp
                        src/vision.cpp
                        src/performance_monitor.cpp
                        src/robust_cost.cpp
                        src/user_input_thread.cpp
                        src/pinhole_camera.cpp
                        src/equidistant_camera.cpp
                        src/polynomial_camera.cpp
                        src/homography.cpp
                        src/img_align.cpp)

# Create vikit library
ADD_LIBRARY(${PROJECT_NAME} SHARED ${SOURCEFILES})
TARGET_LINK_LIBRARIES(${PROJECT_NAME}
  ${OpenCV_LIBS}
  Sophus::Sophus
  fmt::fmt)

IF(USE_ROS)
  TARGET_LINK_LIBRARIES(${PROJECT_NAME} ${catkin_LIBRARIES})
ENDIF()

# Tests
ADD_EXECUTABLE(test_vk_common_camera test/test_camera.cpp)
TARGET_LINK_LIBRARIES(test_vk_common_camera ${PROJECT_NAME} ${OpenCV_LIBS})

ADD_EXECUTABLE(test_vk_common_triangulation test/test_triangulation.cpp)
TARGET_LINK_LIBRARIES(test_vk_common_triangulation ${PROJECT_NAME} ${OpenCV_LIBS})

ADD_EXECUTABLE(test_vk_common_patch_score test/test_patch_score.cpp)
TARGET_LINK_LIBRARIES(test_vk_common_patch_score ${PROJECT_NAME} ${OpenCV_LIBS})


################################################################################
# Create the vikit_commonConfig.cmake file for other cmake projects.
IF(NOT USE_ROS)
  # In CMake 3.6 and later, reading the LOCATION property from a target is no longer allowed.
  # GET_TARGET_PROPERTY( FULL_LIBRARY_NAME ${PROJECT_NAME} LOCATION )
  set(VIKIT_COMMON_LOCATION $<TARGET_FILE:vikit_common>)
  SET(vikit_common_LIBRARIES ${FULL_LIBRARY_NAME} )
  SET(vikit_common_LIBRARY_DIR ${PROJECT_BINARY_DIR} )
  SET(vikit_common_INCLUDE_DIR "${PROJECT_SOURCE_DIR}/include")
  CONFIGURE_FILE( ${CMAKE_CURRENT_SOURCE_DIR}/vikit_commonConfig.cmake.in
      ${CMAKE_CURRENT_BINARY_DIR}/vikit_commonConfig.cmake @ONLY IMMEDIATE )
  export( PACKAGE vikit_common )

  INSTALL(DIRECTORY include/vikit DESTINATION ${CMAKE_INSTALL_PREFIX}/include FILES_MATCHING PATTERN "*.h" )
  INSTALL(TARGETS ${PROJECT_NAME} DESTINATION ${CMAKE_INSTALL_PREFIX}/lib )

  # Install a cmake config pointing to the installed library so downstream
  # find_package(vikit_common) works after a colcon build.
  set(_install_cmake_dir "${CMAKE_INSTALL_PREFIX}/lib/cmake/vikit_common")
  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/vikit_commonInstallConfig.cmake"
    "get_filename_component(_VKCOMMON_PREFIX \"\${CMAKE_CURRENT_LIST_DIR}/../../../\" ABSOLUTE)\n"
    "set(vikit_common_INCLUDE_DIR  \"\${_VKCOMMON_PREFIX}/include\")\n"
    "set(vikit_common_INCLUDE_DIRS \"\${_VKCOMMON_PREFIX}/include\")\n"
    "set(vikit_common_LIBRARY  \"\${_VKCOMMON_PREFIX}/lib/libvikit_common${CMAKE_SHARED_LIBRARY_SUFFIX}\")\n"
    "set(vikit_common_LIBRARIES \"\${_VKCOMMON_PREFIX}/lib/libvikit_common${CMAKE_SHARED_LIBRARY_SUFFIX}\")\n"
    "set(vikit_common_LIBRARY_DIR \"\${_VKCOMMON_PREFIX}/lib\")\n"
    "set(vikit_common_LIBRARY_DIRS \"\${_VKCOMMON_PREFIX}/lib\")\n"
    "if(NOT TARGET vikit_common::vikit_common)\n"
    "  add_library(vikit_common::vikit_common SHARED IMPORTED)\n"
    "  set_target_properties(vikit_common::vikit_common PROPERTIES\n"
    "    IMPORTED_LOCATION \"\${vikit_common_LIBRARY}\"\n"
    "    INTERFACE_INCLUDE_DIRECTORIES \"\${vikit_common_INCLUDE_DIRS}\"\n"
    "  )\n"
    "endif()\n"
    "set(vikit_common_FOUND TRUE)\n"
  )
  INSTALL(FILES "${CMAKE_CURRENT_BINARY_DIR}/vikit_commonInstallConfig.cmake"
    DESTINATION "${_install_cmake_dir}"
    RENAME "vikit_commonConfig.cmake"
  )
ENDIF()