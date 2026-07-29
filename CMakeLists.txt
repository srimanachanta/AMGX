# Copyright (c) 2011-2025, NVIDIA CORPORATION. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of NVIDIA CORPORATION nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
# OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required (VERSION 3.18)

# IMPORTANT: CMAKE_CUDA_ARCHITECTURES must be set before project() enables CUDA
# Backward compatibility: translate old CUDA_ARCH to new CMAKE_CUDA_ARCHITECTURES
if(DEFINED CUDA_ARCH)
    message(WARNING "CUDA_ARCH is deprecated. Please use CMAKE_CUDA_ARCHITECTURES instead.")
    set(CMAKE_CUDA_ARCHITECTURES "${CUDA_ARCH}" CACHE STRING "Target CUDA architectures" FORCE)
endif()

# If CMAKE_CUDA_ARCHITECTURES is not set, provide a default
if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
    # Default to common architectures (will be validated later based on CUDA version)
    set(CMAKE_CUDA_ARCHITECTURES "90;100;120" CACHE STRING 
        "CUDA architectures to compile for (e.g., 90;100;120)")
    message(STATUS "Using default CUDA architectures: ${CMAKE_CUDA_ARCHITECTURES}")
else()
    message(STATUS "Using CUDA architectures: ${CMAKE_CUDA_ARCHITECTURES}")
endif()

# the project
project (AMG LANGUAGES C CXX CUDA)

# Read the release version so the shared library can carry SONAME/version metadata
file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/ReleaseVersion.txt" AMGX_VERSION LIMIT_COUNT 1)
string(STRIP "${AMGX_VERSION}" AMGX_VERSION)
string(REGEX MATCH "^[0-9]+" AMGX_VERSION_MAJOR "${AMGX_VERSION}")

# Build-shaping options. Defaults preserve upstream behavior; a downstream consumer
# (e.g. a Python extension linking only the AMGX C API) turns tests/examples/static off.
option(AMGX_BUILD_TESTS           "Build AMGX unit-test targets"                 ON)
option(AMGX_BUILD_EXAMPLES        "Build AMGX example executables"               ON)
option(AMGX_BUILD_STATIC          "Build the static amgx library"                ON)
option(AMGX_BUILD_SHARED          "Build the shared amgxsh library"              ON)
option(AMGX_HIDE_INTERNAL_SYMBOLS "Export only the AMGX_* C API from the shared library" ON)
option(AMGX_STRIP_RELEASE         "Strip + gc-sections the shared library in Release" ON)
option(AMGX_INSTALL               "Generate AMGX install rules (headers/libs/configs)" ON)

# Size-shaping options. Defaults preserve upstream behavior (all modes/solvers).
# A downstream consumer that needs only one precision mode and a fixed solver set
# (e.g. a Python extension pinned to dDDI + PCG/AMG) can slash the device fatbin
# without dropping any CUDA architecture.
option(AMGX_ENABLE_COMPLEX "Compile complex-valued precision modes" ON)
# AMGX instantiates three real precision combos (double-double, double-float,
# float-float) for its AMG/cycle machinery via hardcoded explicit instantiations,
# independent of the FORALL mode list. When OFF, only the double-double (matDouble)
# variants are built. This MUST accompany a double-only mode restriction
# (AMGX_BUILD_DEVICE_MODE=AMGX_mode_dDDI / AMGX_BUILD_HOST_MODE=AMGX_mode_hDDI),
# otherwise the FORALL side would emit float modes with no matching hardcoded roots.
option(AMGX_ENABLE_MIXED_PRECISION "Compile the double-float and float-float AMG precision variants" ON)
option(AMGX_MINIMAL_SOLVERS "Compile only the solvers/eigensolvers in the minimal keep-set" OFF)
set(AMGX_BUILD_DEVICE_MODE "" CACHE STRING
    "Restrict the device build to a single mode, e.g. AMGX_mode_dDDI (empty = all)")
set(AMGX_BUILD_HOST_MODE "" CACHE STRING
    "Restrict the host build to a single mode, e.g. AMGX_mode_hDDI (empty = all)")

find_package(MPI)

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/../cmake" ${CMAKE_MODULE_PATH})

