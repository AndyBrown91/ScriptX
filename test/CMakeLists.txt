cmake_minimum_required(VERSION 3.0)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

file(STRINGS ${CMAKE_CURRENT_LIST_DIR}/../VERSION SCRIPTX_VERSION)
project(ScriptXUnitTests VERSION ${SCRIPTX_VERSION} LANGUAGES CXX C)

option(DEVOPS_ENABLE_COVERAGE "enable code coverage" OFF)
enable_testing()

add_executable(UnitTests)
include(cmake/TestEnv.cmake)

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if (TEST_FLAG_ENABLE_ASAN)
        # macOS doesn't support -fsanitize=memory
        string(APPEND UNIT_TEST_ASAN_FLAGS
                " -fsanitize=address"
                " -fsanitize-recover=address"
                " -fno-omit-frame-pointer")
    endif ()

    if (TEST_FLAG_ENABLE_UBSAN)
        # UBSan make UnitTests runs very slow
        # enable as need
        string(APPEND UNIT_TEST_ASAN_FLAGS
                " -fsanitize=undefined"
                " -fsanitize-recover=undefined")
    endif ()

    string(APPEND CMAKE_C_FLAGS ${UNIT_TEST_ASAN_FLAGS})
    string(APPEND CMAKE_CXX_FLAGS ${UNIT_TEST_ASAN_FLAGS})
    string(APPEND CMAKE_EXE_LINKER ${UNIT_TEST_ASAN_FLAGS})

    if (DEVOPS_ENABLE_COVERAGE)
        # devops support gcov only
        if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
            # for clang
            string(APPEND CMAKE_CXX_FLAGS " --coverage")
        else ()
            # for gcc
            string(APPEND CMAKE_CXX_FLAGS " -fprofile-arcs -ftest-coverage")
        endif()
    endif ()
endif ()

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

set(SCRIPTX_DIR ${CMAKE_CURRENT_LIST_DIR}/..)
########### unit test config ###########
# import googletest
include(cmake/gtest/CMakeLists.txt)

target_sources(UnitTests PRIVATE
        src/gtest_main.cc
        src/BookKeepingTest.cc
        src/ScopeTests.cc
        src/ValueTest.cc
        src/NativeTest.cc
        src/CustomConverterTest.cc
        src/Demo.cc
        src/ByteBufferTest.cc
        src/MessageQueueTest.cc
        src/ThreadPoolTest.cc
        src/UtilsTest.cc
        src/ReferenceTest.cc
        src/ManagedObjectTest.cc
        src/InteroperateTest.cc
        src/ExceptionTest.cc
        src/PressureTest.cc
        src/EngineTest.cc
        src/ShowCaseTest.cc
        )

######## ScriptX config ##########

# 1. import ScriptX
#   set which backend engine to use
set(SCRIPTX_BACKEND Hermes CACHE STRING "" FORCE)

# we want the default behavior, so don't set this
# set(SCRIPTX_NO_EXCEPTION_ON_BIND_FUNCTION YES CACHE BOOL "" FORCE)
# set(SCRIPTX_FEATURE_INSPECTOR ON CACHE BOOL "" FORCE)

#   1.2 include ScriptX's CMake file
include(${SCRIPTX_DIR}/CMakeLists.txt)

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" AND TEST_FLAG_ENABLE_CLANG_TIDY)

    set_target_properties(ScriptX PROPERTIES
            CXX_CLANG_TIDY "clang-tidy"
            )

endif ()

include_directories(${DEVOPS_LIBS_INCLUDE})

if (NOT ${SCRIPTX_BACKEND} STREQUAL Hermes)
    target_link_libraries(ScriptX ${DEVOPS_LIBS_LIBPATH})
endif()

target_compile_definitions(ScriptX PUBLIC ${DEVOPS_LIBS_MARCO})

# if you want to close rtti feature for ScriptX and your project
#target_compile_options(ScriptX PRIVATE -fno-rtti)
#target_compile_options(UnitTests PRIVATE -fno-rtti)
target_link_libraries(UnitTests gtest ScriptX)

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    # using clang or gcc
    target_compile_options(ScriptX PRIVATE -Werror -Wall -Wextra -Wno-unused-parameter)

    if (SCRIPTX_TEST_BUILD_ONLY)
        if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
            target_link_options(UnitTests PRIVATE -Wl,-undefined,dynamic_lookup)
        else ()
            target_link_options(UnitTests PRIVATE -Wl,--unresolved-symbols=ignore-in-object-files)
        endif ()
        message(WARNING "SCRIPTX_TEST_BUILD_ONLY is ON, the compiled UnitTests won't run properly. "
                "Compiler is ${CMAKE_CXX_COMPILER_ID}")
    endif ()
