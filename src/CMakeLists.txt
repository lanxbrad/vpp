# Copyright (c) 2018 Cisco and/or its affiliates.
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at:
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 3.10)

set(CMAKE_C_COMPILER_NAMES
  clang-12
  clang-11
  clang-10
  clang-9
  gcc-10
  gcc-9
  cc
)

project(vpp C)

set(CMAKE_EXPORT_COMPILE_COMMANDS on)

include(CheckCCompilerFlag)
include(CheckIPOSupported)
include(cmake/misc.cmake)
include(cmake/cpu.cmake)
include(cmake/ccache.cmake)

##############################################################################
# VPP Version
##############################################################################
execute_process(
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  COMMAND scripts/version
  OUTPUT_VARIABLE VPP_VERSION
  OUTPUT_STRIP_TRAILING_WHITESPACE
)
string(REPLACE "-" ";" VPP_LIB_VERSION ${VPP_VERSION})
list(GET VPP_LIB_VERSION 0 VPP_LIB_VERSION)

##############################################################################
# cross compiling
##############################################################################
if(CMAKE_CROSSCOMPILING)
  set(CMAKE_IGNORE_PATH
    /usr/lib/${CMAKE_HOST_SYSTEM_PROCESSOR}-linux-gnu/
    /usr/lib/${CMAKE_HOST_SYSTEM_PROCESSOR}-linux-gnu/lib/
  )
endif()
set(CMAKE_C_COMPILER_TARGET ${CMAKE_SYSTEM_PROCESSOR}-linux-gnu)

##############################################################################
# build config
##############################################################################
check_c_compiler_flag("-Wno-address-of-packed-member"
		      compiler_flag_no_address_of_packed_member)
set(VPP_RUNTIME_DIR "bin" CACHE STRING "Relative runtime directory path")
set(VPP_LIBRARY_DIR "lib" CACHE STRING "Relative library directory path")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${VPP_RUNTIME_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/${VPP_LIBRARY_DIR})
set(VPP_BINARY_DIR ${CMAKE_BINARY_DIR}/CMakeFiles)

if (CMAKE_BUILD_TYPE)
  set(CMAKE_C_FLAGS "-g -fPIC -Werror -Wall ${CMAKE_C_FLAGS}")
endif()

if (compiler_flag_no_address_of_packed_member)
  set(CMAKE_C_FLAGS "-Wno-address-of-packed-member ${CMAKE_C_FLAGS}")
endif()

# release
list(APPEND BUILD_TYPES "release")
string(CONCAT CMAKE_C_FLAGS_RELEASE
  "-O3 "
  "-fstack-protector "
  "-D_FORTIFY_SOURCE=2 "
  "-fno-common "
)

string(CONCAT CMAKE_EXE_LINKER_FLAGS_RELEASE "-pie")

# debug
list(APPEND BUILD_TYPES "debug")
string(CONCAT CMAKE_C_FLAGS_DEBUG
  "-O0 "
  "-DCLIB_DEBUG "
  "-fstack-protector "
  "-fno-common "
)

# coverity
list(APPEND BUILD_TYPES "coverity")
string(CONCAT CMAKE_C_FLAGS_COVERITY "-O2 -D__COVERITY__")

# gcov
list(APPEND BUILD_TYPES "gcov")
string(CONCAT CMAKE_C_FLAGS_GCOV
  "-O0 "
  "-DCLIB_DEBUG "
  "-DCLIB_GCOV "
  "-fprofile-arcs "
  "-ftest-coverage ")

string(TOUPPER "${CMAKE_BUILD_TYPE}" CMAKE_BUILD_TYPE_UC)


string(REPLACE ";" " " BUILD_TYPES_STR "${BUILD_TYPES}")
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY
	     HELPSTRING "Build type - valid options are: ${BUILD_TYPES_STR}")

##############################################################################
# link time optimizations
##############################################################################
if (CMAKE_BUILD_TYPE_UC STREQUAL "RELEASE")
  check_ipo_supported(RESULT _result)
  if (_result)
    option(VPP_USE_LTO "Link time optimization of release binaries" ON)
  endif()
endif()

##############################################################################
# sanitizers
##############################################################################

option(VPP_ENABLE_SANITIZE_ADDR "Enable Address Sanitizer" OFF)
set(VPP_SANITIZE_ADDR_OPTIONS
  "unmap_shadow_on_exit=1:disable_coredump=0:abort_on_error=1:detect_leaks=0"
  CACHE
  STRING "Address sanitizer arguments"
)

if (VPP_ENABLE_SANITIZE_ADDR)
  set(CMAKE_C_FLAGS "-fsanitize=address -DCLIB_SANITIZE_ADDR ${CMAKE_C_FLAGS}")
  set(CMAKE_EXE_LINKER_FLAGS "-fsanitize=address ${CMAKE_EXE_LINKER_FLAGS}")
  set(CMAKE_SHARED_LINKER_FLAGS "-fsanitize=address ${CMAKE_SHARED_LINKER_FLAGS}")
