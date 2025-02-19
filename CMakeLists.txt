cmake_minimum_required(VERSION 3.10)
include(CheckIncludeFiles)
project(phoenix VERSION 1.1.1)

set(CMAKE_CXX_STANDARD 17)
set(PHOENIX_LOG_LEVEL 3 CACHE STRING "The logging level to use for phoenix. Set to 4, 3, 2, or 1 for DEBUG, INFO, WARN or ERROR respectively")

option(PHOENIX_BUILD_EXAMPLES "Build example code" OFF)
option(PHOENIX_BUILD_TESTS "Build tests" ON)
option(PHOENIX_BUILD_SHARED "Build phoenix as a shared library" OFF)
option(PHOENIX_DISABLE_SANITIZERS "Build without sanitizers in debug mode" OFF)
option(PHOENIX_INSTALL "Configure phoenix for cmake install" ON)

set(PHOENIX_DEBUG_POSTFIX d CACHE STRING "Debug library postfix.")

if (MSVC)
    # enable all warnings
    set(PHOENIX_CXX_FLAGS "/W4")

    # in debug mode, enable sanitizers
    if (${CMAKE_BUILD_TYPE} MATCHES "Debug" AND NOT ${PHOENIX_DISABLE_SANITIZERS})
        set(PHOENIX_CXX_FLAGS ${PHOENIX_CXX_FLAGS} "/fsanitize=address")
    endif ()
