cmake_minimum_required(VERSION 3.0.2)
project(octomap-distribution)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
enable_testing() # enable CTest environment of subprojects
set(CMAKE_POSITION_INDEPENDENT_CODE ON) # enables -fPIC in applicable compilers
                                        # (required to avoid link errors in some
                                        # cases)

option(BUILD_OCTOVIS_SUBPROJECT "Build targets from subproject octovis" ON)
option(BUILD_DYNAMICETD3D_SUBPROJECT
       "Build targets from subproject dynamicEDT3D" ON)
option(OCTOVIS_QT5 "Link Octovis against Qt5?" ON)



if(OCTOVIS_QT5)
  # Compiling against QT5 requires C++11.
  set(CMAKE_CXX_STANDARD 11)
endif(OCTOVIS_QT5)

add_subdirectory(octomap)

if(BUILD_OCTOVIS_SUBPROJECT)
  add_subdirectory(octovis)
endif()

if(BUILD_DYNAMICETD3D_SUBPROJECT)
  add_subdirectory(dynamicEDT3D)
endif()
