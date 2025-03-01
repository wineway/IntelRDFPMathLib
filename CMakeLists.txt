##############################################################################
# ==============================================================================
#  Copyright (c) 2007-2024, Intel Corp.
#  All rights reserved.
#
#  Redistribution and use in source and binary forms, with or without 
#  modification, are permitted provided that the following conditions are met:
#
#    * Redistributions of source code must retain the above copyright notice, 
#      this list of conditions and the following disclaimer.
#    * Redistributions in binary form must reproduce the above copyright 
#      notice, this list of conditions and the following disclaimer in the 
#      documentation and/or other materials provided with the distribution.
#    * Neither the name of Intel Corporation nor the names of its contributors 
#      may be used to endorse or promote products derived from this software 
#      without specific prior written permission.
#
#  THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
#  AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
#  IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
#  ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
#  LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
#  CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
#  SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
#  INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
#  CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
#  ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF
#  THE POSSIBILITY OF SUCH DAMAGE.
# ==============================================================================
##############################################################################

# CMake minimum version
cmake_minimum_required(VERSION 3.10)

# Project name and version
project(IntelRDFPMathLib VERSION 20.0.3 LANGUAGES C)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

# Add option to control whether to build as a subproject
# Determine if this is the master project by checking if current source dir equals source dir
if(${CMAKE_CURRENT_SOURCE_DIR} STREQUAL ${CMAKE_SOURCE_DIR})
  set(IS_MASTER_PROJECT ON)
else()
  set(IS_MASTER_PROJECT OFF)
endif()
option(INTELRDFPMATHLIB_MASTER_PROJECT "Build as master project" ${IS_MASTER_PROJECT})

add_subdirectory("LIBRARY")

# Create config file
configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/IntelRDFPMathLibConfig.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/IntelRDFPMathLib
)

# Install config file
install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/IntelRDFPMathLibConfig.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/IntelRDFPMathLib
)

# Export targets for build tree usage (important for FetchContent)
export(EXPORT IntelRDFPMathLibTargets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/IntelRDFPMathLibTargets.cmake"
  NAMESPACE IntelRDFPMathLib::
)

# Make the targets available in the build directory
if(NOT INTELRDFPMATHLIB_MASTER_PROJECT)
  # If we're included via add_subdirectory or FetchContent, make the targets available to the parent project
  export(PACKAGE IntelRDFPMathLib)
endif()