else ()
    # enable all warnings
    set(PHOENIX_CXX_FLAGS -Wall -Wextra -Werror -Wconversion -Wshadow -Wpedantic)

    # in debug mode, enable sanitizers; note: MinGW does not seem to understand sanitizers on Windows
    if (${CMAKE_BUILD_TYPE} MATCHES "Debug" AND NOT ${PHOENIX_DISABLE_SANITIZERS} AND NOT WIN32)
        set(PHOENIX_CXX_FLAGS ${PHOENIX_CXX_FLAGS} -fsanitize=address -fsanitize=undefined)

        # when not compiling for MacOS, enable leak sanitizer
        if (NOT APPLE)
            set(PHOENIX_CXX_FLAGS ${PHOENIX_CXX_FLAGS} -fsanitize=leak)
        endif ()
    endif ()

    # in debug mode on Clang to get proper debugging support, add -fstandalone-debug
    if (${CMAKE_BUILD_TYPE} MATCHES "Debug" AND ${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
        set(PHOENIX_CXX_FLAGS ${PHOENIX_CXX_FLAGS} -fstandalone-debug)
    endif ()
endif ()

add_subdirectory(vendor)

# add log level definition
set(PHOENIX_DEFINES PHOENIX_LOG_LEVEL=${PHOENIX_LOG_LEVEL})

# find all header files; required for them to show up properly in VisualStudio
file(GLOB_RECURSE PHOENIX_HEADERS "${CMAKE_CURRENT_SOURCE_DIR}/include/**/*.hh")

set(PHOENIX_SOURCES
        source/animation.cc
        source/archive.cc
        source/archive/archive_ascii.cc
        source/archive/archive_binary.cc
        source/archive/archive_binsafe.cc
        source/buffer.cc
        source/font.cc
        source/material.cc
        source/math.cc
        source/mesh.cc
        source/messages.cc
        source/model.cc
        source/model_hierarchy.cc
        source/model_mesh.cc
        source/model_script.cc
        source/model_script_dsl.cc
        source/morph_mesh.cc
        source/phoenix.cc
        source/proto_mesh.cc
        source/save_game.cc
        source/script.cc
        source/softskin_mesh.cc
        source/texture.cc
        source/vdfs.cc
        source/Vfs.cc
        source/vobs/camera.cc
        source/vobs/light.cc
        source/vobs/misc.cc
        source/vobs/mob.cc
        source/vobs/sound.cc
        source/vobs/trigger.cc
        source/vobs/vob.cc
        source/vobs/zone.cc
        source/vm.cc
        source/world.cc
        source/world/bsp_tree.cc
        source/world/vob_tree.cc
        source/world/way_net.cc)

set(PHOENIX_EXTENSIONS
        source/ext/dds_convert.cc
        source/ext/daedalus_classes.cc)

set(PHOENIX_TESTS
        tests/test_animation.cc
        tests/test_archive.cc
        tests/test_buffer.cc
        tests/test_font.cc
        tests/test_material.cc
        tests/test_messages.cc
        tests/test_model.cc
        tests/test_model_hierarchy.cc
        tests/test_model_mesh.cc
        tests/test_model_script.cc
        tests/test_morph_mesh.cc
        tests/test_proto_mesh.cc
        tests/test_save_game.cc
        tests/test_script.cc
        tests/test_texture.cc
        tests/TestVfs.cc
        tests/test_vobs_g1.cc
        tests/test_vobs_g2.cc
        tests/test_world.cc)

# add the phoenix library definition
if (BUILD_SHARED_LIBS AND PHOENIX_BUILD_SHARED)
    add_library(phoenix SHARED)
    set_target_properties(phoenix PROPERTIES CXX_VISIBILITY_PRESET hidden VISIBILITY_INLINES_HIDDEN 1)
else ()
    add_library(phoenix STATIC)
    set(PHOENIX_DEFINES ${PHOENIX_DEFINES} PHOENIX_STATIC=1)
    set_target_properties(phoenix PROPERTIES COMPILE_FLAGS "-DPHOENIX_STATIC=1")
endif ()

target_sources(phoenix PRIVATE ${PHOENIX_SOURCES} ${PHOENIX_EXTENSIONS} ${PHOENIX_HEADERS})
target_include_directories(phoenix PUBLIC include)

# Apps using phoenix will also need to link to glm, fmt and squish to avoid missing symbols
target_link_libraries(phoenix
        PUBLIC glm::glm_static squish
        PRIVATE mio)

target_compile_definitions(phoenix PUBLIC ${PHOENIX_DEFINES} PRIVATE PHOENIX_EXPORTS=1)
target_compile_options(phoenix PRIVATE ${PHOENIX_CXX_FLAGS})

if (NOT MSVC)
    target_link_options(phoenix PUBLIC ${PHOENIX_CXX_FLAGS})
endif ()

set_target_properties(phoenix PROPERTIES
        DEBUG_POSTFIX "${PHOENIX_DEBUG_POSTFIX}"
        VERSION ${PROJECT_VERSION})

if (PHOENIX_INSTALL)
    install(TARGETS phoenix ARCHIVE LIBRARY RUNTIME)
    install(DIRECTORY "include/phoenix" TYPE INCLUDE)

    if (NOT PHOENIX_BUILD_SHARED)
        # For static linking we'll need to provide the dependency static libraries
        install(DIRECTORY "${glm_SOURCE_DIR}/glm" TYPE INCLUDE FILES_MATCHING PATTERN "*.hpp" PATTERN "*.inl" PATTERN "*.h")

        foreach (lib glm::glm_static squish)
            install(FILES "$<TARGET_LINKER_FILE:${lib}>" TYPE LIB)
        endforeach ()
    endif ()
endif ()

# when building tests, create a test executable and load it into CTest
if (PHOENIX_BUILD_TESTS AND CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    enable_testing()
    include(${doctest_SOURCE_DIR}/scripts/cmake/doctest.cmake)

    add_executable(phoenix-tests ${PHOENIX_TESTS})
    target_link_libraries(phoenix-tests PRIVATE phoenix doctest_with_main)
    target_compile_options(phoenix-tests PRIVATE ${PHOENIX_CXX_FLAGS})

    if (NOT MSVC)
        target_compile_options(phoenix-tests PRIVATE -Wno-overloaded-shift-op-parentheses -Wno-deprecated-declarations)
    endif ()

    target_link_options(phoenix-tests PRIVATE ${PHOENIX_CXX_FLAGS})

    doctest_discover_tests(phoenix-tests EXTRA_ARGS -tse=messages,script,world WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}/tests)
endif ()

# when building examples, include the subdirectory
if (PHOENIX_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif ()
