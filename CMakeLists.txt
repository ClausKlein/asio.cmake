#
# MIT License
#
# Copyright (c) 2021 Olivier Le Doeuff
#
# Permission is hereby granted, free of charge, to any person obtaining a copy of this software and
# associated documentation files (the "Software"), to deal in the Software without restriction,
# including without limitation the rights to use, copy, modify, merge, publish, distribute,
# sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all copies or
# substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT
# NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
# NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
# DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT
# OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
#

cmake_minimum_required(VERSION 3.14...3.31)

project(
  asio.cmake
  VERSION 1.2.1
  LANGUAGES CXX
)

# Define PROJECT_IS_TOP_LEVEL if not available
if(CMAKE_VERSION VERSION_LESS "3.21")
  string(COMPARE EQUAL "${CMAKE_CURRENT_SOURCE_DIR}" "${CMAKE_SOURCE_DIR}" PROJECT_IS_TOP_LEVEL)
endif()

set(ASIO_REPOSITORY
    "https://github.com/chriskohlhoff/asio"
    CACHE STRING "asio git repository url"
)
set(ASIO_TAG
    "asio-1-32-0"
    CACHE STRING "asio git tag"
)

option(ASIO_NO_DEPRECATED "Disables Asio deprecated interfaces and functionality" ON)
option(ASIO_ENABLE_EXAMPLES "add one example to test binding/compilation" ${PROJECT_IS_TOP_LEVEL})
option(ASIO_ENABLE_INSTALL "Install the asio standalone library" ${PROJECT_IS_TOP_LEVEL})

# Declare ##

include(GNUInstallDirs) # for CMAKE_INSTALL_INCLUDEDIR
include(cmake/get_cpm.cmake)

if(NOT TARGET asio::asio)

  # Download ##

  CPMAddPackage(
    NAME asio
    GIT_REPOSITORY ${ASIO_REPOSITORY}
    GIT_TAG ${ASIO_TAG}
    DOWNLOAD_ONLY TRUE
  )

  # Find Version ##
  file(STRINGS "${asio_SOURCE_DIR}/asio/include/asio/version.hpp" ASIO_VERSION
       REGEX "^#define ASIO_VERSION [0-9]+"
  )
  string(REGEX REPLACE "^#define ASIO_VERSION [0-9]+ // ([0-9.]+)$" "\\1" ASIO_VERSION
                       "${ASIO_VERSION}"
  )

  set(ASIO_VERSION
      ${ASIO_VERSION}
      CACHE STRING "asio library version" FORCE
  )

  # Configure ##

  set(CMAKE_DEBUG_POSTFIX D)
  add_library(asio STATIC src/asio.cpp)
  add_library(asio::asio ALIAS asio)
  target_include_directories(
    asio PUBLIC $<BUILD_INTERFACE:${asio_SOURCE_DIR}/asio/include>
                $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  )

  find_package(Threads REQUIRED)
  target_link_libraries(asio PUBLIC Threads::Threads)

  target_compile_features(asio PUBLIC cxx_std_11)
  target_compile_definitions(
    asio PUBLIC -DASIO_STANDALONE -DASIO_SEPARATE_COMPILATION
                $<$<BOOL:${ASIO_NO_DEPRECATED}>:ASIO_NO_DEPRECATED>
  )

  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(asio PRIVATE -Wall -Wextra -Wpedantic -Werror -Wshadow)
  endif()

  # Fix warning : "Please define _WIN32_WINNT or _WIN32_WINDOWS appropriately."
  # https://stackoverflow.com/questions/9742003/platform-detection-in-cmake
  if(WIN32 AND CMAKE_SYSTEM_VERSION)
    set(_ver ${CMAKE_SYSTEM_VERSION})
    string(REPLACE "." "" _ver ${_ver})
    string(REGEX REPLACE "([0-9])" "0\\1" _ver ${_ver})
    set(_version "0x${_ver}")
    target_compile_definitions(asio PUBLIC -D_WIN32_WINNT=${_version})
  endif()

endif()

#
# Examples
#
if(ASIO_ENABLE_EXAMPLES)
  enable_testing()

  add_executable(
    asio_example_cpp11_timers_time_t_timer
    ${asio_SOURCE_DIR}/asio/src/examples/cpp11/timers/time_t_timer.cpp
  )
  target_link_libraries(asio_example_cpp11_timers_time_t_timer asio::asio)
  # NOTE: why used from CI workflow? CK
  add_custom_target(asio_examples DEPENDS asio_example_cpp11_timers_time_t_timer)
  add_test(NAME asio_example COMMAND asio_example_cpp11_timers_time_t_timer)
endif()

#
# Install
#
if(ASIO_ENABLE_INSTALL)
  CPMAddPackage("gh:TheLartians/PackageProject.cmake@1.12.0")

  # Print Installing but not Up-to-date messages.
  set(CMAKE_INSTALL_MESSAGE LAZY)
  packageProject(
    NAME asio
    VERSION ${ASIO_VERSION}
    NAMESPACE asio
    BINARY_DIR ${PROJECT_BINARY_DIR}
    INCLUDE_DIR ${asio_SOURCE_DIR}/asio/include
    INCLUDE_DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    INCLUDE_HEADER_PATTERN "*.hpp" # NOTE: without *.ipp files! CK
    DISABLE_VERSION_SUFFIX YES
    COMPATIBILITY SameMajorVersion
    DEPENDENCIES "Threads"
  )
endif()