#disable in-place builds
if(${CMAKE_BINARY_DIR} STREQUAL ${CMAKE_CURRENT_SOURCE_DIR})
  MESSAGE(FATAL_ERROR "Error:  In-place builds are not supported. Please create a separate build directory")
endif(${CMAKE_BINARY_DIR} STREQUAL ${CMAKE_CURRENT_SOURCE_DIR})

# declare the supported configurations. Only for a standalone build: FORCEing this
# rewrites the configuration list a multi-config generator already set up for a parent
# project that added AMGX as a subdirectory.
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  set(CMAKE_CONFIGURATION_TYPES "Debug;Release;Profile;RelWithTraces" CACHE STRING "Available Configuration Types" FORCE)
endif()

# make sure a build type has been chosen!!!
IF(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are Debug Release Profile RelWithTraces" FORCE)
ENDIF(NOT CMAKE_BUILD_TYPE)

find_package(CUDAToolkit 12.0 REQUIRED)

# AMGX requires at least C++17 (CUDA >= 13.1 Thrust / CUDA C++ Core Libraries). The per-target
# minimum is requested via target_compile_features(amgx_libs ...) below so that, when AMGX is
# added as a subdirectory, a parent project that already selected a newer standard (e.g. C++20)
# is never downgraded. Only seed a default here for standalone builds, and never FORCE it.
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
  set(CMAKE_CXX_STANDARD_REQUIRED ON)
endif()
if(NOT CMAKE_CUDA_STANDARD)
  set(CMAKE_CUDA_STANDARD 17)
  set(CMAKE_CUDA_STANDARD_REQUIRED ON)
endif()

# update/define the compilation flags.
#
# AMGX historically FORCEd CMAKE_{C,CXX}_FLAGS* into the cache on Windows. Those are
# global variables, so when AMGX is add_subdirectory()'d the flags leak onto every
# other target in the project: /MT selects the static CRT, which a CPython extension
# module must not use; /Zl strips the default-library directives an MSVC object needs
# to link; and FORCEing the cache discards whatever the toolchain had already chosen.
# Everything that block genuinely needed is expressed per-target below, so leave the
# defaults alone. CMAKE_MSVC_RUNTIME_LIBRARY (CMP0091) picks /MD, Windows-MSVC.cmake
# already supplies /DWIN32 /D_WINDOWS /W3 /GR /EHsc, and CMAKE_*_FLAGS_<CONFIG>_INIT
# already supplies the optimization flags.
option(AMGX_OVERRIDE_MSVC_FLAGS
  "Legacy: FORCE the global MSVC flags (/MT, /Zl) into the cache" OFF)

IF(WIN32 AND AMGX_OVERRIDE_MSVC_FLAGS)
  set(CMAKE_C_FLAGS                "/DWIN32 /D_WINDOWS /W3 /bigobj" CACHE STRING "" FORCE)
  set(CMAKE_C_FLAGS_DEBUG          "/D_DEBUG /Zl /Zi /Ob0 /Od /RTC1" CACHE STRING "" FORCE)
  set(CMAKE_C_FLAGS_MINSIZEREL     "/D_DEBUG /Zl /Zi /Ob0 /Od /RTC1" CACHE STRING "" FORCE)
  set(CMAKE_C_FLAGS_RELEASE        "/MT /O2 /Ob2 /D NDEBUG" CACHE STRING "" FORCE)
  set(CMAKE_C_FLAGS_RELWITHDEBINFO "/MT /O2 /Ob2 /D NDEBUG" CACHE STRING "" FORCE)

  set(CMAKE_CXX_FLAGS                "/DWIN32 /D_WINDOWS /DNOMINMAX /W3 /GR /EHsc /bigobj" CACHE STRING "" FORCE)
  set(CMAKE_CXX_FLAGS_DEBUG          "/D_DEBUG /Zl /Zi /Ob0 /Od /RTC1" CACHE STRING "" FORCE)
  set(CMAKE_CXX_FLAGS_MINSIZEREL     "/D_DEBUG /Zl /Zi /Ob0 /Od /RTC1" CACHE STRING "" FORCE)
  set(CMAKE_CXX_FLAGS_RELEASE        "/MT /O2 /Ob2 /DNDEBUG" CACHE STRING "Force flags. /Zl is used to remove deps" FORCE)
  set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "/MT /O2 /Ob2 /DNDEBUG" CACHE STRING "Force flags. /Zl is used to remove deps" FORCE)
