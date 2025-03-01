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

# Set default build type if not specified
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type" FORCE)
endif()

# Define directory structure
set(BID_SRC_ROOT ${CMAKE_CURRENT_SOURCE_DIR})
set(SRC_DIR ${BID_SRC_ROOT}/src)
set(TSRC_DIR ${BID_SRC_ROOT}/tests)
set(F128_DIR ${BID_SRC_ROOT}/float128)
set(OBJ_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(GEN_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(TOBJ_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(EXE_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(LIB_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(RES_DIR ${CMAKE_CURRENT_BINARY_DIR})

# Platform detection
if(WIN32)
  set(IML_HOST_OS_TYPE "WINNT")
  set(IML_HOST_OS "WINNT")
else()
  if(APPLE)
    set(IML_HOST_OS "MACH")
  else()
    set(IML_HOST_OS "LINUX")
  endif()
  set(IML_HOST_OS_TYPE "LINUX")
endif()

# Architecture detection
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(IML_HOST_ARCH "EFI2")
  set(IML_HOST_ARCH_TYPE "EFI2")
else()
  set(IML_HOST_ARCH "IA32")
  set(IML_HOST_ARCH_TYPE "IA32")
endif()

# Compiler detection
if(CMAKE_C_COMPILER_ID MATCHES "Intel")
  set(IS_INTEL_CC TRUE)
  if(WIN32)
    set(CC_NAME "icl")
  else()
    set(CC_NAME "icc")
  endif()
else()
  set(IS_INTEL_CC FALSE)
  if(CMAKE_C_COMPILER_ID MATCHES "MSVC")
    set(CC_NAME "cl")
  elseif(CMAKE_C_COMPILER_ID MATCHES "Clang")
    set(CC_NAME "clang")
  else()
    set(CC_NAME "gcc")
  endif()
endif()

# Set up compiler flags
set(_CFLAGS_INC "-I${SRC_DIR}")
set(_CFLAGS_CONFIG "")
if(IML_HOST_OS_TYPE STREQUAL "LINUX")
  set(_CFLAGS_OS "LINUX")
else()
  set(_CFLAGS_OS "WINDOWS")
endif()

if(IML_HOST_ARCH_TYPE STREQUAL "IA32")
  set(_CFLAGS_ARCH "ia32")
elseif(IML_HOST_ARCH_TYPE STREQUAL "IA64")
  set(_CFLAGS_ARCH "ITANIUM" "ia64")
else()
  set(_CFLAGS_ARCH "efi2")
endif()

# BID build options
option(BID_BIG_ENDIAN "Use big endian" OFF)
option(DFP_WRAP "Use DFP wrappers" OFF)
option(CALL_BY_REF "Use call by reference" OFF)
option(GLOBAL_RND "Use global rounding" OFF)
option(GLOBAL_FLAGS "Use global flags" OFF)
option(UNCHANGED_BINARY_FLAGS "Use unchanged binary flags" OFF)
option(NO_BINARY80 "Disable binary80" OFF)

# Configure compiler flags based on options
if(BID_BIG_ENDIAN)
  list(APPEND _CFLAGS_CONFIG "BID_BIG_ENDIAN=1")
endif()

# Intel compiler specific flags
if(IS_INTEL_CC)
  if(CC_NAME STREQUAL "icl")
    set(_CFLAGS_CC "/Qlong-double /Qpc80 /Qstd=c99")
  endif()
  
  if(IML_HOST_OS_TYPE STREQUAL "WINNT")
    if(CC_NAME STREQUAL "icx")
      set(_CFLAGS_CC "/Qlong-double /Qpc80 /Qstd=c99")
    endif()
  endif()
  
  if(IML_HOST_OS_TYPE STREQUAL "LINUX")
    set(_CFLAGS_CC "-mlong-double-80 -pc80 -std=c99")
  endif()
  
  set(_USE_COMPILER_F128_TYPE TRUE)
  set(_USE_COMPILER_F80_TYPE TRUE)
else()
  set(_USE_COMPILER_F80_TYPE FALSE)
  set(_USE_COMPILER_F128_TYPE FALSE)
endif()

# Set USE_COMPILER_F128_TYPE and USE_COMPILER_F80_TYPE if not already set
if(NOT DEFINED USE_COMPILER_F128_TYPE)
  set(USE_COMPILER_F128_TYPE ${_USE_COMPILER_F128_TYPE})
endif()

if(NOT DEFINED USE_COMPILER_F80_TYPE)
  set(USE_COMPILER_F80_TYPE ${_USE_COMPILER_F80_TYPE})
endif()

# Configure compiler flags based on F128 and F80 options
if(NOT USE_COMPILER_F128_TYPE)
  list(APPEND _CFLAGS_CONFIG "USE_COMPILER_F128_TYPE=0")
else()
  list(APPEND _CFLAGS_CONFIG "USE_COMPILER_F128_TYPE=1")
  if(IS_INTEL_CC)
    set(_CFLAGS_CC "${_CFLAGS_CC} -Qoption,cpp,--extended_float_types")
  endif()
endif()

if(NOT USE_COMPILER_F80_TYPE)
  list(APPEND _CFLAGS_CONFIG "USE_COMPILER_F80_TYPE=0")
else()
  list(APPEND _CFLAGS_CONFIG "USE_COMPILER_F80_TYPE=1")
endif()

# Set BID build definitions
# set(BID_BLD_DEFS
#   DECIMAL_CALL_BY_REFERENCE=${CALL_BY_REF}
#   DECIMAL_GLOBAL_ROUNDING=${GLOBAL_RND}
#   DECIMAL_GLOBAL_EXCEPTION_FLAGS=${GLOBAL_FLAGS})

if(CALL_BY_REF)
  list(APPEND BID_BLD_DEFS DECIMAL_CALL_BY_REFERENCE=1)
else()
  list(APPEND BID_BLD_DEFS DECIMAL_CALL_BY_REFERENCE=0)
endif()

if(GLOBAL_RND)
  list(APPEND BID_BLD_DEFS DECIMAL_GLOBAL_ROUNDING=1)
else()
  list(APPEND BID_BLD_DEFS DECIMAL_GLOBAL_ROUNDING=0)
endif()

if(GLOBAL_FLAGS)
  list(APPEND BID_BLD_DEFS DECIMAL_GLOBAL_EXCEPTION_FLAGS=1)
else()
  list(APPEND BID_BLD_DEFS DECIMAL_GLOBAL_EXCEPTION_FLAGS=0)
endif()



if(UNCHANGED_BINARY_FLAGS)
  list(APPEND BID_BLD_DEFS UNCHANGED_BINARY_STATUS_FLAGS)
endif()

if(IS_INTEL_CC AND DFP_WRAP)
  list(APPEND BID_BLD_DEFS __DFP_WRAPPERS_ON=1)
endif()

if(NO_BINARY80)
  list(APPEND BID_BLD_DEFS __NO_BINARY80__)
endif()

# Define the library components
set(BID_COMMON_LIBM
    acos acosh asin asinh atan atan2 atanh cbrt cos cosh erf erfc exp
    exp10 exp2 expm1 hypot lgamma log log10 log1p log2 pow sin sinh tan
    tanh tgamma)

set(BID_COMMON_OPS
    add compare div fdimd fma fmod frexp ldexp llrintd logb logbd lrintd
    lround minmax modf mul nearbyintd next nexttowardd noncomp quantexpd
    quantize rem round_integral scalb scalbl sqrt string to_int16
    to_int32 to_int64 to_int8 to_uint16 to_uint32 to_uint64 to_uint8
    llround llquantexpd quantumd)

set(COMMON strtod wcstod)

set(BID_32 sub to_bid128 to_bid64)
set(BID_64 to_bid128)
set(BID_128 2_str_tables)

set(BID
    binarydecimal convert_data decimal_data decimal_globals dpd
    feclearexcept fegetexceptflag feraiseexcept fesetexceptflag
    fetestexcept flag_operations from_int round)

set(BID_MISC bid128)

# Define F128 components if needed
if(NOT USE_COMPILER_F128_TYPE)
  set(F128_NAMES
      dpml_ux_bid dpml_ux_bessel dpml_ux_cbrt dpml_ux_erf dpml_ux_exp 
      dpml_ux_int dpml_ux_inv_hyper dpml_ux_inv_trig dpml_ux_lgamma 
      dpml_ux_log dpml_ux_mod dpml_ux_powi dpml_ux_pow dpml_ux_sqrt 
      dpml_ux_trig dpml_ux_ops dpml_ux_ops_64
      dpml_four_over_pi dpml_exception sqrt_tab_t)
endif()

# Define F53 components for Windows with MSVC
if(WIN32 AND CMAKE_C_COMPILER_ID MATCHES "MSVC")
  set(F53_NAMES
      dpml_asinh_t dpml_acosh_t dpml_cbrt_t dpml_erf_t dpml_erfc_t 
      dpml_expm1_t dpml_exp10_t dpml_exp2_t dpml_lgamma_t dpml_log1p_t 
      dpml_tgamma_t dpml_rt_lgamma_t
      dpml_pow_t_table dpml_cbrt_t_table dpml_special_exp_t)
  
  if(IML_HOST_ARCH STREQUAL "IA32")
    list(APPEND F53_NAMES dpml_log2_t)
  endif()
endif()

# Function to generate source file lists
function(generate_source_files prefix suffix list result_var)
  set(result "")
  foreach(item ${list})
    list(APPEND result "${prefix}${item}${suffix}")
  endforeach()
  set(${result_var} ${result} PARENT_SCOPE)
endfunction()

# Generate source file lists for BID components
generate_source_files("${SRC_DIR}/bid64_" ".c" "${BID_COMMON_LIBM}" BID64_TRANS_SRCS)
generate_source_files("${SRC_DIR}/bid128_" ".c" "${BID_COMMON_LIBM}" BID128_TRANS_SRCS)
set(BID_TRANS_SRCS ${BID64_TRANS_SRCS} ${BID128_TRANS_SRCS})

# Generate internal source file lists
generate_source_files("${SRC_DIR}/bid32_" ".c" "${BID_COMMON_OPS}" BID32_OPS_SRCS)
generate_source_files("${SRC_DIR}/bid64_" ".c" "${BID_COMMON_OPS}" BID64_OPS_SRCS)
generate_source_files("${SRC_DIR}/bid128_" ".c" "${BID_COMMON_OPS}" BID128_OPS_SRCS)

generate_source_files("${SRC_DIR}/" "32.c" "${COMMON}" COMMON32_SRCS)
generate_source_files("${SRC_DIR}/" "64.c" "${COMMON}" COMMON64_SRCS)
generate_source_files("${SRC_DIR}/" "128.c" "${COMMON}" COMMON128_SRCS)

generate_source_files("${SRC_DIR}/bid32_" ".c" "${BID_COMMON_LIBM}" BID32_LIBM_SRCS)
generate_source_files("${SRC_DIR}/bid32_" ".c" "${BID_32}" BID32_SRCS)
generate_source_files("${SRC_DIR}/bid64_" ".c" "${BID_64}" BID64_SRCS)
generate_source_files("${SRC_DIR}/bid128_" ".c" "${BID_128}" BID128_SRCS)
generate_source_files("${SRC_DIR}/bid_" ".c" "${BID}" BID_SRCS)
generate_source_files("${SRC_DIR}/" ".c" "${BID_MISC}" BID_MISC_SRCS)

# Combine all BID source files
set(BID_INTERNAL_SRCS
    ${BID32_OPS_SRCS} ${BID64_OPS_SRCS} ${BID128_OPS_SRCS}
    ${COMMON32_SRCS} ${COMMON64_SRCS} ${COMMON128_SRCS}
    ${BID32_LIBM_SRCS} ${BID32_SRCS} ${BID64_SRCS} ${BID128_SRCS}
    ${BID_SRCS} ${BID_MISC_SRCS})

set(ALL_BID_SRCS ${BID_TRANS_SRCS} ${BID_INTERNAL_SRCS})

# Add F128 and F53 sources if needed
if(NOT USE_COMPILER_F128_TYPE)
  generate_source_files("${F128_DIR}/" ".c" "${F128_NAMES}" F128_SRCS)
  list(APPEND ALL_BID_SRCS ${F128_SRCS})
  
  if(WIN32 AND CMAKE_C_COMPILER_ID MATCHES "MSVC")
    generate_source_files("${F128_DIR}/" ".c" "${F53_NAMES}" F53_SRCS)
    list(APPEND ALL_BID_SRCS ${F53_SRCS})
  endif()
endif()

# Create the library target
add_library(bid STATIC ${ALL_BID_SRCS})

# Create an alias target with the namespace
add_library(IntelRDFPMathLib::bid ALIAS bid)

# Set include directories
target_include_directories(bid PUBLIC
  $<BUILD_INTERFACE:${SRC_DIR}>
  $<INSTALL_INTERFACE:include>
)

# Set compile definitions
target_compile_definitions(bid PRIVATE
  ${_CFLAGS_OS}
  ${_CFLAGS_ARCH}
  ${_CFLAGS_CONFIG}
  ${BID_BLD_DEFS})

# Set compile options
if(_CFLAGS_CC)
  target_compile_options(bid PRIVATE ${_CFLAGS_CC})
endif()

# Set dependencies between source files
set_source_files_properties(${SRC_DIR}/bid_b2d.c PROPERTIES
  OBJECT_DEPENDS "${SRC_DIR}/bid_b2d.h")
set_source_files_properties(${SRC_DIR}/strtod32.c PROPERTIES
  OBJECT_DEPENDS "${SRC_DIR}/bid_strtod.h")
set_source_files_properties(${SRC_DIR}/bid64_fma.c PROPERTIES
  OBJECT_DEPENDS "${SRC_DIR}/bid_inline_add.h")
set_source_files_properties(${SRC_DIR}/bid32_string.c PROPERTIES
  OBJECT_DEPENDS "${SRC_DIR}/bid128_2_str_macros.h;${SRC_DIR}/bid128_2_str.h")
set_source_files_properties(${SRC_DIR}/bid32_sqrt.c PROPERTIES
  OBJECT_DEPENDS "${SRC_DIR}/bid_sqrt_macros.h")
set_source_files_properties(${SRC_DIR}/bid32_div.c PROPERTIES
  OBJECT_DEPENDS "${SRC_DIR}/bid_div_macros.h")

# Set properties for BID_TRANS_SRCS
foreach(src ${BID_TRANS_SRCS})
  set_source_files_properties(${src} PROPERTIES
    COMPILE_FLAGS "${BID_BLD_FLAGS}"
    OBJECT_DEPENDS "${SRC_DIR}/bid_trans.h")
endforeach()

# Set properties for BID_INTERNAL_SRCS
foreach(src ${BID_INTERNAL_SRCS})
  set_source_files_properties(${src} PROPERTIES
    COMPILE_FLAGS "${BID_BLD_FLAGS}"
    OBJECT_DEPENDS "${SRC_DIR}/bid_internal.h")
endforeach()

# Set header dependencies
set_source_files_properties(${SRC_DIR}/bid_trans.h PROPERTIES
  OBJECT_DEPENDS "${SRC_DIR}/bid_internal.h")

set_source_files_properties(${SRC_DIR}/bid_internal.h PROPERTIES
  OBJECT_DEPENDS "${SRC_DIR}/bid_conf.h;${SRC_DIR}/bid_functions.h")

if(CMAKE_C_COMPILER_ID MATCHES "GNU")
  set_source_files_properties(${SRC_DIR}/bid_functions.h PROPERTIES
    OBJECT_DEPENDS "${SRC_DIR}/bid_gcc_intrinsics.h")
endif()

# Set F128 compile flags if needed
if(NOT USE_COMPILER_F128_TYPE)
  set(F128_CFLAGS "${_CFLAGS_OPT} -DUSE_NATIVE_QUAD_TYPE=0")
  
  if(IML_HOST_ARCH)
    string(TOLOWER "${IML_HOST_ARCH}" arch_lower)
    set(F128_CFLAGS "${F128_CFLAGS} -D${arch_lower}")
  endif()
  
  if(IML_HOST_OS)
    string(TOLOWER "${IML_HOST_OS}" os_lower)
    set(F128_CFLAGS "${F128_CFLAGS} -D${os_lower}")
  endif()
  
  if(CC_NAME)
    string(TOLOWER "${CC_NAME}" cc_lower)
    set(F128_CFLAGS "${F128_CFLAGS} -D${cc_lower}")
  endif()
  
  # Set properties for F128 sources
  foreach(src ${F128_SRCS})
    set_source_files_properties(${src} PROPERTIES
      COMPILE_FLAGS "${F128_CFLAGS}"
      OBJECT_DEPENDS "${F128_DIR}/dpml_ux.h")
  endforeach()
  
  # Set header dependencies for F128
  set_source_files_properties(${F128_DIR}/dpml_ux.h PROPERTIES
    OBJECT_DEPENDS "${F128_DIR}/dpml_private.h;${F128_DIR}/dpml_ux_32_64.h;${F128_DIR}/dpml_cons_x.h")
  
  # Set F53 compile flags if needed
  if(WIN32 AND CMAKE_C_COMPILER_ID MATCHES "MSVC")
    set(F53_CFLAGS "${_CFLAGS_OPT}")
    string(REPLACE "-DWINDOWS" "-DWNT" F53_CFLAGS "${F53_CFLAGS}")
    
    # Set properties for F53 sources
    # Note: The original makefile has complex rules for F53 sources
    # This is a simplified version
    foreach(src ${F53_SRCS})
      set_source_files_properties(${src} PROPERTIES
        COMPILE_FLAGS "${F53_CFLAGS}"
        OBJECT_DEPENDS "${F128_DIR}/dpml_private.h;${F128_DIR}/dpml_globals.h;${F128_DIR}/dpml_error_codes_enum.h")
    endforeach()
  endif()
endif()

# Installation rules
include(GNUInstallDirs)
install(TARGETS bid
  EXPORT IntelRDFPMathLibTargets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

install(FILES
  ${SRC_DIR}/bid_conf.h
  ${SRC_DIR}/bid_functions.h
  ${SRC_DIR}/bid_internal.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(EXPORT IntelRDFPMathLibTargets
  FILE IntelRDFPMathLibTargets.cmake
  NAMESPACE IntelRDFPMathLib::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/IntelRDFPMathLib)

# Add custom targets
add_custom_target(cleanLib
  COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/bid.dir/*.o
  COMMENT "Cleaning BID library object files")

add_custom_target(realCleanLib
  COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_CURRENT_BINARY_DIR}/libbid.a
  DEPENDS cleanLib
  COMMENT "Cleaning BID library and object files")

add_custom_target(cleanBinary
  COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles/bid.dir/float128/*.o
  COMMENT "Cleaning binary transcendental support object files")

add_custom_target(clean_all
  DEPENDS realCleanLib cleanBinary
  COMMENT "Cleaning all files")

# Print help message
add_custom_target(help_build
  COMMAND ${CMAKE_COMMAND} -E echo "This CMakeLists.txt has the following targets:"
  COMMAND ${CMAKE_COMMAND} -E echo "    all           The default target. Builds the bid library"
  COMMAND ${CMAKE_COMMAND} -E echo "    help          Prints this message"
  COMMAND ${CMAKE_COMMAND} -E echo "    cleanLib      Deletes all BID library object files"
  COMMAND ${CMAKE_COMMAND} -E echo "    realCleanLib  Deletes all BID library object files and the library"
  COMMAND ${CMAKE_COMMAND} -E echo "    cleanBinary   Deletes all binary transcendental support object files"
  COMMAND ${CMAKE_COMMAND} -E echo "    clean_all     Implies realCleanLib, cleanBinary"
  COMMAND ${CMAKE_COMMAND} -E echo ""
  COMMAND ${CMAKE_COMMAND} -E echo "Useful CMake options:"
  COMMAND ${CMAKE_COMMAND} -E echo "    -DUSE_COMPILER_F128_TYPE=ON/OFF  Use compiler intrinsic 128-bit floating point type"
  COMMAND ${CMAKE_COMMAND} -E echo "    -DUSE_COMPILER_F80_TYPE=ON/OFF   Use compiler intrinsic 80-bit floating point type"
  COMMAND ${CMAKE_COMMAND} -E echo "    -DBID_BIG_ENDIAN=ON/OFF          Use big endian"
  COMMAND ${CMAKE_COMMAND} -E echo "    -DDFP_WRAP=ON/OFF                Use DFP wrappers"
  COMMAND ${CMAKE_COMMAND} -E echo "    -DCALL_BY_REF=ON/OFF             Use call by reference"
  COMMAND ${CMAKE_COMMAND} -E echo "    -DGLOBAL_RND=ON/OFF              Use global rounding"
  COMMAND ${CMAKE_COMMAND} -E echo "    -DGLOBAL_FLAGS=ON/OFF            Use global flags"
  COMMAND ${CMAKE_COMMAND} -E echo "    -DUNCHANGED_BINARY_FLAGS=ON/OFF  Use unchanged binary flags"
  COMMAND ${CMAKE_COMMAND} -E echo "    -DNO_BINARY80=ON/OFF             Disable binary80"
  COMMENT "Displaying help information")
