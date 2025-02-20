################################################################################
# Copyright (c) 2024 - 2025 Advanced Micro Devices, Inc.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
#
################################################################################

cmake_minimum_required (VERSION 3.10)

# ROCM Path
if(DEFINED ENV{ROCM_PATH})
  set(ROCM_PATH $ENV{ROCM_PATH} CACHE PATH "Default ROCm installation path")
elseif(ROCM_PATH)
  message("-- INFO:ROCM_PATH Set -- ${ROCM_PATH}")
else()
  set(ROCM_PATH /opt/rocm CACHE PATH "Default ROCm installation path")
endif()
# Set AMD Clang as default compiler
if (NOT DEFINED CMAKE_CXX_COMPILER)
  set(CMAKE_C_COMPILER ${ROCM_PATH}/bin/amdclang)
  set(CMAKE_CXX_COMPILER ${ROCM_PATH}/bin/amdclang++)
endif()

project(jpegdecode)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED On)

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/../../cmake)
list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH}/hip ${ROCM_PATH})

find_package(HIP QUIET)
find_package(rocJPEG QUIET)
find_package(rocprofiler-register QUIET)

# find TurboJpeg
find_library(TurboJpeg_LIBRARIES NAMES libturbojpeg.so PATHS /usr/lib ${TURBO_JPEG_PATH}/lib ${TURBO_JPEG_PATH}/lib64 /opt/libjpeg-turbo/lib)
find_path(TurboJpeg_INCLUDE_DIRS NAMES turbojpeg.h PATHS /usr/include ${TURBO_JPEG_PATH}/include /opt/libjpeg-turbo/include)
# mark_as_advanced(TurboJpeg_INCLUDE_DIRS)

# Libjpeg
find_library(LIBJPEG_LIBRARIES NAMES libjpeg.so PATHS /usr/lib ${TURBO_JPEG_PATH}/lib ${TURBO_JPEG_PATH}/lib64 /opt/libjpeg-turbo/lib)
find_path(LIBJPEG_INCLUDE_DIR NAMES jpeglib.h PATHS /usr/include ${TURBO_JPEG_PATH}/include /opt/libjpeg-turbo/include)

if(TurboJpeg_LIBRARIES AND TurboJpeg_INCLUDE_DIRS AND LIBJPEG_LIBRARIES AND LIBJPEG_INCLUDE_DIR)
    set(TURBOJPEG_FOUND TRUE)
    message("-- ${White}Using TurboJpeg -- \n\tLibraries:${TurboJpeg_LIBRARIES} \n\tIncludes:${TurboJpeg_INCLUDE_DIRS}${ColourReset}")
endif()

if(TURBOJPEG_FOUND)
    # TurboJpeg
    include_directories (${TurboJpeg_INCLUDE_DIRS} ${LIBJPEG_INCLUDE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/..)
    set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} ${TurboJpeg_LIBRARIES} ${LIBJPEG_LIBRARIES})
else()
    message("-- ERROR!: ${PROJECT_NAME} excluded! please install all the dependencies and try again!")
    if (NOT TURBOJPEG_FOUND)
        message(FATAL_ERROR "-- ERROR!: turboJpeg Not Found! - please install turboJpeg!")
    endif()
endif()

if(HIP_FOUND AND ROCJPEG_FOUND AND rocprofiler-register_FOUND)
    # HIP
    set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} hip::host)
    # rocJPEG
    include_directories (${ROCJPEG_INCLUDE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/..)
    set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} ${ROCJPEG_LIBRARY})
    # std filesystem
    set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} stdc++fs)
    # rocprofiler-register
    set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} rocprofiler-register::rocprofiler-register)

    list(APPEND SOURCES ${PROJECT_SOURCE_DIR} jpegdecode.cpp)
    add_executable(${PROJECT_NAME} ${SOURCES})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++17")
    target_link_libraries(${PROJECT_NAME} ${LINK_LIBRARY_LIST})
else()
    message("-- ERROR!: ${PROJECT_NAME} excluded! please install all the dependencies and try again!")
    if (NOT HIP_FOUND)
        message(FATAL_ERROR "-- ERROR!: HIP Not Found! - please install ROCm and HIP!")
    endif()
    if (NOT ROCJPEG_FOUND)
        message(FATAL_ERROR "-- ERROR!: rocJPEG Not Found! - please install rocJPEG!")
    endif()
    if (NOT rocprofiler-register_FOUND)
        message(FATAL_ERROR "-- ERROR!: rocprofiler-register Not Found! - please install rocprofiler-register!")
    endif()
endif()
