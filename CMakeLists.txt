cmake_minimum_required(VERSION 2.6.0)

project(ICPCUDA)

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules" ${CMAKE_MODULE_PATH})

set(SOURCES_DIR src)

find_package(Eigen3 REQUIRED)
find_package(CUDA REQUIRED)
find_package(Sophus REQUIRED)
find_package(Pangolin)

include_directories(${CUDA_INCLUDE_DIRS})
include_directories(${EIGEN_INCLUDE_DIRS})
include_directories(${Sophus_INCLUDE_DIR})

file(GLOB cuda src/Cuda/*.cu)
file(GLOB containers src/Cuda/containers/*.cpp)

set(CUDA_ARCH_BIN "30 32 35 37 50 52 61" CACHE STRING "Specify 'real' GPU arch to build binaries for, BIN(PTX) format is supported. Example: 1.3 2.1(1.3) or 13 21(13)")
set(CUDA_ARCH_PTX "" CACHE STRING "Specify 'virtual' PTX arch to build PTX intermediate code for. Example: 1.0 1.2 or 10 12")

include(${CMAKE_MODULE_PATH}/CudaComputeTargetFlags.cmake)
APPEND_TARGET_ARCH_FLAGS()

set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}  "-Xcompiler;-fPIC;-std=c++11")
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} "--ftz=true;--prec-div=false;--prec-sqrt=false")

CUDA_COMPILE(cuda_objs ${cuda})

set(CMAKE_CXX_FLAGS "-O3 -msse2 -msse3 -Wall -std=c++11")

#######################################
#            ICPCUDA Lib              #
#######################################

set(ICPCUDA_LIBRARY "icpcuda")
add_library(${ICPCUDA_LIBRARY} ${SOURCES_DIR}/ICPOdometry.cpp
    ${cuda}
    ${cuda_objs}
    ${containers}
    )
target_link_libraries(${ICPCUDA_LIBRARY}
    ${Eigen_LIBRARIES}
    ${CUDA_LIBRARIES}
    )
install(TARGETS ${ICPCUDA_LIBRARY}
    DESTINATION lib/icpcuda)
install(DIRECTORY ${SOURCES_DIR}/ DESTINATION include/icpcuda
    FILES_MATCHING REGEX "^.*\\.(h|hpp)$"
    )
configure_file(
    "${CMAKE_MODULE_PATH}/cmake_uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)
add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

#######################################
#              ICP app                #
#######################################

if(${Pangolin_FOUND})
    include_directories(${Pangolin_INCLUDE_DIRS})
    add_executable(ICP ${SOURCES_DIR}/ICP.cpp )
    target_link_libraries(ICP
        ${ICPCUDA_LIBRARY}
        ${Pangolin_LIBRARIES}
    )
else(${Pangolin_FOUND})
    message(WARNING "Could not find Pangolin, which is required to build the sample application.
Please give some paths to CMake or make sure Pangolin is installed in your system.")
endif(${Pangolin_FOUND})
