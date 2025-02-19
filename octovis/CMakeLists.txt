cmake_minimum_required(VERSION 3.0.2)
project(octovis)

include(CTest)
include(GNUInstallDirs)

set(OpenGL_GL_PREFERENCE GLVND)

# # version (e.g. for packaging)
set(OCTOVIS_MAJOR_VERSION 1)
set(OCTOVIS_MINOR_VERSION 9)
set(OCTOVIS_PATCH_VERSION 7)
set(OCTOVIS_VERSION
    ${OCTOVIS_MAJOR_VERSION}.${OCTOVIS_MINOR_VERSION}.${OCTOVIS_PATCH_VERSION})
set(OCTOVIS_SOVERSION ${OCTOVIS_MAJOR_VERSION}.${OCTOVIS_MINOR_VERSION})
# get rid of a useless warning:
if(COMMAND cmake_policy)
  cmake_policy(SET CMP0003 NEW)
  if(POLICY CMP0042)
    # Enable MACOSX_RPATH by default.
    cmake_policy(SET CMP0042 NEW)
  endif(POLICY CMP0042)
endif(COMMAND cmake_policy)

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/CMakeModules")

# COMPILER SETTINGS (default: Release) and flags
include(CompilerSettings)

# Set output directories for libraries and executables
set(BASE_DIR ${CMAKE_BINARY_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${BASE_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${BASE_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${BASE_DIR}/bin)
# output dirs for multi-config builds (MSVC)
foreach(OUTPUTCONFIG ${CMAKE_CONFIGURATION_TYPES})
  string(TOUPPER ${OUTPUTCONFIG} OUTPUTCONFIG)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${BASE_DIR}/lib)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${BASE_DIR}/lib)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${BASE_DIR}/bin)
endforeach(OUTPUTCONFIG CMAKE_CONFIGURATION_TYPES)

# We need the main octomap library to link against (same version as octovis)
# Look at parent directory by default, in case the complete distribution
# (octomap & octovis together) is built.
#
# Otherwise you need to export octomap_DIR to the directory containing the file
# octomap-config.cmake
find_package(
  octomap ${OCTOVIS_MAJOR_VERSION}.${OCTOVIS_MINOR_VERSION} REQUIRED HINTS
  ${CMAKE_BINARY_DIR}/lib/cmake/octomap
  ${CMAKE_BINARY_DIR}/../octomap/lib/cmake/octomap)
message(STATUS "Found octomap version: " ${octomap_VERSION})

include_directories(BEFORE SYSTEM ${OCTOMAP_INCLUDE_DIRS})

# Export the package for use from the build-tree (this registers the build-tree
# with a global CMake-registry)
export(PACKAGE octovis)

set(INSTALL_TARGETS_DEFAULT_ARGS
    RUNTIME
    DESTINATION
    "${CMAKE_INSTALL_BINDIR}"
    LIBRARY
    DESTINATION
    "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE
    DESTINATION
    "${CMAKE_INSTALL_LIBDIR}")

# Builds the "octovis" viewer based on OpenGL and libQGLViewer, if dependencies
# available
set(BUILD_VIEWER 0)

option(OCTOVIS_QT5 "Link Octovis against Qt5?" ON)

# Look for required libraries:
find_package(OpenGL)
if(NOT OCTOVIS_QT5)
  find_package(Qt4)
endif(NOT OCTOVIS_QT5)

if(OpenGL-NOTFOUND OR Qt4-NOTFOUND)
  message("OpenGL and QT4 are required for octovis but could not be found.")
else()
  find_package(QGLViewer)
  if(QGLViewer_FOUND)
    set(BUILD_VIEWER 1)
  else()
    message("\n")
    message("libQGLViewer could not be found or generated.")
  endif()
endif()

