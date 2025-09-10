cmake_minimum_required(VERSION 3.24)

# ---- Project ----

# Note: update this to your new project's name and version
project(
  Greeter
  VERSION 1.0.0.1
  LANGUAGES CXX
)

# ---- Include guards ----

if(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
  message(
    FATAL_ERROR
      "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there."
  )
endif()

# ---- Add dependencies via CPM ----
# see https://github.com/TheLartians/CPM.cmake for more info

include(cmake/CPM.cmake)

# PackageProject.cmake will be used to make our target installable
include(cmake/PackageProject.cmake)
# XXX # CPMAddPackage("gh:TheLartians/PackageProject.cmake@1.13.0")

# XXX set(CMAKE_SKIP_INSTALL_RULES YES)

CPMAddPackage(
  NAME fmt
  GIT_TAG 11.1.4
  GITHUB_REPOSITORY fmtlib/fmt
  OPTIONS "FMT_INSTALL YES" # create an installable target
)

CPMAddPackage(
  NAME Format.cmake
  VERSION 1.8.3
  GITHUB_REPOSITORY TheLartians/Format.cmake
  OPTIONS
      # set to yes skip cmake formatting
      "FORMAT_SKIP_CMAKE NO"
      # set to yes skip clang formatting
      "FORMAT_SKIP_CLANG NO"
      # path to exclude (optional, supports regular expressions)
      #"CMAKE_FORMAT_EXCLUDE cmake/CPM.cmake"
      # extra arguments for cmake_format (optional)
      #"CMAKE_FORMAT_EXTRA_ARGS -c /path/to/cmake-format.{yaml,json,py}"
)

set(CMAKE_VERIFY_INTERFACE_HEADER_SETS ${PROJECT_IS_TOP_LEVEL})

# ---- Add source files ----

# Note: globbing sources is considered bad practice as CMake's generators may not detect new files
# automatically. Keep that in mind when changing files, or explicitly mention them here.
file(GLOB_RECURSE headers CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/include/*.h")
file(GLOB_RECURSE sources CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/source/*.cpp")

# ---- Create library ----

# the location where the project's version header will be placed should match the project's regular
# header paths
string(TOLOWER ${PROJECT_NAME}/version.h VERSION_HEADER_LOCATION)

# Note: for header-only libraries change all PUBLIC flags to INTERFACE and create an interface
# target: add_library(${PROJECT_NAME} INTERFACE)
add_library(${PROJECT_NAME})
target_sources(
  ${PROJECT_NAME}
  PRIVATE ${sources}
  PUBLIC FILE_SET
         public_headers
         TYPE
         HEADERS
         BASE_DIRS
         ${CMAKE_CURRENT_SOURCE_DIR}/include
         ${CMAKE_CURRENT_BINARY_DIR}
         FILES
         ${headers}
         ${CMAKE_CURRENT_BINARY_DIR}/${VERSION_HEADER_LOCATION}
)
set_target_properties(${PROJECT_NAME} PROPERTIES CXX_STANDARD 20)

# being a cross-platform target, we enforce standards conformance on MSVC
target_compile_options(${PROJECT_NAME} PUBLIC "$<$<COMPILE_LANG_AND_ID:CXX,MSVC>:/permissive->")

# Link dependencies
target_link_libraries(${PROJECT_NAME} PRIVATE fmt::fmt)

set(GREETER_VERSION \"${PROJECT_VERSION}\")
string(TOUPPER ${PROJECT_NAME} UPPERCASE_PROJECT_NAME)
configure_file(${PACKAGE_PROJECT_ROOT_PATH}/version.h.in ${VERSION_HEADER_LOCATION} @ONLY)

if(CMAKE_SKIP_INSTALL_RULES)
  add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})
  return()
endif()

# ---- Create an installable target ----
# this allows users to install and find the library via `find_package()`.

include(cmake/AddUninstallTarget.cmake)

packageProject(
  NAME ${PROJECT_NAME}
  VERSION ${PROJECT_VERSION}
  NAMESPACE ${PROJECT_NAME}
  BINARY_DIR ${PROJECT_BINARY_DIR}
  # Not used! INCLUDE_DIR ${PROJECT_SOURCE_DIR}/include
  COMPATIBILITY SameMajorVersion
  RUNTIME_DESTINATION /
  DEPENDENCIES "fmt 11.1.4"
  HEADER_SETS public_headers
)
