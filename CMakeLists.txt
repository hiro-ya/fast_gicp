cmake_minimum_required(VERSION 3.10.0)
project(fast_gicp)

option(BUILD_VGICP_CUDA "Build GPU-powered VGICP" OFF)
option(BUILD_apps "Build application programs" ON)
option(BUILD_test "Build test programs" OFF)

add_definitions(-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2)
set(CMAKE_C_FLAGS "-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2")
set(CMAKE_CXX_FLAGS "-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2")

set(CMAKE_BUILD_TYPE "Release")

find_package(catkin)
find_package(PCL REQUIRED)
find_package(Eigen3 REQUIRED)

find_package(OpenMP)
if (OPENMP_FOUND)
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

if(BUILD_VGICP_CUDA)
  find_package(CUDA REQUIRED)
  include_directories(${CUDA_INCLUDE_DIRS})
  link_directories(${CUDA_LIBRARY_DIRS})
endif()

###################################
## catkin specific configuration ##
###################################
if(catkin_FOUND)
  catkin_package(
    INCLUDE_DIRS include
    LIBRARIES fast_gicp
  )
endif()

###########
## Build ##
###########

add_library(fast_gicp
  src/fast_gicp/gicp/lsq_registration.cpp
  src/fast_gicp/gicp/fast_gicp.cpp
  src/fast_gicp/gicp/fast_gicp_st.cpp
  src/fast_gicp/gicp/fast_vgicp.cpp
)
target_link_libraries(fast_gicp
  ${PCL_LIBRARIES}
)
target_include_directories(fast_gicp PUBLIC
  include
  ${PCL_INCLUDE_DIRS}
  ${EIGEN3_INCLUDE_DIR}
)

### APPS ###
if(BUILD_apps)
  add_executable(gicp_align src/align.cpp)
  add_dependencies(gicp_align fast_gicp)
  target_link_libraries(gicp_align
    ${PCL_LIBRARIES}
    fast_gicp
  )

  add_executable(gicp_kitti src/kitti.cpp)
  add_dependencies(gicp_kitti fast_gicp)
  target_link_libraries(gicp_kitti
    ${PCL_LIBRARIES}
    fast_gicp
  )
endif()

### CUDA ###
if(BUILD_VGICP_CUDA)
  set(CUDA_NVCC_FLAGS "--expt-relaxed-constexpr")
  add_definitions(-DUSE_VGICP_CUDA)

  cuda_add_library(fast_vgicp_cuda
    src/fast_gicp/cuda/fast_vgicp_cuda.cu
    src/fast_gicp/cuda/brute_force_knn.cu
    src/fast_gicp/cuda/covariance_estimation.cu
    src/fast_gicp/cuda/covariance_estimation_rbf.cu
    src/fast_gicp/cuda/covariance_regularization.cu
    src/fast_gicp/cuda/gaussian_voxelmap.cu
    src/fast_gicp/cuda/find_voxel_correspondences.cu
    src/fast_gicp/cuda/compute_derivatives.cu
    src/fast_gicp/cuda/compute_mahalanobis.cu

  )
  target_include_directories(fast_vgicp_cuda PRIVATE
    include
    thirdparty/Eigen
    thirdparty/nvbio
  )
  cuda_add_cublas_to_target(fast_vgicp_cuda)

  # add vgicp_cuda to libfast_gicp
  target_sources(fast_gicp PRIVATE
    src/fast_gicp/gicp/fast_vgicp_cuda.cpp
  )
  target_link_libraries(fast_gicp
    fast_vgicp_cuda
  )
  add_dependencies(fast_gicp fast_vgicp_cuda)
endif()

### TEST ###
if(BUILD_test)
  find_package(GTest REQUIRED)

  add_executable(gicp_test src/test/gicp_test.cpp)
  add_dependencies(gicp_test fast_gicp)
  target_link_libraries(gicp_test ${GTEST_LIBRARIES} ${PCL_LIBRARIES} fast_gicp)
  gtest_add_tests(TARGET gicp_test WORKING_DIRECTORY ${CMAKE_SOURCE_DIR} EXTRA_ARGS "${CMAKE_SOURCE_DIR}/data")
endif()