cmake_minimum_required(VERSION 3.20)

project(Tutti
    VERSION 2.1.1
    DESCRIPTION "A modern C++23 thread pool"
    HOMEPAGE_URL "https://github.com/dqsjqian/Tutti"
    LANGUAGES CXX)

# ---------------------------------------------------------------------------
# Library: header only, C++23, nothing beyond the standard library.
# ---------------------------------------------------------------------------

add_library(tutti INTERFACE)
add_library(tutti::tutti ALIAS tutti)

target_include_directories(tutti INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

# C++23 is the minimum, not a ceiling. On MSVC this CMake maps cxx_std_23 to
# /std:c++latest, which enables the C++26 preview on top of the C++23 features
# Tutti actually uses. Pass -DCMAKE_CXX_STANDARD=26 to ask for C++26 explicitly
# on toolchains that support it.
target_compile_features(tutti INTERFACE cxx_std_23)

find_package(Threads REQUIRED)
target_link_libraries(tutti INTERFACE Threads::Threads)

# Warnings are applied to Tutti's own targets only. A library should not force
# its warning policy onto the projects that consume it.
if(MSVC)
    set(TUTTI_WARNING_FLAGS /W4 /permissive- /utf-8)
else()
    set(TUTTI_WARNING_FLAGS -Wall -Wextra -Wpedantic)
endif()

option(TUTTI_WERROR "Treat warnings in Tutti's own targets as errors" OFF)
if(TUTTI_WERROR)
    if(MSVC)
        list(APPEND TUTTI_WARNING_FLAGS /WX)
    else()
        list(APPEND TUTTI_WARNING_FLAGS -Werror)
    endif()
endif()

# ---------------------------------------------------------------------------
# Standard library side libraries
#
# Two of the C++23 library features Tutti uses are not always part of the main
# runtime library:
#
#   * libc++ has no <stacktrace> header at all, Apple's included, so there is
#     nothing to link against there.
#   * libstdc++ keeps the experimental C++23 library, which includes both
#     std::stacktrace and std::print, in libstdc++exp. GCC 13 put the
#     std::stacktrace symbols in a separate libstdc++_libbacktrace instead.
#   * MSVC and newer libstdc++ need nothing extra.
#
# So probe rather than hardcode: link the snippet on its own, then against each
# candidate library in turn.
# ---------------------------------------------------------------------------

include(CheckCXXSourceCompiles)

set(TUTTI_EXPERIMENTAL_LIBRARIES stdc++exp stdc++_libbacktrace)

set(TUTTI_STACKTRACE_PROBE "
#include <stacktrace>
int main() {
    const std::stacktrace trace = std::stacktrace::current();
    return trace.empty() ? 0 : 1;
}")

set(TUTTI_PRINT_PROBE "
#include <print>
int main() {
    std::println(\"tutti\");
    return 0;
}")

# The probes must compile as C++23 themselves; check_cxx_source_compiles does
# not inherit the standard requirement from target_compile_features, and
# without it <stacktrace> and <print> do not exist at all. Appending rather
# than overwriting lets a caller inject extra flags through
# -DCMAKE_REQUIRED_FLAGS (needed when the standard library has to be selected
# explicitly, e.g. Clang with --gcc-install-dir).
if(MSVC)
    set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} /std:c++latest")
else()
    set(CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS} -std=c++23")
endif()

# Sets <prefix>_OK, and <prefix>_LIBRARIES to whatever had to be linked.
function(tutti_resolve_side_libraries prefix snippet)
    check_cxx_source_compiles("${snippet}" "${prefix}_LINKS_UNAIDED")
    if(${prefix}_LINKS_UNAIDED)
        set(${prefix}_OK TRUE PARENT_SCOPE)
        set(${prefix}_LIBRARIES "" PARENT_SCOPE)
        return()
    endif()

    set(index 0)
    foreach(candidate IN LISTS ARGN)
        math(EXPR index "${index} + 1")
        set(CMAKE_REQUIRED_LIBRARIES "${candidate}")
        check_cxx_source_compiles("${snippet}" "${prefix}_LINKS_WITH_${index}")
        unset(CMAKE_REQUIRED_LIBRARIES)
        if(${prefix}_LINKS_WITH_${index})
            set(${prefix}_OK TRUE PARENT_SCOPE)
            set(${prefix}_LIBRARIES "${candidate}" PARENT_SCOPE)
            return()
        endif()
    endforeach()

    set(${prefix}_OK FALSE PARENT_SCOPE)
    set(${prefix}_LIBRARIES "" PARENT_SCOPE)
endfunction()

tutti_resolve_side_libraries(TUTTI_STACKTRACE "${TUTTI_STACKTRACE_PROBE}"
    ${TUTTI_EXPERIMENTAL_LIBRARIES})

if(TUTTI_STACKTRACE_OK)
    if(TUTTI_STACKTRACE_LIBRARIES)
        message(STATUS "Tutti: std::stacktrace needs -l${TUTTI_STACKTRACE_LIBRARIES}")
        target_link_libraries(tutti INTERFACE ${TUTTI_STACKTRACE_LIBRARIES})
    else()
        message(STATUS "Tutti: std::stacktrace links against the standard library")
    endif()
else()
    # Not fatal: the header falls back to an empty trace so that the pool still
    # builds and runs, it just cannot say where a task came from.
    message(STATUS "Tutti: no usable <stacktrace> here, task provenance is disabled")
    target_compile_definitions(tutti INTERFACE TUTTI_NO_STACKTRACE=1)
endif()

# ---------------------------------------------------------------------------
# Tests
# ---------------------------------------------------------------------------

option(TUTTI_BUILD_TESTS "Build the Tutti test suite" ON)

if(TUTTI_BUILD_TESTS)
    enable_testing()

    add_executable(tutti_tests tests/thread_pool_test.cpp)
    target_link_libraries(tutti_tests PRIVATE tutti::tutti)
    target_compile_options(tutti_tests PRIVATE ${TUTTI_WARNING_FLAGS})
    target_compile_definitions(tutti_tests PRIVATE
        TUTTI_EXPECTED_VERSION_MAJOR=${PROJECT_VERSION_MAJOR}
        TUTTI_EXPECTED_VERSION_MINOR=${PROJECT_VERSION_MINOR}
        TUTTI_EXPECTED_VERSION_PATCH=${PROJECT_VERSION_PATCH})

    add_test(NAME tutti_tests COMMAND tutti_tests)
endif()

# ---------------------------------------------------------------------------
# Examples
# ---------------------------------------------------------------------------

option(TUTTI_BUILD_EXAMPLES "Build the Tutti examples" ON)

if(TUTTI_BUILD_EXAMPLES)
    # std::print is the one side library Tutti's own headers do not pull in, so
    # the probe result is attached to the example rather than to the interface.
    tutti_resolve_side_libraries(TUTTI_PRINT "${TUTTI_PRINT_PROBE}"
        ${TUTTI_EXPERIMENTAL_LIBRARIES})

    if(TUTTI_PRINT_OK)
        add_executable(tutti_example examples/basic_usage.cpp)
        target_link_libraries(tutti_example PRIVATE tutti::tutti)
        target_compile_options(tutti_example PRIVATE ${TUTTI_WARNING_FLAGS})
        if(TUTTI_PRINT_LIBRARIES)
            message(STATUS "Tutti: std::print needs -l${TUTTI_PRINT_LIBRARIES}")
            target_link_libraries(tutti_example PRIVATE ${TUTTI_PRINT_LIBRARIES})
        endif()
    else()
        message(STATUS "Tutti: no usable <print> here, skipping the example")
    endif()
endif()

unset(CMAKE_REQUIRED_FLAGS)

# ---------------------------------------------------------------------------
# Install
# ---------------------------------------------------------------------------

include(GNUInstallDirs)

install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(TARGETS tutti EXPORT TuttiTargets INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(EXPORT TuttiTargets
    NAMESPACE tutti::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Tutti)
