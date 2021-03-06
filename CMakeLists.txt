#/*============================================================================
#
#  BARD: Basic Augmented Reality Demo.
#
#  Copyright (c) University College London (UCL). All rights reserved.
#
#  This software is distributed WITHOUT ANY WARRANTY; without even
#  the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR
#  PURPOSE.
#
#  See LICENSE.txt in the top level directory for details.
#
#============================================================================*/

######################################################################
# Set the minimum CMake version.
######################################################################
cmake_minimum_required(VERSION 3.1)


##################################################################################
# Set some CMake Policies.
# See http://cmake.org/cmake/help/cmake-2-8-docs.html#section_Policies for details
##################################################################################

set(project_policies
#  CMP0001 # NEW: CMAKE_BACKWARDS_COMPATIBILITY should no longer be used.
#  CMP0002 # NEW: Logical target names must be globally unique.
#  CMP0003 # NEW: Libraries linked via full path no longer produce linker search paths.
#  CMP0004 # NEW: Libraries linked may NOT have leading or trailing whitespace.
#  CMP0005 # NEW: Preprocessor definition values are now escaped automatically.
#  CMP0006 # NEW: Installing MACOSX_BUNDLE targets requires a BUNDLE DESTINATION.
#  CMP0007 # NEW: List command no longer ignores empty elements.
#  CMP0008 # NEW: Libraries linked by full-path must have a valid library file name.
#  CMP0009 # NEW: FILE GLOB_RECURSE calls should not follow symlinks by default.
#  CMP0010 # NEW: Bad variable reference syntax is an error.
#  CMP0011 # NEW: Included scripts do automatic cmake_policy PUSH and POP.
#  CMP0012 # NEW: if() recognizes numbers and boolean constants.
#  CMP0013 # NEW: Duplicate binary directories are not allowed.
#  CMP0014 # NEW: Input directories must have CMakeLists.txt
#  CMP0048 # NEW: Project specifies the version number.
#  CMP0053 # NEW: New references and variable expansion.
  )
foreach(policy ${project_policies})
  if(POLICY ${policy})
    cmake_policy(SET ${policy} NEW)
  endif()
endforeach()


######################################################################
# We have a super-build option. 
######################################################################

option(BUILD_SUPERBUILD "Build BARD and the projects it depends on via SuperBuild.cmake." ON)

if (BUILD_SUPERBUILD)
  project(BARD-superbuild)
  set(BARD_SOURCE_DIR ${PROJECT_SOURCE_DIR})
  set(BARD_BINARY_DIR ${PROJECT_BINARY_DIR})
else()
  project(BARD VERSION 15.05.0)
endif()

######################################################################
# Setting supported build types. Should ONLY be Release or Debug.
######################################################################

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Valid options are Release or Debug" FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Release" "Debug")
endif()

if (NOT (CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "Debug"))
  message(FATAL_ERROR "Build type \"${CMAKE_BUILD_TYPE}\" is not supported.")
endif()

if(WIN32)
  # Restrict the generated configuration to be what we configured above.
  # No point creating project files for build types that will not compile.
  # Note: it's set to FORCE so that both CMAKE_BUILD_TYPE and CMAKE_CONFIGURATION_TYPES match up.
  set(CMAKE_CONFIGURATION_TYPES ${CMAKE_BUILD_TYPE} CACHE STRING "Build configurations to generate." FORCE)
  mark_as_advanced(CMAKE_CONFIGURATION_TYPES)
endif()

######################################################################
# Options. These are set up front, so are available when configuring
# the SuperBuild, and hence they must also be passed to the normal
# build. So, look in CMake/Superbuild.cmake to see where they are
# passed to the main build of BARD when doing the Superbuild.
######################################################################
option(BUILD_SHARED_LIBS "Build BARD with shared libraries." ON)
set(BARD_EP_TARBALL_LOCATION "http://cmic.cs.ucl.ac.uk/platform/dependencies")

if (NOT BUILD_SHARED_LIBS)
  message(FATAL_ERROR "This project must be built with BUILD_SHARED_LIBS=ON due to bug in ArUco.")
