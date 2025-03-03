# ##############################################################################
# Copyright (c) 2024 Advanced Micro Devices, Inc.
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
# ##############################################################################

cmake_minimum_required(VERSION 3.10)

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
  set(CMAKE_CXX_COMPILER ${ROCM_PATH}/bin/amdclang++)
endif()

set(VERSION "0.7.0")
set(CMAKE_CXX_STANDARD 17)

# Set Project Version and Language
project(rocjpeg VERSION ${VERSION} LANGUAGES CXX)

set(CMAKE_INSTALL_LIBDIR "lib" CACHE STRING "Library install directory")
set(CMAKE_INSTALL_INCLUDEDIR "include" CACHE STRING "Include install directory")
include(GNUInstallDirs)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})

find_program(MAKE_NSIS_EXE makensis)
find_program(RPMBUILD_EXE rpmbuild)
find_program(DPKG_EXE dpkg)

# avoid setting the default installation path to /usr/local
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX ${ROCM_PATH} CACHE PATH "rocJPEG default installation path" FORCE)
endif(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)

set(DEFAULT_BUILD_TYPE "Release")
set(ENHANCED_MESSAGE ON)

if(ENHANCED_MESSAGE)
  string(ASCII 27 Esc)
  set(ColourReset "${Esc}[m")
  set(Red "${Esc}[31m")
  set(Green "${Esc}[32m")
  set(Yellow "${Esc}[33m")
  set(Blue "${Esc}[34m")
  set(BoldBlue "${Esc}[1;34m")
  set(Magenta "${Esc}[35m")
  set(Cyan "${Esc}[36m")
  set(White "${Esc}[37m")
endif()
message("-- ${BoldBlue}rocJPEG Version -- ${VERSION}${ColourReset}")
message("-- ${BoldBlue}rocJPEG Install Path -- ${CMAKE_INSTALL_PREFIX}${ColourReset}")

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
list(APPEND CMAKE_PREFIX_PATH ${ROCM_PATH} ${ROCM_PATH}/hip)

# rocJPEG Default Options
option(BUILD_WITH_AMD_ADVANCE "Build rocJPEG for advanced AMD GPU Architecture" OFF)

# rocJPEG Build Type
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}" CACHE STRING "rocJPEG Default Build Type" FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
endif()
if(CMAKE_BUILD_TYPE MATCHES Debug)
  # -O0 -- Don't Optimize output file 
  # -g  -- generate debugging information, generate debugging information, dwarf-4 for making valgrind work
  # -Og -- Optimize for debugging experience rather than speed or size
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0 -gdwarf-4 -Og")
else()
  # -O3       -- Optimize output file 
  # -DNDEBUG  -- turn off asserts 
  # -fPIC     -- Generate position-independent code if possible
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -DNDEBUG -fPIC")
endif()
message("-- ${BoldBlue}rocJPEG Build Type -- ${CMAKE_BUILD_TYPE}${ColourReset}")

# Set supported GPU Targets
set(DEFAULT_GPU_TARGETS "gfx908;gfx90a;gfx942;gfx1030;gfx1031;gfx1032;gfx1100;gfx1101;gfx1102;gfx1200;gfx1201")

# Set AMD GPU_TARGETS
if((AMDGPU_TARGETS OR DEFINED ENV{AMDGPU_TARGETS}) AND (NOT GPU_TARGETS))
  message("-- ${Red}${PROJECT_NAME} DEPRECATION -- AMDGPU_TARGETS use is deprecated. Use GPU_TARGETS${ColourReset}")
  if(DEFINED ENV{AMDGPU_TARGETS})
    set(GPU_TARGETS $ENV{AMDGPU_TARGETS} CACHE STRING "List of specific machine types for library to target")
  else()
    set(GPU_TARGETS ${AMDGPU_TARGETS})
  endif()
endif()

if(DEFINED ENV{GPU_ARCHS})
  set(GPU_TARGETS $ENV{GPU_ARCHS} CACHE STRING "List of specific machine types for library to target")
elseif(GPU_TARGETS)
  message("-- ${White}${PROJECT_NAME} -- GPU_TARGETS set with -D option${ColourReset}")
