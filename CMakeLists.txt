cmake_minimum_required(VERSION 3.7)

project(argon2-gpu CXX)
set(BINARY_INSTALL_DIR /usr/local/bin)
set(LIBRARY_INSTALL_DIR /usr/local/lib)
set(INCLUDE_INSTALL_DIR /usr/local/include)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR})
if(MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++11")
endif()
if(WIN32)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/lib)
    include(GNUInstallDirs)
endif()

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(CUDA_ARCH "sm_75" CACHE STRING "Specify the CUDA architecture.")

set(CUDA_FOUND FALSE)

if(NOT NO_CUDA)
    find_package(CUDA)
endif()

if(CUDA_FOUND)
    message("INFO: Using CUDA version ${CUDA_VERSION}")
    add_definitions(-DHAVE_CUDA=1)
else()
    message("FATAL_ERROR: Building without CUDA support")
    add_definitions(-DHAVE_CUDA=0)
endif()

if(CUDA_FOUND)
    set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS};
        -std=c++11;-O3;--ptxas-options=-v;-arch ${CUDA_ARCH};-lineinfo
    )
endif()

add_subdirectory(nguyen/ext/argon2)

add_library(argon2-gpu-common SHARED
nguyen/lib/argon2-gpu-common/argon2params.cpp
nguyen/lib/argon2-gpu-common/blake2b.cpp
)
target_include_directories(argon2-gpu-common INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/nguyen/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(argon2-gpu-common PRIVATE
${CMAKE_CURRENT_SOURCE_DIR}/nguyen/include/argon2-gpu-common
${CMAKE_CURRENT_SOURCE_DIR}/nguyen/lib/argon2-gpu-common
)

if(CUDA_FOUND)
    cuda_add_library(argon2-cuda SHARED
    nguyen/lib/argon2-cuda/cudaexception.cpp
    nguyen/lib/argon2-cuda/device.cpp
    nguyen/lib/argon2-cuda/globalcontext.cpp
    nguyen/lib/argon2-cuda/programcontext.cpp
    nguyen/lib/argon2-cuda/processingunit.cpp
    nguyen/lib/argon2-cuda/kernelrunner.cu
    )
else()
    add_library(argon2-cuda SHARED
    nguyen/lib/argon2-cuda/nocuda.cpp
    )
endif()

target_include_directories(argon2-cuda PRIVATE
nguyen/include/argon2-cuda
nguyen/lib/argon2-cuda
)
target_include_directories(argon2-cuda INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/nguyen/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(argon2-cuda argon2-gpu-common)

add_library(argon2-opencl SHARED
nguyen/lib/argon2-opencl/device.cpp
nguyen/lib/argon2-opencl/globalcontext.cpp
nguyen/lib/argon2-opencl/kernelloader.cpp
nguyen/lib/argon2-opencl/programcontext.cpp
nguyen/lib/argon2-opencl/processingunit.cpp
nguyen/lib/argon2-opencl/kernelrunner.cpp
)
target_include_directories(argon2-opencl INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/nguyen/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(argon2-opencl PRIVATE
nguyen/include/argon2-opencl
nguyen/lib/argon2-opencl
)

if (UNIX AND NOT APPLE)
    target_link_libraries(argon2-opencl
        argon2-gpu-common -lOpenCL
    )
endif()
if (APPLE)
    target_link_libraries(argon2-opencl
        argon2-gpu-common "-framework OpenCL"
    )
endif()
if (WIN32)
    target_link_libraries(argon2-opencl
        argon2-gpu-common OpenCL.lib
    )
endif()

add_executable(1011today
nguyen/src/argon2-gpu-bench/cpuexecutive.cpp
nguyen/src/argon2-gpu-bench/cudaexecutive.cpp
nguyen/src/argon2-gpu-bench/openclexecutive.cpp
nguyen/src/argon2-gpu-bench/benchmark.cpp
nguyen/src/argon2-gpu-bench/main.cpp
)
target_include_directories(1011today PRIVATE nguyen/src/argon2-gpu-bench)
target_link_libraries(1011today
    argon2-cuda argon2-opencl argon2 -lOpenCL
)

install(
    TARGETS argon2-gpu-common argon2-opencl argon2-cuda
    DESTINATION ${LIBRARY_INSTALL_DIR}
)
install(FILES
nguyen/include/argon2-gpu-common/argon2-common.h
nguyen/include/argon2-gpu-common/argon2params.h
nguyen/include/argon2-opencl/cl.hpp
nguyen/include/argon2-opencl/opencl.h
nguyen/include/argon2-opencl/device.h
nguyen/include/argon2-opencl/globalcontext.h
nguyen/include/argon2-opencl/programcontext.h
nguyen/include/argon2-opencl/processingunit.h
nguyen/include/argon2-opencl/kernelrunner.h
nguyen/include/argon2-cuda/cudaexception.h
nguyen/include/argon2-cuda/kernelrunner.h
nguyen/include/argon2-cuda/device.h
nguyen/include/argon2-cuda/globalcontext.h
nguyen/include/argon2-cuda/programcontext.h
nguyen/include/argon2-cuda/processingunit.h
    DESTINATION ${INCLUDE_INSTALL_DIR}
)
install(
    TARGETS 1011today
    DESTINATION ${BINARY_INSTALL_DIR}
)