endif()


######################################################################
# Setup the path to load CMake macros, and extra CMake files.
######################################################################
set(CMAKE_MODULE_PATH
    ${CMAKE_BINARY_DIR}
    ${CMAKE_SOURCE_DIR}/CMake
    ${CMAKE_SOURCE_DIR}/CMake/CMakeExternals
    ${CMAKE_MODULE_PATH}
   )


######################################################################
# Make sure Git is available.
######################################################################
find_package(Git REQUIRED)
if (WIN32)
  set(GITCOMMAND ${GIT_EXECUTABLE})
endif()


######################################################################
# This is for setting up runtime VS environment.
######################################################################
set(ADDITIONAL_SEARCH_PATHS)

######################################################################
# Find Qt. We need to check for Qt first, before we go ahead and do
# the Superbuild, because VTK needs to know if we have Qt.
######################################################################
set(DESIRED_QT_VERSION 5 CACHE STRING "Pick a version of Qt to use: 4 or 5")

if(DESIRED_QT_VERSION MATCHES 4)
  set(QT4_COMPONENTS QtCore QtGui)
  find_package(Qt4 COMPONENTS ${QT4_COMPONENTS} REQUIRED)
  if(QT_FOUND)
    set(QT_USE_QTTEST 1)
    set(QT_USE_QTXML 1)
    set(QT_USE_QTXMLPATTERNS 1)
    set(QT_USE_PHONON 0)
    include(${QT_USE_FILE})
  endif()
endif()
if(DESIRED_QT_VERSION MATCHES 5)
  set(CMAKE_PREFIX_PATH "${CMAKE_PREFIX_PATH}" CACHE PATH "")
  set(QT5_COMPONENTS Core Gui Widgets WebKitWidgets)
  find_package(Qt5 COMPONENTS ${QT5_COMPONENTS} REQUIRED)
  if(Qt5_DIR)
    get_filename_component(_Qt5_DIR "${Qt5_DIR}/../../../" ABSOLUTE)
    list(FIND CMAKE_PREFIX_PATH "${_Qt5_DIR}" _result)
    if(_result LESS 0)
      set(CMAKE_PREFIX_PATH "${_Qt5_DIR};${CMAKE_PREFIX_PATH}" CACHE PATH "" FORCE)
    endif()
    set(BARD_PREFIX_PATH ${_Qt5_DIR})
  endif()
endif()

# Determine the Qt4/5 library installation prefix
set(_qmake_location )
if(DESIRED_QT_VERSION MATCHES 4)
  set(_qmake_location ${QT_QMAKE_EXECUTABLE})
elseif(DESIRED_QT_VERSION MATCHES 5 AND TARGET ${Qt5Core_QMAKE_EXECUTABLE})
  get_property(_qmake_location TARGET ${Qt5Core_QMAKE_EXECUTABLE}
               PROPERTY IMPORT_LOCATION)
endif()
if(_qmake_location)
  if(NOT _qt_install_libs)
    if(WIN32)
      execute_process(COMMAND ${_qmake_location} -query QT_INSTALL_BINS
                      OUTPUT_VARIABLE _qt_install_libs
                      OUTPUT_STRIP_TRAILING_WHITESPACE)
    else()
      execute_process(COMMAND ${_qmake_location} -query QT_INSTALL_LIBS
                      OUTPUT_VARIABLE _qt_install_libs
                      OUTPUT_STRIP_TRAILING_WHITESPACE)
    endif()
    file(TO_CMAKE_PATH "${_qt_install_libs}" _qt_install_libs)
    set(_qt_install_libs ${_qt_install_libs} CACHE INTERNAL "Qt library installation prefix" FORCE)
  endif()
  if(_qt_install_libs)
    list(APPEND ADDITIONAL_SEARCH_PATHS ${_qt_install_libs})
  endif()
elseif(MITK_USE_QT)
  message(WARNING "The qmake executable could not be found.")
endif()

