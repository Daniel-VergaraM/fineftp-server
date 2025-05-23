# CMake 3.8 required for cxx_std_14
cmake_minimum_required(VERSION 3.8...4.0)

include("${CMAKE_CURRENT_LIST_DIR}/version.cmake")
project(server VERSION ${FINEFTP_SERVER_VERSION_MAJOR}.${FINEFTP_SERVER_VERSION_MINOR}.${FINEFTP_SERVER_VERSION_PATCH})

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Disable default export of symbols
set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_VISIBILITY_INLINES_HIDDEN 1)

find_package(Threads REQUIRED)
find_package(asio REQUIRED)

# Include GenerateExportHeader that will create export macros for us
include(GenerateExportHeader)

# Public API include directory
set (includes
    include/fineftp/server.h
    include/fineftp/permissions.h
)

# Private source files
set(sources
    src/filesystem.cpp
    src/filesystem.h
    src/ftp_message.h
    src/ftp_session.cpp
    src/ftp_session.h
    src/ftp_user.h
    src/server.cpp
    src/server_impl.cpp
    src/server_impl.h
    src/user_database.cpp
    src/user_database.h
    src/win_str_convert.cpp
    src/win_str_convert.h
)

if (WIN32)
    list(APPEND sources src/win32/file_man.cpp)
    list(APPEND sources src/win32/file_man.h)
    set(platform_include src/win32)
else()
    list(APPEND sources src/unix/file_man.cpp)
    list(APPEND sources src/unix/file_man.h)
    set(platform_include src/unix)
endif()

add_library (${PROJECT_NAME}
    ${includes}
    ${sources}
)

# Generate version defines
configure_file("fineftp_version.h.in" "${PROJECT_BINARY_DIR}/include/fineftp/fineftp_version.h" @ONLY)

# Generate header with export macros
generate_export_header(${PROJECT_NAME}
  EXPORT_FILE_NAME ${PROJECT_BINARY_DIR}/include/fineftp/fineftp_export.h
  BASE_NAME FINEFTP
)

add_library (fineftp::server ALIAS ${PROJECT_NAME})


target_link_libraries(${PROJECT_NAME}
    PRIVATE
        Threads::Threads
        $<$<BOOL:${WIN32}>:ws2_32>
        $<$<BOOL:${WIN32}>:wsock32>

        # Link header-only libs (asio & recycle) as described in this workaround:
        # https://gitlab.kitware.com/cmake/cmake/-/issues/15415#note_633938
        $<BUILD_INTERFACE:asio::asio>
)

target_compile_definitions(${PROJECT_NAME}
    PRIVATE
        ASIO_STANDALONE
        ASIO_DISABLE_VISIBILITY
        __USE_FILE_OFFSET64=1
        _WIN32_WINNT=0x0601
)

target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_14)

target_compile_options(${PROJECT_NAME} PRIVATE
                           $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:GNU>>:
                                -Wall -Wextra>
                           $<$<CXX_COMPILER_ID:MSVC>:
                                /W4>)

# The below is used to configure an optional delay when sending the 226 response code when a file has been fetched.
# The background for this option is that an FTP client implementation has been observed to close the data connection
# as soon as it receives the 226 status code - even though it hasn't received/ all data, yet. To improve
# interoperability with such buggy clients, sending of the 226 status code can be delayed a bit. The delay is specified
# in milliseconds. If the delay is 0, no delay is introduced.
set(FINEFTP_SERVER_DELAY_226_RESP_MS 0 CACHE STRING
    "An optional delay (in ms) for the 226 response when a file has been fetched. Used to improve interoperability with buggy clients.")
target_compile_definitions(${PROJECT_NAME} PRIVATE DELAY_226_RESP_MS=${FINEFTP_SERVER_DELAY_226_RESP_MS})

# Add own public include directory
target_include_directories(${PROJECT_NAME}
  PUBLIC 
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>          # To find the export file generated by generate_export_header
    $<INSTALL_INTERFACE:include>
  PRIVATE
    src/
    ${platform_include}
)

set_target_properties(${PROJECT_NAME} PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
    OUTPUT_NAME fineftp-${PROJECT_NAME}
)

##################################

source_group(TREE "${CMAKE_CURRENT_SOURCE_DIR}" FILES 
    ${includes}
    ${sources}
)

################################################################################
### Installation rules
################################################################################

set(FINEFTP_INSTALL_CMAKE_DIR "lib/cmake/fineftp")

# Install Runtime & Libs
install(
  TARGETS ${PROJECT_NAME}
  EXPORT fineftpTargets
  
  RUNTIME
    DESTINATION "bin"
    COMPONENT fineftp_server_runtime

  LIBRARY
    DESTINATION "lib"
    COMPONENT fineftp_server_runtime

  ARCHIVE
    DESTINATION "lib"
    COMPONENT fineftp_server_dev
)

# Install public header files (-> dev package)
install(
  DIRECTORY "include/fineftp"
  DESTINATION "include"
  COMPONENT fineftp_server_dev
  FILES_MATCHING PATTERN "*.h"
)

# Install the auto-generated header with the export macros (-> dev package)
install(
  DIRECTORY "${PROJECT_BINARY_DIR}/include/fineftp"
  DESTINATION "include"
  COMPONENT fineftp_server_dev
  FILES_MATCHING PATTERN "*.h"
)

install(
  EXPORT fineftpTargets 
  FILE fineftpTargets.cmake 
  DESTINATION ${FINEFTP_INSTALL_CMAKE_DIR}
  NAMESPACE fineftp::
  COMPONENT fineftp_server_dev
)

# Create and install Config.cmake file (-> dev package)

include(CMakePackageConfigHelpers)

configure_package_config_file(
  "../cmake/fineftpConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake_/fineftpConfig.cmake"
  INSTALL_DESTINATION ${FINEFTP_INSTALL_CMAKE_DIR}
  PATH_VARS FINEFTP_INSTALL_CMAKE_DIR
)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/cmake_/fineftpConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion
)
install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_/fineftpConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_/fineftpConfigVersion.cmake"
  DESTINATION ${FINEFTP_INSTALL_CMAKE_DIR}
  COMPONENT fineftp_server_dev
)

