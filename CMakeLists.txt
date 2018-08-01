cmake_minimum_required(VERSION 3.9.0 FATAL_ERROR)
project(LAPACK-VisualStudio-mingw-gfortran 
  VERSION 1.0
  DESCRIPTION "A wrapper around LAPACK to allow compilation under Visual Studio  with assistance from MinGW gfortran."
  LANGUAGES C)

# Build LAPACK on Windows using CMake, MinGW's gfortran and Visual Studio.

# This is a master project that will 
# 1. download and extract lapack sources (${LAPACK_VERSION})
# 2. Use cmake to generate a Visual Studio Solution that leverages CMake's CMakeAddFortranSubdirectory features.

# Requirements:
# 1. Visual Studio
# 2. CMake
# 3. MSYS with MinGW with gfortran.exe

# Reference:
# https://blog.kitware.com/fortran-for-cc-developers-made-easier-with-cmake/

if( ${CMAKE_GENERATOR} MATCHES "Unix Makefiles" AND
    (DEFINED ENV{FC} OR DEFINED CMAKE_Fortran_COMPILER ))
  enable_language( Fortran OPTIONAL )
endif()

# Build system configuration files are located here.
set( CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/config )

# We expect the developer to provide lapack sources or we will download them...
set( LAPACK_VERSION "3.8.0" )
set( tarfile "lapack-${LAPACK_VERSION}.tar.gz" ) 
set( url "https://www.netlib.org/lapack/${tarfile}" )
# alternate https://github.com/Reference-LAPACK/lapack/archive/v${LAPACK_VERSION}.zip

if( NOT EXISTS ${PROJECT_BINARY_DIR}/${tarfile} )
  message( STATUS "Downloading lapack source files to ${PROJECT_BINARY_DIR}/${tarfile}" )
  file(DOWNLOAD ${url} ${PROJECT_BINARY_DIR}/${tarfile} SHOW_PROGRESS )
endif()

if( EXISTS ${PROJECT_BINARY_DIR}/${tarfile} AND NOT EXISTS ${PROJECT_SOURCE_DIR}/lapack-${LAPACK_VERSION}/README.md )
  message( "Extracting downloaded lapack sources to ${PROJECT_SOURCE_DIR}/lapack-${LAPACK_VERSION}.")
  execute_process( 
    COMMAND cmake -E tar xvf ${PROJECT_BINARY_DIR}/${tarfile} 
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} 
    OUTPUT_QUIET
    # COMMENT "Unpacking lapack sources to ${PROJECT_SOURCE_DIR}/lapack-${LAPACK_VERSION}" 
    )
endif()    

# We repeat the build options found in the actual LAPACK sources so that the most 
# common options can be set from this super-build project.

# Set a default build type if none was specified
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo" "Coverage")
endif()

if(NOT CMAKE_CONFIGURATION_TYPES)
  string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_UPPER)
  if(${CMAKE_BUILD_TYPE_UPPER} STREQUAL "COVERAGE")
    message(STATUS "Adding coverage")
    find_package(codecov)
  endif()
endif()

# By default static library
option(BUILD_SHARED_LIBS "Build shared libraries" ON) # Required for MSVC + gfortran.

# deprecated LAPACK and LAPACKE routines
option(BUILD_DEPRECATED "Build deprecated routines" OFF)

# --------------------------------------------------
# Precision to build
# By default all precisions are generated
option(BUILD_SINGLE "Build single precision real" ON)
option(BUILD_DOUBLE "Build double precision real" ON)
option(BUILD_COMPLEX "Build single precision complex" ON)
option(BUILD_COMPLEX16 "Build double precision complex" ON)

if(NOT (BUILD_SINGLE OR BUILD_DOUBLE OR BUILD_COMPLEX OR BUILD_COMPLEX16))
  message(FATAL_ERROR "Nothing to build, no precision selected.
  Please enable at least one of these:
  BUILD_SINGLE, BUILD_DOUBLE, BUILD_COMPLEX, BUILD_COMPLEX16.")
endif()

# --------------------------------------------------
# Subdirectories that need to be processed
option(USE_OPTIMIZED_BLAS "Whether or not to use an optimized BLAS library instead of included netlib BLAS" OFF)

# --------------------------------------------------
# CBLAS
option(CBLAS "Build CBLAS" OFF)

# --------------------------------------------------
# XBLAS

option(USE_XBLAS "Build extended precision (needs XBLAS)" OFF)
option(USE_OPTIMIZED_LAPACK "Whether or not to use an optimized LAPACK library instead of included netlib LAPACK" OFF)

# --------------------------------------------------
# LAPACKE
option(LAPACKE "Build LAPACKE" OFF)

# LAPACKE has also the interface to some routines from tmglib,
# if LAPACKE_WITH_TMG is selected, we need to add those routines to LAPACKE
option(LAPACKE_WITH_TMG "Build LAPACKE with tmglib routines" OFF)

# --------------------------------------------------
# Build netlib-lapack via cmake_add_fortran_subdirectory.
# --------------------------------------------------
include(CMakeAddFortranSubdirectory)

# Pass build parameters down wto the Makefile based build.
set( build_system_state
  "-DBUILD_SHARED_LIBS=${BUILD_SHARED_LIBS}"
  "-DBUILD_DEPRECATED=${BUILD_DEPRECATED}"
  "-DBUILD_SINGLE=${BUILD_SINGLE}"
  "-DBUILD_DOUBLE=${BUILD_DOUBLE}"
  "-DBUILD_COMPLEX=${BUILD_COMPLEX}"
  "-DBUILD_COMPLEX16=${BUILD_COMPLEX16}"
  "-DUSE_OPTIMIZED_BLAS=${USE_OPTIMIZED_BLAS}"
  "-DCBLAS=${CBLAS}"
  "-DUSE_XBLAS=${USE_XBLAS}"
  "-DUSE_OPTIMIZED_LAPACK=${USE_OPTIMIZED_LAPACK}"
  "-DLAPACKE=${LAPACKE}"
  "-DLAPACKE_WITH_TMG=${LAPACKE_WITH_TMG}"
  )  

# add the lapack subdirectory as a fortran project the subdir is 
# netlib-lapack, the project is netlib-lapack
cmake_add_fortran_subdirectory(
  lapack-${LAPACK_VERSION}      # directory name
  PROJECT netlib-lapack         # project name in toplevel CMakeLists.txt in lapack
                                # VS target will be 'netlib-lapack_build'
  ARCHIVE_DIR ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/lib # .lib location relative to root binary tree
  RUNTIME_DIR ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/bin # .dll location relative to root binary tree
  LIBRARIES blas lapack         # these target libraries created
                                # --> libblas.dll and liblapack.dll
  # TARGET_NAMES foo bar
  # DEPENDS LapackFetchSources
  NO_EXTERNAL_INSTALL
  CMAKE_COMMAND_LINE "${build_system_state}"
  # VERBOSE
  )

#include (print_target_properties)  
#print_targets_properties( "lapack;blas" )

add_executable(testc src/test.c)
target_link_libraries(testc lapack blas)

# --------------------------------------------------
# Install commands
install(
  FILES 
    ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/bin/libblas.dll
    ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/bin/liblapack.dll
    ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/lib/libblas.dll.a
    ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/lib/libblas.exp
    ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/lib/libblas.lib
    ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/lib/liblapack.dll.a
    ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/lib/liblapack.exp
    ${PROJECT_BINARY_DIR}/lapack-${LAPACK_VERSION}/lib/liblapack.lib
  DESTINATION ${CMAKE_INSTALL_PREFIX}/lib/$<CONFIG> )

# --------------------------------------------------