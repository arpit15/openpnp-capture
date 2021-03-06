#
# CMAKE build file for OpenPnP Capture library
#
# This generates make files for several build systems,
# such as GNU Make, Ninja and visual studio
#
# When invoking on Windows systems, make sure the
# compiler (Visual Studio) is in the search path.
#
# Author: Niels A. Moseley, Jason von Nieda
#

cmake_minimum_required(VERSION 3.1)
project (openpnp-capture)

# make sure the libjpegturbo is compiled with the
# position independent flag -fPIC
IF (UNIX)
    set(POSITION_INDEPENDENT_CODE TRUE)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
ENDIF()

# make CMAKE search the current cmake dir inside the
# current project
set (CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# force C++11 standard
set(CMAKE_CXX_STANDARD 11)

# load module that gets info from GIT 
# see: http://brianmilco.blogspot.nl/2012/11/cmake-automatically-use-git-tags-as.html
include(GetGitRevisionDescription)

# create library version from GIT tag using cmake/version.h.in as a template
# and write it to common/version.h
git_describe(GITVERSION --tags)
MESSAGE(STATUS "Using GIT tag: " ${GITVERSION} )
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/version.h.in ${CMAKE_CURRENT_SOURCE_DIR}/common/version.h)

# determine number of bits of compiler
if( CMAKE_SIZEOF_VOID_P EQUAL 8 )
  set( COMPILERBITS "64 bit")
else( CMAKE_SIZEOF_VOID_P EQUAL 8 )
  set( COMPILERBITS "32 bit")
endif( CMAKE_SIZEOF_VOID_P EQUAL 8 )

# check the build type and set the build type string
if(CMAKE_BUILD_TYPE MATCHES Release)
    add_definitions(-D__BUILDTYPE__="release")
else(CMAKE_BUILD_TYPE MATCHES Release)
    add_definitions(-D__BUILDTYPE__="debug")
endif(CMAKE_BUILD_TYPE MATCHES Release)


# add include directory 
include_directories(include)

IF (WIN32)
	# build with static runtime rather than DLL based so that we
	# don't have to distribute it
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")

    # set the platform identification string
    add_definitions(-D__PLATFORM__="Win ${COMPILERBITS}")
    
    # remove annoying 'unsafe' function warnings
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)

    # add files for WIN32
    set (SOURCE common/libmain.cpp
                common/context.cpp
                common/logging.cpp
                common/stream.cpp
                win/platformcontext.cpp
                win/platformstream.cpp)

    # create the library
    add_library(openpnp-capture SHARED ${SOURCE})

    # add windows-specific test application
#     add_subdirectory(win/tests)

ELSEIF(APPLE)
    # set the platform identification string
    add_definitions(-D__PLATFORM__="OSX ${COMPILERBITS}")

    set (SOURCE common/libmain.cpp
                common/context.cpp
                common/logging.cpp
                common/stream.cpp
                mac/platformcontext.mm
                mac/platformstream.mm
                mac/uvcctrl.mm)


    # create the library
    add_library(openpnp-capture SHARED ${SOURCE})

    # include OS X specific frameworks
    target_link_libraries(openpnp-capture
        "-framework AVFoundation"
        "-framework Foundation"
        "-framework CoreMedia"
        "-framework CoreVideo"
        "-framework Accelerate"
        "-framework IOKit"
        )

    # add mac specific test application
#     add_subdirectory(mac/tests)

ELSEIF(UNIX)

    # compile libjpeg-turbo for MJPEG decoding support
    # right now, we need to disable SIMD because it
    # causes a compile problem.. we need to fix this
    # later...
    
    set(ENABLE_SHARED OFF)
    set(WITH_SIMD OFF)      
    add_subdirectory(linux/contrib/libjpeg-turbo-dev)

    # set the platform identification string
    add_definitions(-D__PLATFORM__="Linux ${COMPILERBITS}")

    set (SOURCE common/libmain.cpp
                common/context.cpp
                common/logging.cpp
                common/stream.cpp
                linux/platformcontext.cpp
                linux/platformstream.cpp
                linux/mjpeghelper.cpp
                linux/yuvconverters.cpp)

    # force include directories for libjpeg-turbo
    include_directories(SYSTEM "${CMAKE_CURRENT_SOURCE_DIR}/linux/contrib/libjpeg-turbo-dev")
    
    # create our capture library
    add_library(openpnp-capture SHARED ${SOURCE})

    # add pthreads library 
    set(THREADS_PREFER_PTHREAD_FLAG ON)
    find_package(Threads REQUIRED)    
    target_link_libraries(openpnp-capture Threads::Threads)

    # add turbojpeg-static library
    target_link_libraries(openpnp-capture turbojpeg-static)

    # add linux-specific test application
#     add_subdirectory(linux/tests)

ENDIF()

# install
target_include_directories(openpnp-capture 
        PUBLIC 
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/common>
            $<INSTALL_INTERFACE:include/openpnp-capture/include>
            $<INSTALL_INTERFACE:include/openpnp-capture/common>)

install(TARGETS openpnp-capture
    EXPORT openpnp-capture
    PUBLIC_HEADER 
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/openpnp-capture)

install(DIRECTORY $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/common>
            DESTINATION include/openpnp-capture
            FILES_MATCHING PATTERN "*.h")

install(EXPORT openpnp-capture
  NAMESPACE openpnp-capture::
  FILE openpnp-captureConfig.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/openpnp-capture
)