elseif (CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    # using Visual Studio C++
    target_compile_options(ScriptX PRIVATE /W4 /WX /Zc:__cplusplus /utf-8 /MP
            # disable warnings
            /wd4100 # unreferenced formal parameter
            /wd4702 # unreachable code
            )

    target_compile_options(UnitTests PRIVATE /MP)
elseif (CMAKE_CXX_COMPILER_ID MATCHES "Intel")
    # using Intel C++
endif ()

if (CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    # using Visual Studio C++
    target_compile_options(UnitTests PRIVATE /utf-8)
endif ()

if (${SCRIPTX_BACKEND} STREQUAL ${SCRIPTX_BACKEND_WEBASSEMBLY})
    configure_file(src/wasm/run_test.html run_test.html COPYONLY)
    target_compile_options(UnitTests PRIVATE
            -sDISABLE_EXCEPTION_CATCHING=0
            )
    target_link_options(UnitTests PRIVATE
            "-sEXPORTED_FUNCTIONS=['_main']"
            "-sEXTRA_EXPORTED_RUNTIME_METHODS=['callMain']")
endif ()

add_test(
        NAME ScriptXUnitTest
        COMMAND UnitTests
)

if (${SCRIPTX_BACKEND} STREQUAL Hermes)
    if (${CMAKE_BUILD_TYPE} STREQUAL "Debug")
        set(HERMES_SRC_DIR "/Users/andy/Desktop/browsers/ScriptX/test/cmake-build-debug/ScriptXTestLibs/hermes")
        set(HERMES_BUILD_DIR "/Users/andy/Desktop/browsers/ScriptX/test/cmake-build-debug/Hermes")
    elseif(${CMAKE_BUILD_TYPE} STREQUAL "Release")
        set(HERMES_SRC_DIR "/Users/andy/Desktop/browsers/ScriptX/test/cmake-build-release/ScriptXTestLibs/hermes")
        set(HERMES_BUILD_DIR "/Users/andy/Desktop/browsers/ScriptX/test/cmake-build-release/Hermes")
    endif()
    if (NOT HERMES_SRC_DIR)
        message(FATAL_ERROR "Please specify HERMES_SRC_DIR")
    endif ()
    # Validate HERMES_SRC_DIR by checking for API/jsi/jsi/jsi.h
    if (NOT EXISTS "${HERMES_SRC_DIR}/API/jsi/jsi/jsi.h")
        message(FATAL_ERROR "HERMES_SRC_DIR does not contain API/jsi/jsi/jsi.h")
    endif ()

    if (NOT HERMES_BUILD_DIR)
        message(FATAL_ERROR "Please specify HERMES_BUILD_DIR")
    endif ()
    # Validate HERMES_BUILD_DIR by checking for bin/hermes with the platform-specific extension
#    if (NOT EXISTS "${HERMES_BUILD_DIR}/bin/hermes${CMAKE_EXECUTABLE_SUFFIX}")
#        message(FATAL_ERROR "HERMES_BUILD_DIR does not contain bin/hermes${CMAKE_EXECUTABLE_SUFFIX}")
#    endif ()

    # Add Hermes include directories
    include_directories("${HERMES_SRC_DIR}/API")
    include_directories("${HERMES_SRC_DIR}/API/jsi")
    include_directories("${HERMES_SRC_DIR}/public")

    # Add Hermes library directories
    link_directories("${HERMES_BUILD_DIR}/API/hermes")
    link_directories("${HERMES_BUILD_DIR}/jsi")

    #hermesvm_a
    #target_link_libraries(ScriptX libhermes)
    target_link_libraries(ScriptX hermesvm_a)
endif()
# add_executable(InspectorTest src/InspectorTest.cc)
# target_link_libraries(InspectorTest gtest ScriptX)

add_executable(HermesTests)
target_sources(HermesTests PRIVATE
    /Users/andy/Desktop/browsers/ScriptX/hermes_main.cc
)
target_link_libraries(HermesTests PUBLIC ScriptX)
target_sources(HermesTests PUBLIC "${HERMES_SRC_DIR}/API/jsi/jsi/jsilib-posix.cpp")

target_include_directories(HermesTests PUBLIC /Users/andy/Desktop/browsers/ScriptX/test)
