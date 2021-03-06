cmake_minimum_required(VERSION 3.13)
project(SetReplace
    VERSION 0.3.0
    LANGUAGES CXX)
message(STATUS "SetReplace version: ${SetReplace_VERSION}")

option(SET_REPLACE_WITH_MATHEMATICA "Add the library definition for Wolfram LibraryLink" OFF)
option(SET_REPLACE_BUILD_TESTING "Enable cpp testing." OFF)
option(SET_REPLACE_WRAP_PYTHON "Wrap python using pybind11" OFF)
include(GNUInstallDirs) # Define CMAKE_INSTALL_xxx: LIBDIR, INCLUDEDIR
set(SetReplace_export_file "${PROJECT_BINARY_DIR}/SetReplaceTargets.cmake")

# The python package is named wolfram_model
set(CMAKE_INSTALL_PYTHONLIBDIR wolfram_model) # Install folder for python libs
set(CMAKE_BUILD_PYTHONLIBDIR ${PROJECT_BINARY_DIR}/wolfram_model) # copy python libs at build time
if(SET_REPLACE_WRAP_PYTHON)
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

# TODO, might be better to have a structure of include/*.h src/*.cpp test/ rather
# than mixing headers with source files in libSetReplace folder as it is now.
# Not sure if that would break things in wolfram side.
set(libSetReplace_headers
    Rule.hpp
    IDTypes.hpp
    Expression.hpp
    Match.hpp
    Set.hpp
    SetReplaceIO.hpp
    )
set(libSetReplace_sources
    Expression.cpp
    Match.cpp
    Set.cpp
    )
if(SET_REPLACE_WITH_MATHEMATICA)
  list(APPEND libSetReplace_headers SetReplace.hpp)
  list(APPEND libSetReplace_sources SetReplace.cpp)
endif()
list(TRANSFORM libSetReplace_headers PREPEND "libSetReplace/")
list(TRANSFORM libSetReplace_sources PREPEND "libSetReplace/")

add_library(SetReplace ${libSetReplace_sources})
target_include_directories(SetReplace PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libSetReplace>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  )

if(SET_REPLACE_WITH_MATHEMATICA)
  find_package(Mathematica REQUIRED)
  target_include_directories(SetReplace ${Mathematica_INCLUDE_DIRS})
  target_link_libraries(SetReplace ${Mathematica_LIBRARIES})
endif()

set(SET_REPLACE_LIBRARIES SetReplace)

install(TARGETS SetReplace
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
export( TARGETS SetReplace
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

if(SET_REPLACE_WRAP_PYTHON)
    # Fetch pybind11
    include(FetchContent)
    FetchContent_Declare(
        pybind11
        GIT_REPOSITORY https://github.com/pybind/pybind11
        GIT_TAG v2.5
        )
    FetchContent_GetProperties(pybind11)
    if(NOT pybind11_POPULATED)
        FetchContent_Populate(pybind11)
        add_subdirectory(${pybind11_SOURCE_DIR} ${pybind11_BINARY_DIR})
    endif()

    add_subdirectory(python)
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
if(SET_REPLACE_WRAP_PYTHON)
  add_dependencies(set-replace-install-runtime _wolfram_model)
endif()