endif (VPP_ENABLE_SANITIZE_ADDR)

##############################################################################
# trajectory trace
##############################################################################

option(VPP_ENABLE_TRAJECTORY_TRACE "Build vpp with trajectory tracing enabled" OFF)
if(VPP_ENABLE_TRAJECTORY_TRACE)
  set(CMAKE_C_FLAGS "-DVLIB_BUFFER_TRACE_TRAJECTORY=1 ${CMAKE_C_FLAGS}")
endif()

##############################################################################
# install config
##############################################################################
option(VPP_SET_RPATH "Set rpath for resulting binaries and libraries." ON)
if(VPP_SET_RPATH)
  set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
endif()
set(CMAKE_INSTALL_MESSAGE NEVER)

include_directories (
	${CMAKE_SOURCE_DIR}
	${VPP_BINARY_DIR}
)
set(CMAKE_INSTALL_DEFAULT_COMPONENT_NAME "vpp")

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

include(cmake/syscall.cmake)
include(cmake/api.cmake)
include(cmake/library.cmake)
include(cmake/exec.cmake)
include(cmake/plugin.cmake)

##############################################################################
# subdirs - order matters
##############################################################################
option(VPP_HOST_TOOLS_ONLY "Build only host tools" OFF)
if(VPP_HOST_TOOLS_ONLY)
  set(SUBDIRS tools/vppapigen cmake)
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  find_package(OpenSSL REQUIRED)
  set(SUBDIRS
    vppinfra svm vlib vlibmemory vlibapi vnet vpp vat vat2 vcl plugins
    vpp-api tools/vppapigen tools/g2 tools/perftool cmake pkg
    tools/appimage
  )
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
  set(SUBDIRS vppinfra)
else()
  message(FATAL_ERROR "Unsupported system: ${CMAKE_SYSTEM_NAME}")
endif()

foreach(DIR ${SUBDIRS})
  add_subdirectory(${DIR} ${VPP_BINARY_DIR}/${DIR})
endforeach()

##############################################################################
# detect if we are inside git repo and add configure dependency
##############################################################################
execute_process(
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  COMMAND git rev-parse --show-toplevel
  OUTPUT_VARIABLE VPP_GIT_TOPLEVEL_DIR
  OUTPUT_STRIP_TRAILING_WHITESPACE
  ERROR_QUIET
)

if (VPP_GIT_TOPLEVEL_DIR)
  set_property(
    DIRECTORY APPEND PROPERTY
    CMAKE_CONFIGURE_DEPENDS ${VPP_GIT_TOPLEVEL_DIR}/.git/index
  )
endif()

##############################################################################
# custom targets
##############################################################################

add_custom_target(run
  COMMAND ./${VPP_RUNTIME_DIR}/vpp -c startup.conf
  COMMENT "Starting VPP..."
  USES_TERMINAL
)

add_custom_target(debug
  COMMAND gdb --args ./${VPP_RUNTIME_DIR}/vpp -c startup.conf
  COMMENT "Starting VPP in the debugger..."
  USES_TERMINAL
)

add_custom_target(config
  COMMAND ccmake ${CMAKE_BINARY_DIR}
  COMMENT "Starting Configuration TUI..."
  USES_TERMINAL
)

add_custom_target(compdb
  COMMAND ninja -C ${CMAKE_BINARY_DIR} -t compdb |
          ${CMAKE_SOURCE_DIR}/scripts/compdb_cleanup.py >
	  ${CMAKE_BINARY_DIR}/compile_commands.json
  COMMENT "Generating compile_commands.json"
  USES_TERMINAL
)

foreach(bt ${BUILD_TYPES})
  add_custom_target(set-build-type-${bt}
    COMMAND cmake -DCMAKE_BUILD_TYPE:STRING=${bt} .
    COMMENT "Changing build type to ${bt}"
    USES_TERMINAL
  )
endforeach()

##############################################################################
# print configuration
##############################################################################
message(STATUS "Configuration:")
pr("VPP version" "${VPP_VERSION}")
pr("VPP library version" "${VPP_LIB_VERSION}")
pr("GIT toplevel dir" "${VPP_GIT_TOPLEVEL_DIR}")
pr("Build type" "${CMAKE_BUILD_TYPE}")
pr("C flags" "${CMAKE_C_FLAGS}${CMAKE_C_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
pr("Linker flags (apps)" "${CMAKE_EXE_LINKER_FLAGS}${CMAKE_EXE_LINKER_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
pr("Linker flags (libs)" "${CMAKE_SHARED_LINKER_FLAGS}${CMAKE_SHARED_LINKER_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
pr("Host processor" "${CMAKE_HOST_SYSTEM_PROCESSOR}")
pr("Target processor" "${CMAKE_SYSTEM_PROCESSOR}")
pr("Prefix path" "${CMAKE_PREFIX_PATH}")
pr("Install prefix" "${CMAKE_INSTALL_PREFIX}")
