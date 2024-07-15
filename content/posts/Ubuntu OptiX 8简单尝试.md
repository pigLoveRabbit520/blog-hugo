---
title: Ubuntu上OptiX8 简单尝试
author: pigRabbit
tags:
  - OptiX
  - Ubuntu
  - cuda
categories:
  - cuda
date: 2024-07-05 16:00:00
---

这是[Reference.pdf](https://raytracing-docs.nvidia.com/optix8/api/OptiX_API_Reference.pdf)

新建一个 CMake 项目，目录结构是这样的

```
optix_example/
├── CMakeLists.txt
├── src
    └── main.cpp
└── cmake
    └── FindOptiX80.cmake
```

`CMakeLists.txt`文件：

```cmake
cmake_minimum_required(VERSION 3.10)

project(OptiXExample)

set(CMAKE_CXX_STANDARD 17)

# Find OptiX package
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

find_package(OptiX80)

# Add CUDA

find_package(CUDAToolkit 12.0 REQUIRED)

# Include directories
include_directories(${OPTIX80_INCLUDE_DIR} ${CUDAToolkit_INCLUDE_DIRS})

# Source files
set(SOURCES src/main.cpp)

# Executable
add_executable(OptiXExample ${SOURCES})

# Link libraries
target_link_libraries(OptiXExample CUDA::cudart)
```

`FindOptix80.cmake`文件：

```cmake
# Looks for the environment variable:

# OPTIX80_PATH

# Sets the variables :

# OPTIX80_INCLUDE_DIR

# OptiX80_FOUND

set(OPTIX80_PATH $ENV{OPTIX80_PATH})

if("${OPTIX80_PATH}" STREQUAL "")
    if(WIN32)
        # Try finding it inside the default installation directory under Windows first.
        set(OPTIX80_PATH "C:/ProgramData/NVIDIA Corporation/OptiX SDK 8.0.0")
    else()
        # Adjust this if the OptiX SDK 8.0.0 installation is in a different location.
        set(OPTIX80_PATH "~/NVIDIA-OptiX-SDK-8.0.0-linux64")
    endif()
endif()

find_path(OPTIX80_INCLUDE_DIR optix_host.h ${OPTIX80_PATH}/include)

# message("OPTIX80_INCLUDE_DIR = " "${OPTIX80_INCLUDE_DIR}")
include(FindPackageHandleStandardArgs)

find_package_handle_standard_args(OptiX80 DEFAULT_MSG OPTIX80_INCLUDE_DIR)

mark_as_advanced(OPTIX80_INCLUDE_DIR)

# message("OptiX80_FOUND = " "${OptiX80_FOUND}")
```

上面的内容是参考 Github 上[OptiX_Apps](https://github.com/NVIDIA/OptiX_Apps/blob/master/3rdparty/CMake/FindOptiX80.cmake)，它读取了`OPTIX80_PATH` 环境变量，代表 Optix 在你机器上的位置，例如我的是`/home/rabbit/NVIDIA-OptiX-SDK-8.0.0-linux64-x86_64`。

main.cpp 文件：

```cpp
#include <optix.h>
#include <optix_stubs.h>
#include <optix_function_table_definition.h>
#include <iostream>
#include <string.h>
#include <vector>
#include <stdexcept>
#include <cuda.h>
#include <cuda_runtime_api.h>

#define OPTIX_CHECK(call)                                                                             \
    {                                                                                                 \
        OptixResult res = call;                                                                       \
        if (res != OPTIX_SUCCESS)                                                                     \
        {                                                                                             \
            fprintf(stderr, "Optix call (%s) failed with code %d (line %d)\n", #call, res, __LINE__); \
            exit(2);                                                                                  \
        }                                                                                             \
    }

void initOptix()
{
    // -------------------------------------------------------
    // check for available optix7 capable devices
    // -------------------------------------------------------
    cudaFree(0);
    int numDevices;
    cudaGetDeviceCount(&numDevices);
    if (numDevices == 0)
        throw std::runtime_error("#osc: no CUDA capable devices found!");
    std::cout << "#osc: found " << numDevices << " CUDA devices" << std::endl;

    // -------------------------------------------------------
    // initialize optix
    // -------------------------------------------------------
    OPTIX_CHECK(optixInit());
}

int main()
{
    // Initialize CUDA
    cudaFree(0);

    initOptix();
    return 0;
}
```
