# CMake definitions for SoftFM

cmake_minimum_required(VERSION 3.0.2)
project(airspyhf-fmradion)

# Workaround for the compilation warning
if(COMMAND cmake_policy)
  cmake_policy(SET CMP0003 NEW)
endif(COMMAND cmake_policy)

find_package(Threads)
find_package(PkgConfig)
find_package(Boost 1.47)

# Require ALSA only for Linux
if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
    find_package(ALSA REQUIRED)
endif()

# Find Airspy HF library.
pkg_check_modules(PKG_AIRSPYHF libairspyhf)
if (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    find_path(AIRSPYHF_INCLUDE_DIR airspyhf.h
        HINT /usr/local/include/libairspyhf ${PKG_AIRSPYHF_INCLUDE_DIRS})
    find_library(AIRSPYHF_LIBRARY libairspyhf.a
        HINT /usr/local/lib ${PKG_AIRSPYHF_LIBRARY_DIRS})
    set(AIRSPYHF_INCLUDE_OPTION "-I/usr/local/include")
else()
    find_path(AIRSPYHF_INCLUDE_DIR airspyhf.h
             HINT ${PKG_AIRSPYHF_INCLUDE_DIRS})
    find_library(AIRSPYHF_LIBRARY libairspyhf.a
             HINT ${PKG_AIRSPYHF_LIBRARY_DIRS})
    set(AIRSPYHF_INCLUDE_OPTION "")
endif()

# Find libusb
# See https://github.com/texane/stlink/blob/master/cmake/modules/FindLibUSB.cmake
pkg_check_modules(PKG_LIBUSB libusb-1.0)
    find_path(LIBUSB_INCLUDE_DIR libusb.h
        HINTS /usr /usr/local /opt ${PKG_LIBUSB_INCLUDE_DIRS}
        PATH_SUFFIXES libusb-1.0
    )
    set(LIBUSB_NAME usb-1.0)
find_library(LIBUSB_LIBRARY ${LIBUSB_NAME}
             HINTS /usr /usr/local /opt ${PKG_LIBUSB_LIBRARY_DIRS})

if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(ALSA_OPTION "-DUSE_ALSA")
else ()
    set(ALSA_OPTION "")
endif()

set(AIRSPYHF_INCLUDE_DIRS ${AIRSPYHF_INCLUDE_DIR} ${LIBUSB_INCLUDE_DIR})
set(AIRSPYHF_LIBRARIES    ${AIRSPYHF_LIBRARY} ${LIBUSB_LIBRARY})

# Compiler flags and options.
##
# Enable speed-based optimization
set(CMAKE_CXX_FLAGS "-Wall -std=c++11 -O3 -ffast-math -ftree-vectorize -march=native ${ALSA_OPTION} ${AIRSPYHF_INCLUDE_OPTION} ${EXTRA_FLAGS}")
##
# Use conservative options when failed to run
#set(CMAKE_CXX_FLAGS "-Wall -std=c++11 -O2 ${ALSA_OPTION} ${AIRSPYHF_INCLUDE_OPTION} ${EXTRA_FLAGS}")
##
# For vectorization analysis (in Clang only)
#set(CMAKE_CXX_FLAGS "-Wall -std=c++11 -O3 -ffast-math -ftree-vectorize -march=native -Rpass=loop-vectorize -Rpass-missed=loop-vectorize -Rpass-analysis=loop-vectorize ${ALSA_OPTION} ${AIRSPYHF_INCLUDE_OPTION} ${EXTRA_FLAGS}")
##
# For clang profiling
# set(CMAKE_CXX_FLAGS "-Wall -std=c++11 -g -fprofile-instr-generate -fcoverage-mapping ${ALSA_OPTION} ${AIRSPYHF_INCLUDE_OPTION} ${EXTRA_FLAGS}")
# SET(CMAKE_EXE_LINKER_FLAGS "-fprofile-instr-generate")
##
# For clang profiling with optimization
#set(CMAKE_CXX_FLAGS "-Wall -std=c++11 -O3 -ffast-math -ftree-vectorize -march=native -g -fprofile-instr-generate -fcoverage-mapping ${ALSA_OPTION} ${AIRSPYHF_INCLUDE_OPTION} ${EXTRA_FLAGS}")
#SET(CMAKE_EXE_LINKER_FLAGS "-fprofile-instr-generate")

set(sfmbase_SOURCES
    sfmbase/Filter.cpp
    sfmbase/FmDecode.cpp
    sfmbase/AudioOutput.cpp 
    sfmbase/EqParameters.cpp
)

set(sfmbase_HEADERS
    include/AudioOutput.h
    include/Filter.h
    include/FilterParameters.h
    include/FmDecode.h
    include/MovingAverage.h
    include/Source.h
    include/SoftFM.h
    include/DataBuffer.h
    include/parsekv.h
    include/util.h
    include/EqParameters.h
)

# Base sources

set(sfmbase_SOURCES
    ${sfmbase_SOURCES}
    ${sfmbase_HEADERS}
)

# Airspy HF sources

set(sfmairspyhf_SOURCES
	sfmbase/AirspyHFSource.cpp
)

set(sfmairspyhf_HEADERS
	include/AirspyHFSource.h
)

set(sfmairspyhf_SOURCES
    ${sfmairspyhf_SOURCES}
    ${sfmairspyhf_HEADERS}
)

# Libraries

add_library(sfmbase STATIC
    ${sfmbase_SOURCES}
)

add_library(sfmairspyhf STATIC
    ${sfmairspyhf_SOURCES}
)

add_executable(airspyhf-fmradion
	main.cpp
)

include_directories(
    ${CMAKE_SOURCE_DIR}/include
    ${ALSA_INCLUDE_DIRS}
    ${EXTRA_INCLUDES} 
)

target_link_libraries(airspyhf-fmradion
    sfmbase
    sfmairspyhf
    ${CMAKE_THREAD_LIBS_INIT}
    ${ALSA_LIBRARIES}
    ${EXTRA_LIBS} 
)

target_link_libraries(sfmairspyhf
    ${AIRSPYHF_LIBRARIES}
)

install(TARGETS airspyhf-fmradion DESTINATION bin)
install(TARGETS sfmbase sfmairspyhf DESTINATION lib)
