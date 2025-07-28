cmake_minimum_required(VERSION 3.23)
project(BasioProject LANGUAGES CXX)

# Enable folders in IDEs (optional)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# GoogleTest requires at least C++23
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Avoid warnings by using new policy
cmake_policy(SET CMP0135 NEW)

include(FetchContent)

# ------ Fetch Google Test ------
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG v1.15.2
  GIT_SHALLOW TRUE
)
# For Windows: Prevent overriding the parent project's compiler/linker settings
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

# ------ Fetch standalone Asio ------
# Official repository: https://github.com/chriskohlhoff/asio
FetchContent_Declare(
  asio
  GIT_REPOSITORY https://github.com/chriskohlhoff/asio.git
  GIT_TAG asio-1-34-2
  GIT_SHALLOW TRUE
)
FetchContent_MakeAvailable(asio)

# Create an interface library for asio
add_library(asio INTERFACE)
target_include_directories(asio INTERFACE ${asio_SOURCE_DIR}/asio/include)
target_compile_definitions(asio INTERFACE ${ASIO_DEFS})

# Ensure pthread is used on Unix-like systems
if(UNIX AND NOT APPLE)
  set(THREADS_PREFER_PTHREAD_FLAG ON)
elseif(APPLE)
  find_package(Threads REQUIRED)
endif()

# Helper to link system libs needed by Asio (sockets, threads)
function(link_platform_libs target)
  if(WIN32)
    target_link_libraries(${target} PRIVATE ws2_32) # Winsock
  else()
    target_link_libraries(${target} PRIVATE Threads::Threads)
  endif()
endfunction()

# Shared definitions for standalone Asio
set(ASIO_DEFS
  ASIO_STANDALONE            # use standalone (no Boost)
  ASIO_NO_DEPRECATED         # optional: hide deprecated API
)

# ------ Issuer ----- #
set (TARGET_ISS "issuer")
file(GLOB LIB_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/src/${TARGET_ISS}/*.h" "${CMAKE_CURRENT_SOURCE_DIR}/src/${TARGET_ISS}/*.cpp")

add_executable(${TARGET_ISS} ${LIB_SOURCES})
set_target_properties(${TARGET_ISS} PROPERTIES 
    LINKER_LANGUAGE CXX
    CXX_STANDARD 23
    CXX_STANDARD_REQUIRED ON
)

# Set the library version (optional)
set_target_properties(${TARGET_ISS} PROPERTIES VERSION 1.0 SOVERSION 1)

target_link_libraries(${TARGET_ISS} PRIVATE asio)
target_compile_definitions(${TARGET_ISS} PRIVATE ${ASIO_DEFS})
target_include_directories(${TARGET_ISS} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
link_platform_libs(${TARGET_ISS})

install(TARGETS ${TARGET_ISS} DESTINATION lib)


# ------ Client ----- #
set (TARGET_CLIENT "client")
file(GLOB LIB_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/src/${TARGET_CLIENT}/*.h" "${CMAKE_CURRENT_SOURCE_DIR}/src/${TARGET_CLIENT}/*.cpp")

add_executable(${TARGET_CLIENT} ${LIB_SOURCES})
set_target_properties(${TARGET_CLIENT} PROPERTIES 
    LINKER_LANGUAGE CXX
    CXX_STANDARD 23
    CXX_STANDARD_REQUIRED ON
)

# Set the library version (optional)
set_target_properties(${TARGET_CLIENT} PROPERTIES VERSION 1.0 SOVERSION 1)

target_link_libraries(${TARGET_CLIENT} PRIVATE asio)
target_compile_definitions(${TARGET_CLIENT} PRIVATE ${ASIO_DEFS})
target_include_directories(${TARGET_CLIENT} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
link_platform_libs(${TARGET_CLIENT})

install(TARGETS ${TARGET_CLIENT} DESTINATION lib)

# ------ Tests ------
enable_testing()

# UUID tests
add_executable(test_uuid tests/test_uuid.cpp)
target_include_directories(test_uuid PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_link_libraries(test_uuid PRIVATE gtest_main asio)
target_compile_definitions(test_uuid PRIVATE ${ASIO_DEFS})
link_platform_libs(test_uuid)
set_target_properties(test_uuid PROPERTIES 
    CXX_STANDARD 23
    CXX_STANDARD_REQUIRED ON
)

# Network integration tests
add_executable(test_network tests/test_network.cpp)
target_include_directories(test_network PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_link_libraries(test_network PRIVATE gtest_main asio)
target_compile_definitions(test_network PRIVATE ${ASIO_DEFS})
link_platform_libs(test_network)
set_target_properties(test_network PROPERTIES 
    CXX_STANDARD 23
    CXX_STANDARD_REQUIRED ON
)

include(GoogleTest)
gtest_discover_tests(test_uuid)
gtest_discover_tests(test_network)