else()
  set(GPU_TARGETS "${DEFAULT_GPU_TARGETS}" CACHE STRING "List of specific machine types for library to target")
endif()
message("-- ${White}${PROJECT_NAME} -- AMD GPU_TARGETS: ${GPU_TARGETS}${ColourReset}")

find_package(HIP QUIET)
find_package(Libva QUIET)

if(HIP_FOUND AND Libva_FOUND)
  # HIP
  set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} hip::device)
  # LibVA
  include_directories(${LIBVA_INCLUDE_DIR})
  set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} ${LIBVA_LIBRARY})
  set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} ${LIBVA_DRM_LIBRARY})

  #filesystem: c++ compilers less than equal to 8.5 need explicit link with stdc++fs
  if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS_EQUAL "8.5")
    set(LINK_LIBRARY_LIST ${LINK_LIBRARY_LIST} stdc++fs)
  endif()

  # local include files
  include_directories(api src)
  # source files
  file(GLOB_RECURSE SOURCES "./src/*.cpp")
  # rocJPEG.so
  add_library(${PROJECT_NAME} SHARED ${SOURCES})

  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++17")
  target_link_libraries(${PROJECT_NAME} ${LINK_LIBRARY_LIST})

  set_target_properties(${PROJECT_NAME} PROPERTIES POSITION_INDEPENDENT_CODE ON)
  set_target_properties(${PROJECT_NAME} PROPERTIES LINKER_LANGUAGE CXX)
  set_target_properties(${PROJECT_NAME} PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})

  # install rocJPEG libs -- {ROCM_PATH}/lib
  install(TARGETS ${PROJECT_NAME} LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT runtime NAMELINK_SKIP)
  install(TARGETS ${PROJECT_NAME} LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT dev NAMELINK_ONLY)
  install(TARGETS ${PROJECT_NAME} LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT asan)
  # install rocJPEG include files -- {ROCM_PATH}/include/rocJPEG
  install(FILES api/rocjpeg.h api/rocjpeg_version.h
          DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME} COMPONENT dev)
  # install rocJPEG samples -- {ROCM_PATH}/share/rocJPEG
  install(DIRECTORY cmake DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME} COMPONENT dev)

  install(FILES samples/jpegDecode/CMakeLists.txt samples/jpegDecode/jpegdecode.cpp samples/jpegDecode/README.md DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples/jpegDecode COMPONENT dev)
  install(FILES samples/jpegDecodePerf/CMakeLists.txt samples/jpegDecodePerf/jpegdecodeperf.cpp samples/jpegDecodePerf/README.md DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples/jpegDecodePerf COMPONENT dev)
  install(FILES samples/jpegDecodeBatched/CMakeLists.txt samples/jpegDecodeBatched/jpegdecodebatched.cpp samples/jpegDecodeBatched/README.md DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples/jpegDecodeBatched COMPONENT dev)
  install(FILES samples/rocjpeg_samples_utils.h DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/samples COMPONENT dev)
  install(DIRECTORY data/images DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/ COMPONENT dev)
  # install license information - {ROCM_PATH}/share/doc/rocJPEG
  set(CPACK_RESOURCE_FILE_LICENSE  "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
  install(FILES ${CPACK_RESOURCE_FILE_LICENSE} DESTINATION ${CMAKE_INSTALL_DOCDIR} COMPONENT runtime)
  install(FILES ${CPACK_RESOURCE_FILE_LICENSE} DESTINATION ${CMAKE_INSTALL_DOCDIR}-asan COMPONENT asan)
  install(FILES ${CPACK_RESOURCE_FILE_LICENSE} DESTINATION ${CMAKE_INSTALL_DOCDIR}-dev COMPONENT dev)
  install(FILES ${CPACK_RESOURCE_FILE_LICENSE} DESTINATION ${CMAKE_INSTALL_DOCDIR}-test COMPONENT test)
  # install test cmake
  install(FILES test/CMakeLists.txt DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/test COMPONENT test)

  message("-- ${White}AMD ROCm rocJPEG -- CMAKE_CXX_FLAGS:${CMAKE_CXX_FLAGS}${ColourReset}")
  message("-- ${White}AMD ROCm rocJPEG -- Link Libraries: ${LINK_LIBRARY_LIST}${ColourReset}")

  # make test with CTest
  enable_testing()
  include(CTest)
  add_subdirectory(samples)

  # set package information
  set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
  set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
  set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
  set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
  set(CPACK_PACKAGE_LICENSE "MIT")
  set(CPACK_PACKAGE_CONTACT "rocJPEG Support <mivisionx.support@amd.com>")
  set(CPACK_PACKAGE_VENDOR "AMD ROCm")
  set(CPACK_PACKAGE_GROUP "Development/Tools")
  set(CPACK_PACKAGE_HOMEPAGE "https://github.com/ROCm/rocJPEG")

  if(ENABLE_ASAN_PACKAGING)
    set(CPACK_DEBIAN_PACKAGE_DESCRIPTION "AMD rocJPEG address sanitizer libraries")
    set(CPACK_RPM_PACKAGE_SUMMARY "AMD rocJPEG address sanitizer libraries")
  else()
    set(CPACK_DEBIAN_PACKAGE_DESCRIPTION "AMD rocJPEG is a high performance JPEG decode SDK for AMD GPUs")
    set(CPACK_RPM_PACKAGE_SUMMARY "AMD rocJPEG is a high performance JPEG decode SDK for AMD GPUs")
  endif()

  if(DEFINED ENV{ROCM_LIBPATCH_VERSION})
    set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION}.$ENV{ROCM_LIBPATCH_VERSION}")
  endif()

  # Find Ubuntu 22.04 - Add libstdc++-12-dev package deps for Dev Package
  file(READ "/etc/os-release" OS_RELEASE)
  string(REGEX MATCH "22.04" UBUNTU_22_FOUND ${OS_RELEASE})

  # Set the dependent packages
  set(rocJPEG_DEBIAN_PACKAGE_LIST  "rocm-hip-runtime, libva2-amdgpu, libva-amdgpu-drm2, libva-amdgpu-wayland2, libva-amdgpu-x11-2, mesa-amdgpu-va-drivers")
  set(rocJPEG_RPM_PACKAGE_LIST     "rocm-hip-runtime, (libva >= 2.16.0 or libva2 >= 2.16.0 or libva-amdgpu), mesa-amdgpu-va-drivers")
  set(rocJPEG_DEBIAN_DEV_PACKAGE_LIST  "rocm-hip-runtime-dev, libva-amdgpu-dev, pkg-config")
  if(UBUNTU_22_FOUND)
    set(rocJPEG_DEBIAN_DEV_PACKAGE_LIST "${rocJPEG_DEBIAN_DEV_PACKAGE_LIST}, libstdc++-12-dev")
  endif()
  set(rocJPEG_RPM_DEV_PACKAGE_LIST  "rocm-hip-runtime-devel, libva-amdgpu-devel, pkg-config")

  # '%{?dist}' breaks manual builds on debian systems due to empty Provides
  execute_process(
    COMMAND rpm --eval %{?dist}
    RESULT_VARIABLE PROC_RESULT
    OUTPUT_VARIABLE EVAL_RESULT
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  if(PROC_RESULT EQUAL "0" AND NOT EVAL_RESULT STREQUAL "")
    string(APPEND CPACK_RPM_PACKAGE_RELEASE "%{?dist}")
  endif()

  # set package 
  set(CPACK_PACKAGING_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX})
  set(CPACK_DEBIAN_PACKAGE_HOMEPAGE ${CPACK_PACKAGE_HOMEPAGE})
  set(CPACK_RPM_PACKAGE_URL ${CPACK_PACKAGE_HOMEPAGE})
  set(CPACK_RPM_PACKAGE_AUTOREQPROV "no")
  # Debian package
  set(CPACK_DEB_COMPONENT_INSTALL ON)
  set(CPACK_DEBIAN_RUNTIME_PACKAGE_NAME "${PROJECT_NAME}")
  set(CPACK_DEBIAN_RUNTIME_PACKAGE_DEPENDS "rocm-core, ${rocJPEG_DEBIAN_PACKAGE_LIST}")
  set(CPACK_DEBIAN_DEV_PACKAGE_NAME "${PROJECT_NAME}-dev")
  set(CPACK_DEBIAN_DEV_PACKAGE_DEPENDS 
  "rocm-core, ${PROJECT_NAME}, ${rocJPEG_DEBIAN_DEV_PACKAGE_LIST}")
  # Debian package - specific variable for ASAN
  set(CPACK_DEBIAN_ASAN_PACKAGE_NAME "${PROJECT_NAME}-asan" )
  set(CPACK_DEBIAN_ASAN_PACKAGE_DEPENDS "rocm-core-asan, ${rocJPEG_DEBIAN_PACKAGE_LIST}" )
  # Debian package - Test
  set(CPACK_DEBIAN_TEST_PACKAGE_NAME "${PROJECT_NAME}-test" )
  set(CPACK_DEBIAN_TEST_PACKAGE_DEPENDS "rocm-core, ${CPACK_DEBIAN_DEV_PACKAGE_NAME}" )
  # RPM package
  set(CPACK_RPM_COMPONENT_INSTALL ON)
  set(CPACK_RPM_RUNTIME_PACKAGE_NAME "${PROJECT_NAME}")
  set(CPACK_RPM_RUNTIME_PACKAGE_REQUIRES "rocm-core, ${rocJPEG_RPM_PACKAGE_LIST}")
  set(CPACK_RPM_RUNTIME_PACKAGE_OBSOLETES "${PROJECT_NAME}")
  set(CPACK_RPM_DEV_PACKAGE_NAME "${PROJECT_NAME}-devel")
  set(CPACK_RPM_DEV_PACKAGE_REQUIRES "rocm-core, ${PROJECT_NAME}, ${rocJPEG_RPM_DEV_PACKAGE_LIST}")
  set(CPACK_RPM_DEV_PACKAGE_OBSOLETES "${PROJECT_NAME}-devel")
  set(CPACK_RPM_PACKAGE_LICENSE "MIT" )
  # RPM package specific variable for ASAN
  set(CPACK_RPM_ASAN_PACKAGE_NAME "${PROJECT_NAME}-asan" )
  set(CPACK_RPM_ASAN_PACKAGE_REQUIRES "rocm-core-asan, ${rocJPEG_RPM_PACKAGE_LIST}" )
  set(CPACK_RPM_ASAN_PACKAGE_OBSOLETES "${PROJECT_NAME}-asan")
  # RPM package specific variable for Test
  set(CPACK_RPM_TEST_PACKAGE_NAME "${PROJECT_NAME}-test" )
  set(CPACK_RPM_TEST_PACKAGE_REQUIRES "rocm-core, ${CPACK_RPM_DEV_PACKAGE_NAME}" )
  set(CPACK_RPM_TEST_PACKAGE_OBSOLETES "${PROJECT_NAME}-test")

  if(NOT ROCM_DEP_ROCMCORE)
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_RPM_RUNTIME_PACKAGE_REQUIRES ${CPACK_RPM_RUNTIME_PACKAGE_REQUIRES})
    string(REGEX REPLACE ",? ?rocm-core-asan," "" CPACK_RPM_ASAN_PACKAGE_REQUIRES ${CPACK_RPM_ASAN_PACKAGE_REQUIRES})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_RPM_DEV_PACKAGE_REQUIRES ${CPACK_RPM_DEV_PACKAGE_REQUIRES})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_RPM_TEST_PACKAGE_REQUIRES ${CPACK_RPM_TEST_PACKAGE_REQUIRES})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_DEBIAN_RUNTIME_PACKAGE_DEPENDS ${CPACK_DEBIAN_RUNTIME_PACKAGE_DEPENDS})
    string(REGEX REPLACE ",? ?rocm-core-asan," "" CPACK_DEBIAN_ASAN_PACKAGE_DEPENDS ${CPACK_DEBIAN_ASAN_PACKAGE_DEPENDS})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_DEBIAN_DEV_PACKAGE_DEPENDS ${CPACK_DEBIAN_DEV_PACKAGE_DEPENDS})
    string(REGEX REPLACE ",? ?rocm-core," "" CPACK_DEBIAN_TEST_PACKAGE_DEPENDS ${CPACK_DEBIAN_TEST_PACKAGE_DEPENDS})
  endif()

  if(ENABLE_ASAN_PACKAGING)
    # ASAN Package requires asan component with only libraries and license file
    set(CPACK_COMPONENTS_ALL asan)
  else()
    set(CPACK_COMPONENTS_ALL runtime dev test)
  endif()

  set(CPACK_DEBIAN_FILE_NAME "DEB-DEFAULT")
  set(CPACK_RPM_FILE_NAME "RPM-DEFAULT")
  set(CPACK_DEBIAN_PACKAGE_RELEASE "local")
  set(CPACK_RPM_PACKAGE_RELEASE "local")

  if(DEFINED ENV{CPACK_DEBIAN_PACKAGE_RELEASE})
    set(CPACK_DEBIAN_PACKAGE_RELEASE $ENV{CPACK_DEBIAN_PACKAGE_RELEASE})
  endif()
  if(DEFINED ENV{CPACK_RPM_PACKAGE_RELEASE})
    set(CPACK_RPM_PACKAGE_RELEASE $ENV{CPACK_RPM_PACKAGE_RELEASE})
  endif()

  set(CPACK_GENERATOR "TGZ;ZIP")
  if(EXISTS ${MAKE_NSIS_EXE})
    list(APPEND CPACK_GENERATOR "NSIS")
  endif()
  if(EXISTS ${RPMBUILD_EXE})
    list(APPEND CPACK_GENERATOR "RPM")
    message("-- ${White}AMD ROCm rocJPEG RunTime Package -- ${CPACK_RPM_RUNTIME_PACKAGE_REQUIRES}${ColourReset}")
    message("-- ${White}AMD ROCm rocJPEG Dev Package -- ${CPACK_RPM_DEV_PACKAGE_REQUIRES}${ColourReset}")
  endif()
  if(EXISTS ${DPKG_EXE})
    list(APPEND CPACK_GENERATOR "DEB")
    message("-- ${White}AMD ROCm rocJPEG RunTime Package -- ${CPACK_DEBIAN_RUNTIME_PACKAGE_DEPENDS}${ColourReset}")
    message("-- ${White}AMD ROCm rocJPEG Dev Package -- ${CPACK_DEBIAN_DEV_PACKAGE_DEPENDS}${ColourReset}")
  endif()

  include(CPack)

  cpack_add_component(runtime
                    DISPLAY_NAME "rocJPEG Runtime Package"
                    DESCRIPTION "AMD rocJPEG is a high performance JPEG decode SDK for AMD GPUs. \
rocJPEG runtime package provides rocJPEG library and license.txt")

  cpack_add_component(dev
                    DISPLAY_NAME "rocJPEG Develop Package"
                    DESCRIPTION "AMD rocJPEG is a high performance JPEG decode SDK for AMD GPUs. \
rocJPEG develop package provides rocJPEG library, header files, samples, and license.txt")

  cpack_add_component(asan
                    DISPLAY_NAME "rocJPEG ASAN Package"
                    DESCRIPTION "AMD rocJPEG is a high performance JPEG decode SDK for AMD GPUs. \
rocJPEG ASAN package provides rocJPEG ASAN libraries")

  cpack_add_component(test
                  DISPLAY_NAME "rocJPEG Test Package"
                  DESCRIPTION "AMD rocJPEG is a high performance JPEG decode SDK for AMD GPUs. \
rocJPEG Test package provides rocJPEG Test Components")

else()
  message("-- ${Red}AMD ROCm rocJPEG -- unmet dependencies${ColourReset}")
  if(NOT HIP_FOUND)
    message(FATAL_ERROR "-- ERROR!: HIP Not Found! - please install rocm-hip-runtime-dev!")
  endif()
  if(NOT Libva_FOUND)
    message(FATAL_ERROR "-- ERROR!: libva Not Found - please install libva-amdgpu-dev(DEBIAN)/libva-amdgpu-devel(RPM) package!")
  endif()
endif()