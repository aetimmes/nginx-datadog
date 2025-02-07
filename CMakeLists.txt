cmake_minimum_required(VERSION 3.7)

# This build configuration combines the following other build configurations:
# - `nginx-datadog.cmake`, the actual nginx module source in `src/`,
# - `nginx-module.cmake`, the nginx-specific boilerplate code generated by
#   nginx's `configure` script,
# - `dd-opentracing-cpp/CMakeLists.txt`, the Datadog tracing library,
# - `opentracing-cpp/CMakeLists.txt`, the OpenTracing library on which the
#   Datadog tracing library is based.

project(ngx_http_datadog_module)

# The C code doesn't care about the C++ version.
# opentracing-cpp works with C++11 or newer.
# dd-opentracing-cpp works with C++14 or newer.
set(CMAKE_CXX_STANDARD 14)

# Nginx module boilerplate (code within the nginx repository)
include(./nginx-module.cmake)

# The actual source code of this repo
include(./nginx-datadog.cmake)

# The opentracing-cpp library sources
add_subdirectory(./opentracing-cpp EXCLUDE_FROM_ALL)

# The dd-opentracing-cpp library sources
SET(BUILD_OBJECT ON CACHE BOOL "Build dd-opentracing-cpp objects")
SET(BUILD_SHARED OFF CACHE BOOL "Don't build dd-opentracing-cpp shared library")
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/dd-opentracing-cpp/3rd_party/sanitizers-cmake" ${CMAKE_MODULE_PATH})

# In addition to the opentracing-cpp header files, we also need to make visible
# header files generated by the opentracing-cpp build (version.h, config.h).
include_directories(opentracing-cpp/include)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/opentracing-cpp/include)
include_directories(opentracing-cpp/3rd_party/include)
add_subdirectory(./dd-opentracing-cpp EXCLUDE_FROM_ALL)
include_directories(dd-opentracing-cpp/include)

# The shared library (nginx module) that we are building.
add_library(ngx_http_datadog_module SHARED)
target_sources(ngx_http_datadog_module
    PRIVATE
        $<TARGET_OBJECTS:nginx_module>
        $<TARGET_OBJECTS:nginx_datadog>
        $<TARGET_OBJECTS:opentracing-object>
        $<TARGET_OBJECTS:dd_opentracing-object>
)

add_dependencies(ngx_http_datadog_module 
    nginx_module 
    nginx_datadog
    opentracing-object
    dd_opentracing-object
)

target_link_libraries(ngx_http_datadog_module dd_opentracing-object -static-libstdc++ -static-libgcc)