ENDIF()

# disabling specific warnings
if(CMAKE_COMPILER_IS_GNUCXX)
  if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 6.0)
    # we throw in ~Matrix(), we assume any error fatal so std::terminate call is ok
    set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -Wno-terminate)
  endif()
endif()

# C++20 deprecates arithmetic between two different enum types; AMGX and the Thrust/CUB it uses
# do this pervasively, so silence it to avoid flooding a C++20 (e.g. downstream) build. The flag
# is recognized and harmless under C++17. Forwarded to nvcc via -Xcompiler ${CMAKE_CXX_FLAGS}.
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -Wno-deprecated-enum-enum-conversion)
endif()

set(CMAKE_C_FLAGS_PROFILE "${CMAKE_C_FLAGS_RELEASE}" CACHE STRING "" FORCE)
set(CMAKE_CXX_FLAGS_PROFILE "${CMAKE_CXX_FLAGS_RELEASE}" CACHE STRING "" FORCE)
set(CMAKE_EXE_LINKER_FLAGS_PROFILE "${CMAKE_EXE_LINKER_FLAGS_RELEASE}" CACHE STRING "" FORCE)
set(CMAKE_SHARED_LINKER_FLAGS_PROFILE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE}" CACHE STRING "" FORCE)

set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE}" CACHE STRING "")
set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE}" CACHE STRING "")

set(CMAKE_C_FLAGS_RELWITHTRACES "${CMAKE_C_FLAGS_RELEASE}" CACHE STRING "" FORCE)
set(CMAKE_CXX_FLAGS_RELWITHTRACES "${CMAKE_CXX_FLAGS_RELEASE}" CACHE STRING "" FORCE)
set(CMAKE_EXE_LINKER_FLAGS_RELWITHTRACES "${CMAKE_EXE_LINKER_FLAGS_RELEASE}" CACHE STRING "" FORCE)
set(CMAKE_SHARED_LINKER_FLAGS_RELWITHTRACES "${CMAKE_SHARED_LINKER_FLAGS_RELEASE}" CACHE STRING "" FORCE)

# install paths
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX ".." CACHE PATH "Path where AMGX will be installed" FORCE)
endif()
# add CMAKE_INSTALL_PREFIX/lib to the RPATH to be used when installing,
# but only if it's not a system directory
list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES
     "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
if("${isSystemDir}" STREQUAL "-1")
  set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
endif()
# add the automatically determined parts of the RPATH
# which point to directories outside the build tree to the install RPATH
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# ignore rpath completely, if requested:
set(AMGX_NO_RPATH False CACHE BOOL "Don't build RPATH")
if (${AMGX_NO_RPATH})
  set(CMAKE_SKIP_RPATH TRUE)
endif (${AMGX_NO_RPATH})

if (DEFINED ENV{CRAY_MPICH2_DIR})
  set(MPI_C_LIBRARIES "$ENV{CRAY_MPICH2_DIR}/lib/libmpich.so")
  set(MPI_C_INCLUDE_PATH "$ENV{CRAY_MPICH2_DIR}/include")
endif(DEFINED ENV{CRAY_MPICH2_DIR})

set(AMGX_INCLUDE_EXTERNAL True CACHE BOOL "Include external 3rd party libraries")
if (AMGX_INCLUDE_EXTERNAL)
  set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -DRAPIDJSON_DEFINED)
  set(CMAKE_C_FLAGS ${CMAKE_C_FLAGS} -DRAPIDJSON_DEFINED)
  include_directories("${CMAKE_CURRENT_SOURCE_DIR}/external/rapidjson/include")
endif (AMGX_INCLUDE_EXTERNAL)

set(CMAKE_NO_MPI false CACHE BOOL "Force non-MPI build")

if (CMAKE_NO_MPI)
  message("Non-MPI build has been forced")
  set(MPI_FOUND false)