if(BUILD_VIEWER)
  message(STATUS "\n")
  message(STATUS "viewer octovis will be built")

  set(INCLUDE_DIRS "${PROJECT_SOURCE_DIR}/include")
  include_directories(${INCLUDE_DIRS})

  include(CMakeLists_src.txt)

  # Create an octovis-config.cmake file for the use from the build tree
  set(OCTOVIS_INCLUDE_DIRS "${INCLUDE_DIRS}")
  set(OCTOVIS_LIB_DIR "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")
  # Set library names as absolute paths Windows, spec. MSVC requires the .lib
  # suffix for imported libs
  if(WIN32)
    set(OCTOVIS_LIBRARY
        "${CMAKE_IMPORT_LIBRARY_PREFIX}octovis${CMAKE_IMPORT_LIBRARY_SUFFIX}")
  else()
    set(OCTOVIS_LIBRARY
        "${CMAKE_SHARED_LIBRARY_PREFIX}octovis${CMAKE_SHARED_LIBRARY_SUFFIX}")
  endif()

  include(CMakePackageConfigHelpers)

  configure_package_config_file(
    octovis-config.cmake.in
    "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/cmake/octovis/octovis-config.cmake"
    PATH_VARS OCTOVIS_INCLUDE_DIRS OCTOVIS_LIB_DIR
    INSTALL_DESTINATION ${CMAKE_INSTALL_PREFIX}/share/octovis)

  write_basic_package_version_file(
    "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/cmake/octovis/octovis-config-version.cmake"
    VERSION ${OCTOVIS_VERSION}
    COMPATIBILITY AnyNewerVersion)

  # not used right now (export depends?) set(OCTOMAP_CMAKE_DIR
  # "${PROJECT_BINARY_DIR}")

  # Create a octovis-config.cmake file for the use from the install tree and
  # install it
  set(OCTOVIS_INCLUDE_DIRS "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
  set(OCTOVIS_LIB_DIR "${CMAKE_INSTALL_FULL_LIBDIR}")
  # set(OCTOMAP_CMAKE_DIR "${INSTALL_DATA_DIR}/FooBar/CMake")

  set(OCTOVIS_INCLUDE_TARGETS
      "include(\${CMAKE_CURRENT_LIST_DIR}/octovis-targets.cmake)")

  configure_package_config_file(
    octovis-config.cmake.in
    "${PROJECT_BINARY_DIR}/InstallFiles/octovis-config.cmake"
    PATH_VARS OCTOVIS_INCLUDE_DIRS OCTOVIS_LIB_DIR
    INSTALL_DESTINATION "${CMAKE_INSTALL_FULL_DATADIR}/octovis")

  write_basic_package_version_file(
    "${PROJECT_BINARY_DIR}/InstallFiles/octovis-config-version.cmake"
    VERSION ${OCTOVIS_VERSION}
    COMPATIBILITY AnyNewerVersion)

  install(
    FILES "${PROJECT_BINARY_DIR}/InstallFiles/octovis-config.cmake"
          "${PROJECT_BINARY_DIR}/InstallFiles/octovis-config-version.cmake"
    DESTINATION "${CMAKE_INSTALL_DATADIR}/octovis")

  # #installation: # store all header files to install:
  file(GLOB octovis_HDRS *.h *.hxx *.hpp)
  install(FILES ${octovis_HDRS}
          DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/octovis")

  # Install catkin package.xml
  install(FILES package.xml DESTINATION "${CMAKE_INSTALL_DATADIR}/octovis")

  # Allows Colcon to find non-Ament packages when using workspace underlays
  file(
    WRITE
    ${CMAKE_CURRENT_BINARY_DIR}/share/ament_index/resource_index/packages/${PROJECT_NAME}
    "")
  install(
    FILES
      ${CMAKE_CURRENT_BINARY_DIR}/share/ament_index/resource_index/packages/${PROJECT_NAME}
    DESTINATION share/ament_index/resource_index/packages)

else()
  message(
    "Unfortunately, the viewer (octovis) can not be built because some requirements are missing."
  )
  message(
    "This will not affect the compilation of the stand-alone library and tools (octomap)"
  )
  message("See README.txt or http://octomap.sf.net for further information.\n")
endif()
