cmake_minimum_required(VERSION 3.1)
project(rays)

set(TARGET rays)
set(PUBLIC_HDR_DIR include)

# ==================================================================================================
# Sources and headers
# ==================================================================================================
set(PUBLIC_HDRS
        include/rays/PathTracer.h
        include/rays/SimpleMesh.h
        include/rays/SimpleCamera.h
)

set(SRCS
        src/PathTracer.cpp
)

# ==================================================================================================
# Check if embree is available.
# This is an optional dependency and can be installed with homebrew, apt-get, etc.
# ==================================================================================================

find_package(embree 3.0 QUIET PATHS /usr/lib64/cmake)
if (embree_FOUND)
    message("Found embree in ${embree_DIR}, ${EMBREE_INCLUDE_DIRS}")
    add_definitions(-DFILAMENT_HAS_EMBREE)
else()
    message("Embree not found, pipeline features are disabled.")
endif()

# ==================================================================================================
# Include and target definitions
# ==================================================================================================

include_directories(${PUBLIC_HDR_DIR})
include_directories(${EMBREE_INCLUDE_DIRS})
add_library(${TARGET} STATIC ${PUBLIC_HDRS} ${SRCS})
target_link_libraries(${TARGET} PUBLIC math utils image ${EMBREE_LIBRARY})
target_include_directories(${TARGET} PUBLIC ${PUBLIC_HDR_DIR})

# ==================================================================================================
# Compiler flags
# ==================================================================================================

target_compile_options(${TARGET} PRIVATE -Wno-deprecated-register)
if (MSVC OR CLANG_CL)
    target_compile_options(${TARGET} PRIVATE $<$<CONFIG:Release>:/fp:fast>)
else()
    target_compile_options(${TARGET} PRIVATE $<$<CONFIG:Release>:-ffast-math>)
endif()

# ==================================================================================================
# Installation
# ==================================================================================================

install(TARGETS ${TARGET} ARCHIVE DESTINATION lib/${DIST_DIR})
install(DIRECTORY ${PUBLIC_HDR_DIR}/rays DESTINATION include)

# ==================================================================================================
# Tests
# ==================================================================================================

if (NOT ANDROID AND NOT WEBGL AND NOT IOS)
    add_executable(test_${TARGET} tests/test_embree.cpp)
    target_link_libraries(test_${TARGET} PRIVATE rays imageio gtest)
endif()