else (CMAKE_NO_MPI)
  if(MPI_FOUND)
    include_directories(${MPI_INCLUDE_PATH})
    set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -DAMGX_WITH_MPI)
    set(CMAKE_C_FLAGS ${CMAKE_C_FLAGS} -DAMGX_WITH_MPI)
    if(WIN32)
      set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -DOMPI_IMPORTS)
    endif(WIN32)
  endif(MPI_FOUND)
endif (CMAKE_NO_MPI)

message ("This is a MPI build:" ${MPI_FOUND})

# Enable NVTX ranges on Linux
if(NOT WIN32)
  set(NVTXRANGE_FLAG -DNVTX_RANGES)
endif()

# Configuration specific nvcc flags
GET_FILENAME_COMPONENT(CMAKE_CXX_COMPILER_NAME "${CMAKE_CXX_COMPILER}" NAME)
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_NAME MATCHES "clang")
  set(CUDA_NVCC_FLAGS_DEBUG -g -G)
  set(CUDA_NVCC_FLAGS_RELEASE -DNDEBUG)
  set(CUDA_NVCC_FLAGS_PROFILE -DPROFILE)
  set(CUDA_NVCC_FLAGS_RELWITHTRACES -g -DNDEBUG)
else()
  set(CUDA_NVCC_FLAGS_DEBUG -g -G)
  set(CUDA_NVCC_FLAGS_RELEASE -DNDEBUG)
  set(CUDA_NVCC_FLAGS_PROFILE -DPROFILE)
  set(CUDA_NVCC_FLAGS_RELWITHTRACES -g -DNDEBUG)
  if(WIN32)
    set(CUDA_NVCC_FLAGS_RELEASE ${CUDA_NVCC_FLAGS_RELEASE} -DNOMINMAX)
  endif()
endif()

# Add the build-specific flags to the NVCC flags
string(TOUPPER ${CMAKE_BUILD_TYPE} UPP_BUILD_NAME)
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} ${CUDA_NVCC_FLAGS_${UPP_BUILD_NAME}})

# Enable device lambdas
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} --extended-lambda)

# Add errors for execution space warnings and enable NVTX ranges
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} --Werror cross-execution-space-call ${NVTXRANGE_FLAG})

# Keep intermediate files
if (AMGX_keep_intermediate)
    set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} -keep)
endif(AMGX_keep_intermediate)

# Windows/linux specific settings for C
GET_FILENAME_COMPONENT(CMAKE_C_COMPILER_NAME "${CMAKE_C_COMPILER}" NAME)
IF(CMAKE_C_COMPILER_NAME MATCHES cl AND NOT CMAKE_C_COMPILER_NAME MATCHES clang)
    set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
ELSE(CMAKE_C_COMPILER_NAME MATCHES cl AND NOT CMAKE_C_COMPILER_NAME MATCHES clang)
    # Place each function/datum in its own section so the linker can --gc-sections away
    # anything unreferenced. -rdynamic is intentionally omitted: it force-exports every symbol
    # into the dynamic table, which defeats stripping and hidden visibility.
    set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -fPIC -ffunction-sections -fdata-sections)
    if(AMGX_HIDE_INTERNAL_SYMBOLS)
      # Only the AMGX_* C API is tagged visibility("default") in amgx_c.h, so hiding everything
      # else exports just the public C API from amgxsh and lets more internal code be pruned.
      set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -fvisibility=hidden -fvisibility-inlines-hidden)
    else()
      set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -fvisibility=default)
    endif()
ENDIF(CMAKE_C_COMPILER_NAME MATCHES cl AND NOT CMAKE_C_COMPILER_NAME MATCHES clang)

# VS: include object files in target property SOURCES
# otherwise a workaround for extracting ${obj_all} is necessary below
set(CUDA_ATTACH_VS_BUILD_RULE_TO_CUDA_FILE OFF)

IF(CMAKE_CUDA_COMPILER_VERSION VERSION_GREATER_EQUAL 13.0.0)

  SET(CUDA_ALLOW_ARCH "70;75;80;86;89;90;100;120")

ELSEIF(CMAKE_CUDA_COMPILER_VERSION VERSION_GREATER_EQUAL 12.0.0)

  SET(CUDA_ALLOW_ARCH "70;75;80;86;89;90;100;120")

