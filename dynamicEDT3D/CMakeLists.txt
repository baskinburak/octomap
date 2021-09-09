cmake_minimum_required(VERSION 3.0.2)
project(dynamicEDT3D)

include(CTest)
include(GNUInstallDirs)

# version (e.g. for packaging)
set(DYNAMICEDT3D_MAJOR_VERSION 1)
set(DYNAMICEDT3D_MINOR_VERSION 9)
set(DYNAMICEDT3D_PATCH_VERSION 7)
set(DYNAMICEDT3D_VERSION
    ${DYNAMICEDT3D_MAJOR_VERSION}.${DYNAMICEDT3D_MINOR_VERSION}.${DYNAMICEDT3D_PATCH_VERSION}
)
set(DYNAMICEDT3D_SOVERSION
    ${DYNAMICEDT3D_MAJOR_VERSION}.${DYNAMICEDT3D_MINOR_VERSION})

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

set(INCLUDE_DIRS "${PROJECT_SOURCE_DIR}/include")
include_directories(${INCLUDE_DIRS})

link_directories(${CMAKE_LIBRARY_OUTPUT_DIRECTORY})

# Installation

set(INSTALL_TARGETS_DEFAULT_ARGS
    RUNTIME
    DESTINATION
    bin
    LIBRARY
    DESTINATION
    lib
    ARCHIVE
    DESTINATION
    lib)

find_package(octomap REQUIRED HINTS ${CMAKE_SOURCE_DIR}/lib/cmake/octomap
             ${CMAKE_SOURCE_DIR}/../octomap/lib/cmake/octomap)
message(STATUS "Found octomap version: " ${octomap_VERSION})
message(STATUS "octomap libraries: ${OCTOMAP_LIBRARIES}")

include_directories(BEFORE SYSTEM ${OCTOMAP_INCLUDE_DIRS})

add_subdirectory(src)