if(BUILD_SUPERBUILD)
  set(VTK_QT_ARGS
    -DVTK_QT_VERSION:STRING=${DESIRED_QT_VERSION}
    -DVTK_USE_GUISUPPORT:BOOL=ON
    -DVTK_USE_QVTK_QTOPENGL:BOOL=OFF
    -DVTK_USE_QT:BOOL=ON
    -DQT_QMAKE_EXECUTABLE:FILEPATH=${QT_QMAKE_EXECUTABLE}
  )
endif()

 
######################################################################
# Now, if required, do the SuperBuild
# If we are doing SuperBuild
#   We configure up to this point (see the return() statement)
#   and then we call SuperBuild.cmake, which builds all the
#   dependencies as CMake ExternalProjects, and then also builds
#   BARD as an ExternalProject. However instead of downloading
#   a tar file, you set the SOURCE_DIR to be THIS project, and force
#   the BUILD_SUPERBUILD flag to be off (to avoid infinite loop).
#
# If we are NOT doing superbuild, then the next statement has no
# effect, and the build goes past this point.
######################################################################

if(BUILD_SUPERBUILD)
  include("CMake/SuperBuild.cmake")
  return()
endif(BUILD_SUPERBUILD)


######################################################################
# End of SuperBuild. Print out where the source and binary folders
# are, just to make it really explicit... well, explicit to the user
# that bothers to read these messages! :-)
######################################################################

message("CMAKE_SOURCE_DIR=${CMAKE_SOURCE_DIR}")
message("CMAKE_BINARY_DIR=${CMAKE_BINARY_DIR}")


######################################################################
# Add additional entries to the CMAKE_MODULE_PATH, now all externals
# are already built.
######################################################################
set(CMAKE_MODULE_PATH
    ${aruco_DIR}/lib/cmake
    ${CMAKE_MODULE_PATH}
   )


######################################################################
# Force MSVC runtime. Depends on BUILD_SHARED_LIBS
######################################################################
if(MSVC)
  set(variables
    CMAKE_C_FLAGS_DEBUG
    CMAKE_C_FLAGS_MINSIZEREL
    CMAKE_C_FLAGS_RELEASE
    CMAKE_C_FLAGS_RELWITHDEBINFO
    CMAKE_CXX_FLAGS_DEBUG
    CMAKE_CXX_FLAGS_MINSIZEREL
    CMAKE_CXX_FLAGS_RELEASE
    CMAKE_CXX_FLAGS_RELWITHDEBINFO
  )
  if(BUILD_SHARED_LIBS)
    message(STATUS "Forcing MSVC to use dynamic runtime variables.")
    foreach(variable ${variables})
      if(${variable} MATCHES "/MT")
        string(REGEX REPLACE "/MT" "/MD" ${variable} "${${variable}}")
      endif()
    endforeach()
  else()
    message(STATUS "Forcing MSVC to use static runtime variables.")
    foreach(variable ${variables})
      if(${variable} MATCHES "/MD")
        string(REGEX REPLACE "/MD" "/MT" ${variable} "${${variable}}")
      endif()
    endforeach()
  endif()
  message(STATUS "Initial build flags:")
  foreach(variable ${variables})
    message(STATUS "  '${variable}': ${${variable}}")
  endforeach()
  message(STATUS "")
endif()


######################################################################
# Find Mandatory External packages
######################################################################

find_package(VTK REQUIRED)
include(${VTK_USE_FILE})
list(APPEND ADDITIONAL_SEARCH_PATHS "${VTK_INSTALL_PREFIX}/lib")

set(OpenCV_FIND_QUIETLY ON)
find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIRS})
add_definitions(-DBUILD_OpenCV)
if(WIN32)
  list(APPEND ADDITIONAL_SEARCH_PATHS ${OpenCV_LIB_PATH})
else()
  list(APPEND ADDITIONAL_SEARCH_PATHS "${OpenCV_INSTALL_PATH}/lib")
endif()

find_package(aruco REQUIRED)
list(APPEND ADDITIONAL_SEARCH_PATHS ${aruco_LIB_DIR})