ELSE()

  MESSAGE(FATAL_ERROR "Error: CUDA minimum supported version is 12.0.")

ENDIF()

# Validate CMAKE_CUDA_ARCHITECTURES against what this CUDA version supports
message(STATUS "CUDA ${CMAKE_CUDA_COMPILER_VERSION} supports architectures: ${CUDA_ALLOW_ARCH}")
message(STATUS "Building for architectures: ${CMAKE_CUDA_ARCHITECTURES}")

# Add the CXX flags to the host set of CUDA flags
string(REPLACE ";" " " CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
string(REPLACE ";" " " CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} -Xcompiler ${CMAKE_CXX_FLAGS})
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} -DTHRUST_CUB_WRAPPED_NAMESPACE=amgx)

add_library(amgx_libs OBJECT "")

# Request C++17 as a minimum without downgrading a parent project's newer standard (see above).
target_compile_features(amgx_libs PUBLIC cxx_std_17 cuda_std_17)

# What the removed global MSVC flag block was actually needed for, scoped to the target
# that needs it (see the comment at the top of this file).
if(MSVC)
  # NOMINMAX: vector.h pulls in <Windows.h>, whose min/max macros break std::min and
  # std::max and, through them, Thrust. Upstream only added -DNOMINMAX to the Release
  # nvcc flags; it has to hold in every configuration and in the device pass too, so it
  # is a definition rather than a flag.
  target_compile_definitions(amgx_libs PUBLIC NOMINMAX WIN32_LEAN_AND_MEAN)
  # /bigobj: several translation units exceed the 65,535-section COFF limit.
  # /utf-8:  a number of sources carry UTF-8 bytes; without this MSVC decodes them in
  #          the system ANSI code page.
  target_compile_options(amgx_libs PRIVATE
    $<$<COMPILE_LANGUAGE:CXX>:/bigobj;/utf-8>
    $<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=/bigobj;-Xcompiler=/utf-8>)
endif()

# Size-shaping definitions must be visible to every translation unit (config.h is
# included everywhere), so apply them PUBLIC on the object library that feeds both
# the static and shared targets. Keeping them consistent avoids ODR/link mismatch.
if(AMGX_BUILD_DEVICE_MODE)
  target_compile_definitions(amgx_libs PUBLIC AMGX_build_device=${AMGX_BUILD_DEVICE_MODE})
  message(STATUS "AMGX: restricting device build to ${AMGX_BUILD_DEVICE_MODE}")
endif()
if(AMGX_BUILD_HOST_MODE)
  target_compile_definitions(amgx_libs PUBLIC AMGX_build_host=${AMGX_BUILD_HOST_MODE})
  message(STATUS "AMGX: restricting host build to ${AMGX_BUILD_HOST_MODE}")
endif()
if(NOT AMGX_ENABLE_COMPLEX)
  target_compile_definitions(amgx_libs PUBLIC AMGX_NO_COMPLEX)
  message(STATUS "AMGX: complex precision modes disabled")
endif()
if(NOT AMGX_ENABLE_MIXED_PRECISION)
  target_compile_definitions(amgx_libs PUBLIC AMGX_NO_MIXED_PRECISION)
  message(STATUS "AMGX: mixed/single (float) precision variants disabled")
endif()
if(AMGX_MINIMAL_SOLVERS)
  target_compile_definitions(amgx_libs PUBLIC AMGX_MINIMAL_SOLVERS)
  message(STATUS "AMGX: compiling minimal solver keep-set only")
endif()

# The test targets (built in src/) link amgx and amgxsh, and the examples link both as well.
if(AMGX_BUILD_TESTS AND NOT (AMGX_BUILD_STATIC AND AMGX_BUILD_SHARED))
  message(FATAL_ERROR "AMGX_BUILD_TESTS requires both AMGX_BUILD_STATIC and AMGX_BUILD_SHARED")
endif()
if(AMGX_BUILD_EXAMPLES AND NOT (AMGX_BUILD_STATIC AND AMGX_BUILD_SHARED))
  message(FATAL_ERROR "AMGX_BUILD_EXAMPLES requires both AMGX_BUILD_STATIC and AMGX_BUILD_SHARED")