file(GLOB dynamicEDT3D_HDRS ${PROJECT_SOURCE_DIR}/include/dynamicEDT3D/*.h
     ${PROJECT_SOURCE_DIR}/include/dynamicEDT3D/*.hxx)
install(FILES ${dynamicEDT3D_HDRS}
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/dynamicEDT3D")

# Install catkin package.xml, attention package.xml names the catkin package
# "dynamic_edt_3d", so this is also the location where it needs to be installed
# to (and not "dynamicEDT3D")
install(FILES package.xml DESTINATION "${CMAKE_INSTALL_DATADIR}/dynamic_edt_3d")

# Allows Colcon to find non-Ament packages when using workspace underlays
file(
  WRITE
  ${CMAKE_CURRENT_BINARY_DIR}/share/ament_index/resource_index/packages/${PROJECT_NAME}
  "")
install(
  FILES
    ${CMAKE_CURRENT_BINARY_DIR}/share/ament_index/resource_index/packages/${PROJECT_NAME}
  DESTINATION share/ament_index/resource_index/packages)

# TODO: this conflicts with the octomap uninstall it is not only a target name
# problem, also both will use the same manifest file in the same binary
# directory configure_file(
# "${PROJECT_SOURCE_DIR}/CMakeModules/CMakeUninstall.cmake.in"
# "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake" IMMEDIATE @ONLY)
#
# add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P
# ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

# Export the package for use from the build-tree (this registers the build-tree
# with a global CMake-registry)
export(PACKAGE dynamicEDT3D)

# Create a dynamicEDT3DConfig.cmake file for the use from the build tree
set(DYNAMICEDT3D_INCLUDE_DIRS "${INCLUDE_DIRS}")
set(DYNAMICEDT3D_LIB_DIR "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")
# Set library names as absolute paths Windows, spec. MSVC requires the .lib
# suffix for imported libs
if(WIN32)
  set(DYNAMICEDT3D_LIBRARY
      "${CMAKE_IMPORT_LIBRARY_PREFIX}dynamicedt3d${CMAKE_IMPORT_LIBRARY_SUFFIX}"
  )
else()
  set(DYNAMICEDT3D_LIBRARY
      "${CMAKE_SHARED_LIBRARY_PREFIX}dynamicedt3d${CMAKE_SHARED_LIBRARY_SUFFIX}"
  )
endif()
# not used right now (export depends?) set(DYNEDT3D_CMAKE_DIR
# "${PROJECT_BINARY_DIR}")
include(CMakePackageConfigHelpers)

configure_package_config_file(
  dynamicEDT3DConfig.cmake.in
  "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/cmake/dynamicEDT3D/dynamicEDT3DConfig.cmake"
  PATH_VARS DYNAMICEDT3D_INCLUDE_DIRS DYNAMICEDT3D_LIB_DIR
  INSTALL_DESTINATION "${CMAKE_INSTALL_FULL_DATADIR}/dynamicEDT3D")

write_basic_package_version_file(
  "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/cmake/dynamicEDT3D/dynamicEDT3DConfig-version.cmake"
  VERSION ${DYNAMICEDT3D_VERSION}
  COMPATIBILITY AnyNewerVersion)

# Install the export set for use with the install-tree install(EXPORT
# FooBarLibraryDepends DESTINATION "${INSTALL_DATA_DIR}/FooBar/CMake" COMPONENT
# dev)

# Create a dynamicEDT3DConfig.cmake file for the use from the install tree and
# install it
set(DYNAMICEDT3D_INCLUDE_DIRS "${CMAKE_INSTALL_FULL_INCLUDEDIR}")
set(DYNAMICEDT3D_LIB_DIR "${CMAKE_INSTALL_FULL_LIBDIR}")
# set(DYNAMICEDT3D_CMAKE_DIR "${INSTALL_DATA_DIR}/FooBar/CMake")

set(DYNAMICEDT3D_INCLUDE_TARGETS
    "include(\${CMAKE_CURRENT_LIST_DIR}/dynamicEDT3DTargets.cmake)")

configure_package_config_file(
  dynamicEDT3DConfig.cmake.in
  "${PROJECT_BINARY_DIR}/InstallFiles/dynamicEDT3DConfig.cmake"
  PATH_VARS DYNAMICEDT3D_INCLUDE_DIRS DYNAMICEDT3D_LIB_DIR
  INSTALL_DESTINATION "${CMAKE_INSTALL_FULL_DATADIR}/dynamicEDT3D")

write_basic_package_version_file(
  "${PROJECT_BINARY_DIR}/InstallFiles/dynamicEDT3DConfig-version.cmake"
  VERSION ${DYNAMICEDT3D_VERSION}
  COMPATIBILITY AnyNewerVersion)

install(
  FILES "${PROJECT_BINARY_DIR}/InstallFiles/dynamicEDT3DConfig.cmake"
        "${PROJECT_BINARY_DIR}/InstallFiles/dynamicEDT3DConfig-version.cmake"
  DESTINATION "${CMAKE_INSTALL_DATADIR}/dynamicEDT3D")

# Write pkgconfig-file:
include(InstallPkgConfigFile)
install_pkg_config_file(
  dynamicEDT3D
  CFLAGS
  LIBS
  -ldynamicEDT3D
  REQUIRES
  VERSION
  ${DYNAMICEDT3D_VERSION})

# Documentation
find_package(Doxygen)
if(DOXYGEN_FOUND)
  add_custom_target(
    docs_dynamicEDT3D
    ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/dynamicEDT3D.dox
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    COMMENT "Generating documentation (Doxygen)...")
endif(DOXYGEN_FOUND)

# Needs to be last statement:
include(CPackSettings)

# Finished:
message("\n")
message(STATUS "Compile dynamicEDT3D using: make")
message(STATUS "Install dynamicEDT3D using: make install")
message(STATUS "    (be sure to set the correct CMAKE_INSTALL_PREFIX before)")
message(STATUS "Compile API-documentation using: make docs_dynamicEDT3D\n")