set(ALL_LIBS)
list(APPEND ALL_LIBS ${aruco_LIBS} ${OpenCV_LIBS} ${VTK_LIBRARIES})

set(ADDITIONAL_BIN_DIRS)
#if(WIN32)
  foreach (_dir ${ADDITIONAL_SEARCH_PATHS})
    get_filename_component(_up_dir ${_dir} PATH)
    if(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} VERSION_LESS 2.8)
      get_filename_component(_abs_dir ${_up_dir} ABSOLUTE)
    else()
      get_filename_component(_abs_dir ${_up_dir} REALPATH)
    endif()
    list(APPEND ADDITIONAL_BIN_DIRS "${_abs_dir}/bin")
  endforeach()
##endif()


######################################################################
# Output directories, for when compiling, not installing.
######################################################################

configure_file(${CMAKE_SOURCE_DIR}/Doc/Doxygen/bardgroup.pl ${CMAKE_BINARY_DIR}/bardgroup.pl @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/Doc/Doxygen/vxl_doxy.pl ${CMAKE_BINARY_DIR}/vxl_doxy.pl @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/Doc/Doxygen/barddoxygen.pl.in ${CMAKE_BINARY_DIR}/barddoxygen.pl)
configure_file(${CMAKE_SOURCE_DIR}/Doc/Doxygen/doxygen.config.in ${CMAKE_BINARY_DIR}/doxygen.config)

set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)
set(CXX_TEST_PATH ${EXECUTABLE_OUTPUT_PATH})              # Used to control, where to put unit test binaries.
set(BASELINE ${BARD_DATA_DIR}/Baseline)
set(INPUT_DATA ${BARD_DATA_DIR}/Input)
make_directory(${CMAKE_BINARY_DIR}/Testing/Temporary)
set(TEMP ${CMAKE_BINARY_DIR}/Testing/Temporary)

foreach(type LIBRARY RUNTIME ARCHIVE)
  set(output_dir ${CMAKE_BINARY_DIR}/bin)
  set(CMAKE_${type}_OUTPUT_DIRECTORY ${output_dir} CACHE INTERNAL "Single output directory for building all libraries.")
  mark_as_advanced(CMAKE_${type}_OUTPUT_DIRECTORY)
endforeach()


#######################################################################
# Set the main install locations.
# These are relative to CMAKE_INSTALL_PREFIX which we MUST NOT touch.
#######################################################################
set(BARD_INSTALL_BASE_DIR ".")
set(BARD_INSTALL_BIN_DIR "bin")
set(BARD_INSTALL_INCLUDE_DIR "include/BARD")
set(BARD_INSTALL_MATLAB_DIR "matlab")
set(BARD_INSTALL_DOC_DIR "doc")
set(BARD_INSTALL_LICENSES_DIR "licenses")
set(BARD_INSTALL_LIB_DIR "bin")

#######################################################################
# Set the main include path, just before generating BARDConfigure.
#######################################################################
include_directories(
  ${CMAKE_SOURCE_DIR}/Code
  )

######################################################################
# Decide what subdirectories we are building, and go and build them.
######################################################################

add_subdirectory(Code)

######################################################################
# If we are under Windows, create batch files which correctly
# set up the environment for Visual Studio. These are only used to
# start VS when developing. Not used in the final installation package.
######################################################################
if(WIN32)
  set(VS_SOLUTION_FILE "${PROJECT_BINARY_DIR}/${PROJECT_NAME}.sln")
  foreach(VS_BUILD_TYPE ${CMAKE_CONFIGURATION_TYPES})
    configure_file("${CMAKE_SOURCE_DIR}/CMake/StartVS.bat.in" ${PROJECT_BINARY_DIR}/StartVS_${VS_BUILD_TYPE}.bat @ONLY)
    message( "CreateWindowsBatchScript: Creating ${PROJECT_BINARY_DIR}/StartVS_${VS_BUILD_TYPE}.bat" )
  endforeach()
endif(WIN32)


