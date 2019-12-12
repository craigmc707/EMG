cmake_minimum_required(VERSION 3.13.0)

#===============================================================================
# USER OPTIONS
#===============================================================================
option(NI_X64      "Turn ON if compiling for NI Intel x64 embeded hardware"                    OFF)
option(EMG_STATIC "Turn ON to build EMG as a static library (Default is shared)"               ON)
option(MOVE_BINS   "Turn ON to move binariest to conventional bin/lib folders at compilation"  ON)

option(QUANSER     "Turn ON if compiling for Quanser hardware"                                 ON)
option(MYO         "Turn ON to build Thalmic Labs Myo Armband."                                OFF)
option(EXAMPLES    "Turn ON to build example executable(s)."                                   OFF)
option(TESTS       "Turn ON to build test executable(s)"                                       OFF)

set(NI_X64_ROOT  "C:/dev/nirlt-linux-x64"       CACHE FILEPATH "Absolute path to NI Linux Realtime GNU toolchain (x64).")
set(QUANSER_ROOT "C:/Program Files/Quanser"     CACHE FILEPATH "Absolute path to Quanser installation.")
set(EIGEN_ROOT   "C:/dev/eigen"                 CACHE FILEPATH "Absolute path to Eigen installation.")
set(MYO_ROOT     "C:/dev/myo-sdk-win-0.9.0"     CACHE FILEPATH "Absolute path to Myo SDK.")


# enable C/C++
enable_language(C)
enable_language(CXX)

# enable C++11
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CONFIGURATION_TYPES Release)

# create project
project(EMG)

find_package(MEL REQUIRED)

# set binary output locations
if(MOVE_BINS)
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/lib)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/lib)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin)
endif(MOVE_BINS)


# add definitons
add_definitions(-D_CRT_SECURE_NO_WARNINGS -DNOMINMAX -D_WINSOCK_DEPRECATED_NO_WARNINGS)

# set compiler flags
if (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} -Wall -O3 -ldl") # all warnings
elseif(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} /W3") # warning level 4
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} /MP") # multicore build
endif()

#===============================================================================
# INCLUDES
#===============================================================================

# MEL Lib and Eigen Lib
set(INCLUDE_DIRS include ${EIGEN_ROOT})#{MEL_ROOT}/include ${EIGEN_ROOT})
if (MYO)
    list(APPEND INCLUDE_DIRS ${MYO_ROOT}/include)
endif()

# include directories
include_directories(${INCLUDE_DIRS})

#===============================================================================
# FIND PROJECT SOURCE FILES
#===============================================================================

file(GLOB SRC_EMG
    "${CMAKE_SOURCE_DIR}/src/EMG/EMG/*.cpp"
    "${CMAKE_SOURCE_DIR}/include/EMG/EMG/*.hpp"
)

file(GLOB SRC_SIGNAL_PROCESSING
    "${CMAKE_SOURCE_DIR}/src/EMG/SignalProcessing/*.cpp"
    "${CMAKE_SOURCE_DIR}/include/EMG/SignalProcessing/*.hpp"
)

file(GLOB SRC_CLASSIFICATION
    "${CMAKE_SOURCE_DIR}/src/EMG/Classification/*.cpp"
    "${CMAKE_SOURCE_DIR}/include/EMG/Classification/*.hpp"
)

file(GLOB SRC_REGRESSION
    "${CMAKE_SOURCE_DIR}/src/EMG/Regression/*.cpp"
    "${CMAKE_SOURCE_DIR}/include/EMG/Regression/*.hpp"
)

file(GLOB SRC_UTILITY
    "${CMAKE_SOURCE_DIR}/src/EMG/Utility/*.cpp"
    "${CMAKE_SOURCE_DIR}/include/EMG/Utility/*.hpp"
)

# EMG library source files
set(EMG_SOURCE_FILES
    # ${SRC_EMG}
    ${SRC_SIGNAL_PROCESSING}
    # ${SRC_CLASSIFICATION}
    # ${SRC_REGRESSION}
    ${SRC_UNITY}
    ${SRC_UTILITY}
)

# create filters for Visual Studio
source_group("Control" FILES ${SRC_CONTROL})
source_group("EMG" FILES ${SRC_EMG})
source_group("SignalProcessing" FILES ${SRC_SIGNAL_PROCESSING})
source_group("Classification" FILES ${SRC_CLASSIFICATION})
source_group("Regression" FILES ${SRC_REGRESSION})
source_group("Utility" FILES ${SRC_UTILITY})

#===============================================================================
# CREATE EMG LIBRARY
#===============================================================================

# add_library(EMG STATIC ${EMG_SOURCE_FILES})
if (EMG_STATIC)
    add_definitions(-DEMG_STATIC)
endif()

#===============================================================================
# LINK EXTERNAL LIBS
#===============================================================================

set(LINK_LIBS "")
if(QUANSER)
    if (CMAKE_SIZEOF_VOID_P EQUAL 8) # 64-bit
        link_directories("${QUANSER_ROOT}/QUARC/lib/win64")
    else() # 32-bit
        link_directories("${QUANSER_ROOT}/QUARC/lib/windows")
    endif()
endif(QUANSER)

# list(APPEND LINK_LIBS MEL::Quanser)

if(win32)
    list(APPEND LINK_LIBS
    ws2_32
    winmm
    Pdh
    Psapi
    hil
    quanser_communications
    quanser_runtime
    quanser_common
    bufferoverflowU
    legacy_stdio_definitions)
endif()


if (MYO)
    link_directories("${MYO_ROOT}/lib")
    if (CMAKE_SIZEOF_VOID_P EQUAL 8) # 64-bit
        list(APPEND LINK_LIBS myo64)
    else() # 32-bit
        list(APPEND LINK_LIBS myo32)
    endif()
endif()

if (EMG_STATIC)
    add_library(EMG STATIC ${EMG_SOURCE_FILES})
else()
    add_library(EMG SHARED ${EMG_SOURCE_FILES})
endif()

target_link_libraries(EMG MEL::quanser)#${LINK_LIBS})
target_compile_definitions(EMG PRIVATE -DEMG_EXPORTS)



#===============================================================================
# CREATE EXECUTABLES
#===============================================================================


if (MYO)
    add_executable(myo_armband_ml src/myo_armband_ml.cpp)
    target_link_libraries(myo_armband_ml EMG)
endif()

if(EXAMPLES)
    add_subdirectory(examples)
endif()

if(TESTS)
    add_executable(testing tests/testing.cpp)
    target_link_libraries(testing EMG)
endif()
