cmake_minimum_required(VERSION 3.20)
# Prefer NVIDIA's CUDA toolkit in /usr/local/cuda over the distro's nvidia-cuda-toolkit
# (e.g. CUDA 12.4 on Ubuntu 26.04), which is found first via /usr/bin/nvcc and /usr/lib.
if(EXISTS "/usr/local/cuda/bin/nvcc")
  if(NOT DEFINED CMAKE_CUDA_COMPILER AND NOT DEFINED ENV{CUDACXX})
    set(CMAKE_CUDA_COMPILER "/usr/local/cuda/bin/nvcc")
  endif()
  if(NOT DEFINED CUDAToolkit_ROOT AND NOT DEFINED ENV{CUDAToolkit_ROOT})
    set(CUDAToolkit_ROOT "/usr/local/cuda")
  endif()
  if(NOT DEFINED CUDA_TOOLKIT_ROOT_DIR)
    set(CUDA_TOOLKIT_ROOT_DIR "/usr/local/cuda" CACHE PATH "CUDA toolkit root")
  endif()
endif()

project(isaac_ros_gpu_partitioning LANGUAGES C CXX CUDA)

# Enable CUDA
enable_language(CUDA)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake_auto REQUIRED)
ament_auto_find_build_dependencies()
# CUDAToolkit is a CMake package (not a ROS package), so find it explicitly.
find_package(CUDAToolkit REQUIRED)

# GpuPartitionTestNode
ament_auto_add_library(gpu_partition_test_node SHARED src/gpu_partition_test_node.cpp)
target_link_libraries(gpu_partition_test_node CUDA::cuda_driver CUDA::cudart)

install(TARGETS gpu_partition_test_node
  DESTINATION lib)

rclcpp_components_register_nodes(gpu_partition_test_node "nvidia::GpuPartitionTestNode")
set(node_plugins
  "${node_plugins}nvidia::GpuPartitionTestNode;$<TARGET_FILE:gpu_partition_test_node>\n")
set_target_properties(gpu_partition_test_node PROPERTIES
  BUILD_WITH_INSTALL_RPATH TRUE
  BUILD_RPATH_USE_ORIGIN TRUE
  INSTALL_RPATH_USE_LINK_PATH TRUE)

# Install the launch directory if present
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/launch")
  install(
    DIRECTORY launch/
    DESTINATION share/${PROJECT_NAME}/launch
  )
endif()

install(
  PROGRAMS
    scripts/start_cuda_mps.sh
    scripts/stop_cuda_mps.sh
  DESTINATION lib/${PROJECT_NAME}
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()

  find_package(launch_testing_ament_cmake REQUIRED)

  add_launch_test(test/test_started_server.py)
  add_launch_test(test/test_config_file.py)
  add_launch_test(test/test_config_array.py)
  add_launch_test(test/test_action.py)
  add_launch_test(test/test_multiple_process.py)
endif()

# Install Python modules
find_package(ament_cmake_python REQUIRED)
ament_python_install_package(isaac_ros_gpu_partitioning)

ament_package()
