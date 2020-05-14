cmake_minimum_required(VERSION 3.13)
project(SetReplace
    VERSION 0.3.0
    LANGUAGES CXX)
message(STATUS "SetReplace version: ${SetReplace_VERSION}")

option(SET_REPLACE_WITH_MATHEMATICA "Add the library definition for Wolfram LibraryLink" OFF)
option(SET_REPLACE_BUILD_TESTING "Enable cpp testing." OFF)
include(GNUInstallDirs) # Define CMAKE_INSTALL_xxx: LIBDIR, INCLUDEDIR
set(SetReplace_export_file "${PROJECT_BINARY_DIR}/SetReplaceTargets.cmake")

set(default_build_type "Release")
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
    set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
        STRING "Choose the type of build." FORCE)
    # Set the possible values of build type for cmake-gui
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY
        STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

set(libSetReplace_headers
    Rule.hpp
    IDTypes.hpp
    Expression.hpp
    Match.hpp
    Set.hpp
    )
set(libSetReplace_sources
    Expression.cpp
    Match.cpp
    Set.cpp
    )
if(SET_REPLACE_WITH_MATHEMATICA)
  list(APPEND libSetReplace_headers SetReplace.hpp)
endif()
list(TRANSFORM libSetReplace_headers PREPEND "libSetReplace/")
list(TRANSFORM libSetReplace_sources PREPEND "libSetReplace/")

add_library(SetReplace ${libSetReplace_sources})
target_include_directories(SetReplace PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libSetReplace>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  )

set(SET_REPLACE_LIBRARIES SetReplace)

if(SET_REPLACE_WITH_MATHEMATICA)
  set(libSetReplaceMathematica_sources
    SetReplace.cpp)
  list(TRANSFORM libSetReplaceMathematica_sources PREPEND "libSetReplace/")
  add_library(SetReplaceMathematica ${libSetReplaceMathematica_sources})
  target_link_libraries(SetReplaceMathematica SetReplace)
  list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/cmake)
  find_package(Mathematica REQUIRED)
  target_include_directories(SetReplaceMathematica PUBLIC ${Mathematica_INCLUDE_DIR})
  target_link_libraries(SetReplaceMathematica ${Mathematica_LIBRARIES})
  list(APPEND SET_REPLACE_LIBRARIES SetReplaceMathematica)
endif()


install(TARGETS ${SET_REPLACE_LIBRARIES}
        EXPORT SetReplaceTargets
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT runtime
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT runtime
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT development
        )

install(FILES ${libSetReplace_headers}
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/SetReplace
  COMPONENT development
  )

# export to the build tree
export( TARGETS ${SET_REPLACE_LIBRARIES}
  NAMESPACE SetReplace::
  APPEND FILE ${SetReplace_export_file})


if(SET_REPLACE_BUILD_TESTING)
  enable_testing()
  set(INSTALL_GTEST OFF)
  set(CMAKE_POLICY_DEFAULT_CMP0077 NEW) # Propagate INSTALL_GTEST=OFF to subproject
  set(GTEST_LIBRARIES gtest gtest_main)
  include(GoogleTest)
  #############################################################################
  # Fetch GTest
  include(FetchContent)

  FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.8.x
  )

  set(CMAKE_POLICY_DEFAULT_CMP0048 NEW) # google test raises warning about it
  FetchContent_GetProperties(googletest)
  if(NOT googletest_POPULATED)
    FetchContent_Populate(googletest)
    add_subdirectory(${googletest_SOURCE_DIR} ${googletest_BINARY_DIR})
  endif()
  #############################################################################

  add_subdirectory(libSetReplace/test)
endif()

# INSTALL
set(install_cmake_dir "${CMAKE_INSTALL_LIBDIR}/cmake/SetReplace")

install (EXPORT SetReplaceTargets
  NAMESPACE SetReplace::
  DESTINATION ${install_cmake_dir} )

install(FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/FindMathematica.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/cmake/SetReplaceConfig.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/SetReplaceConfigVersion.cmake
  DESTINATION ${install_cmake_dir} )

include(CMakePackageConfigHelpers)

write_basic_package_version_file(SetReplaceConfigVersion.cmake
  VERSION ${SetReplace_VERSION}
  COMPATIBILITY SameMajorVersion)

# Build tree
set(SetReplace_TARGETS_FILE ${SetReplace_export_file})
configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/SetReplaceConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/SetReplaceConfig.cmake
  INSTALL_DESTINATION ${install_cmake_dir}
  PATH_VARS SetReplace_TARGETS_FILE
  NO_CHECK_REQUIRED_COMPONENTS_MACRO # SetReplace does not provide components
  INSTALL_PREFIX ${CMAKE_CURRENT_BINARY_DIR}
  )

# Install tree
set(SetReplace_TARGETS_FILE ${CMAKE_INSTALL_PREFIX}/${install_cmake_dir}/SetReplaceTargets.cmake)
configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/SetReplaceConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/cmake/SetReplaceConfig.cmake
  INSTALL_DESTINATION ${install_cmake_dir}
  PATH_VARS SetReplace_TARGETS_FILE
  NO_CHECK_REQUIRED_COMPONENTS_MACRO # SetReplace does not provide components
  )

# Add custom target to only install component: runtime (libraries)
add_custom_target(set-replace-install-runtime
  ${CMAKE_COMMAND}
  -DCMAKE_INSTALL_COMPONENT=runtime
  -P "${PROJECT_BINARY_DIR}/cmake_install.cmake"
  DEPENDS ${SET_REPLACE_LIBRARIES}
  )
message(STATUS "SET_REPLACE_LIBRARIES: ${SET_REPLACE_LIBRARIES}")
add_dependencies(set-replace-install-runtime ${SET_REPLACE_LIBRARIES})