endif()

add_subdirectory("src")

# CMAKE_CUDA_ARCHITECTURES automatically applies to all CUDA targets
# No need to set per-target properties when set as cache variable above

target_compile_options(amgx_libs PUBLIC $<$<COMPILE_LANGUAGE:CUDA>: ${CUDA_NVCC_FLAGS} >)

if(AMGX_INSTALL)
  install(FILES
    include/amgx_config.h
    include/amgx_c.h
    DESTINATION include)
endif(AMGX_INSTALL)

# build static amgx
if(AMGX_BUILD_STATIC)
  add_library(amgx STATIC $<TARGET_OBJECTS:amgx_libs>)  # static lib
  add_library(AMGX::amgx ALIAS amgx)
  target_link_libraries(amgx amgx_libs)
  set_target_properties(amgx PROPERTIES LINKER_LANGUAGE CUDA)
  target_compile_options(amgx PUBLIC $<$<COMPILE_LANGUAGE:CUDA>: ${CUDA_NVCC_FLAGS} >)

  # CUDA::cudart explicitly: an embedding project may set CMAKE_CUDA_RUNTIME_LIBRARY
  # to None to control where the runtime comes from, and then nothing else supplies it.
  IF (WIN32)
    target_link_libraries(amgx CUDA::cudart CUDA::cublas CUDA::cusparse CUDA::cusolver)
  ELSE (WIN32)
    target_link_libraries(amgx CUDA::cudart CUDA::cublas CUDA::cusparse CUDA::cusolver m pthread)
  ENDIF(WIN32)

  if(MPI_FOUND)
    target_link_libraries(amgx MPI::MPI_CXX)
  endif(MPI_FOUND)

  if(AMGX_INSTALL)
    install(TARGETS amgx DESTINATION "lib")
  endif(AMGX_INSTALL)
endif(AMGX_BUILD_STATIC)

# build shared amgxsh
if(AMGX_BUILD_SHARED)
  add_library(amgxsh SHARED $<TARGET_OBJECTS:amgx_libs>)  # shared lib
  add_library(AMGX::amgxsh ALIAS amgxsh)
  target_link_libraries(amgxsh amgx_libs)
  set_target_properties(amgxsh PROPERTIES
    LINKER_LANGUAGE CUDA
    VERSION "${AMGX_VERSION}"
    SOVERSION "${AMGX_VERSION_MAJOR}")
  target_compile_options(amgxsh PUBLIC $<$<COMPILE_LANGUAGE:CUDA>: ${CUDA_NVCC_FLAGS} >)

  # CUDA::cudart explicitly: an embedding project may set CMAKE_CUDA_RUNTIME_LIBRARY
  # to None to control where the runtime comes from, and then nothing else supplies it.
  IF (WIN32)
    target_link_libraries(amgxsh CUDA::cudart CUDA::cublas CUDA::cusparse CUDA::cusolver)
  ELSE (WIN32)
    target_link_libraries(amgxsh CUDA::cudart CUDA::cublas CUDA::cusparse CUDA::cusolver m pthread)
    # Drop unreferenced sections and stop re-exporting symbols pulled in from the static
    # CUDA libraries (cublas/cusparse/...) so only the AMGX_* C API remains in the export table.
    target_link_options(amgxsh PRIVATE -Wl,--gc-sections -Wl,--exclude-libs,ALL)
    if(AMGX_STRIP_RELEASE)
      target_link_options(amgxsh PRIVATE $<$<CONFIG:Release>:-Wl,--strip-all>)
    endif()
  ENDIF(WIN32)

  if(MPI_FOUND)
    target_link_libraries(amgxsh MPI::MPI_CXX)
  endif(MPI_FOUND)

  if(AMGX_INSTALL)
    install(TARGETS amgxsh DESTINATION "lib")
  endif(AMGX_INSTALL)
endif(AMGX_BUILD_SHARED)

# CMAKE_CUDA_ARCHITECTURES is inherited by all targets automatically

# build examples
if(AMGX_BUILD_EXAMPLES)
  add_subdirectory(examples)
endif(AMGX_BUILD_EXAMPLES)

