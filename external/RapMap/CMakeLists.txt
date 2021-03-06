cmake_minimum_required (VERSION 2.8)

enable_testing()

project (RapMap)

set(CPACK_PACKAGE_VERSION "0.5.0")
SET(CPACK_PACKAGE_VERSION_MAJOR "0")
set(CPACK_PACKAGE_VERSION_MINOR "5")
set(CPACK_PACKAGE_VERSION_PATCH "0")
set(PROJECT_VERSION ${CPACK_PACKAGE_VERSION})
set(CPACK_GENERATOR "TGZ")
set(CPACK_SOURCE_GENERATOR "TGZ")
set(CPACK_PACKAGE_VENDOR "Stony Brook University")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "RapMap - Wicked-fast qasi-mapping")
set(CPACK_PACKAGE_NAME
  "${CMAKE_PROJECT_NAME}-${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
set(CPACK_SOURCE_PACKAGE_FILE_NAME
  "${CMAKE_PROJECT_NAME}-${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}-Source")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")
#include(FindSSE)
#FindSSE ()
#if(SSE4_2_FOUND)
#    message("Enabling popcount")
#    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse4.2 -DEMPHF_USE_POPCOUNT")
#endif(SSE4_2_FOUND)

if (APPLE)
set (WARNING_IGNORE_FLAGS "-Wno-deprecated-register -Wno-unknon-pragmas -Wreturn-type -Werror=return-type")
else()
set (WARNING_IGNORE_FLAGS "-Wno-unknown-pragmas -Wreturn-type -Werror=return-type")
endif()

## Prefer static to dynamic libraries
SET(CMAKE_FIND_LIBRARY_SUFFIXES .a ${CMAKE_FIND_LIBRARY_SUFFIXES})

## Set the standard required compile flags
if (NO_NATIVE_ARCH)
  message (STATUS "DISABLING NATIVE ARCH.")
else()
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
endif()


if (QUIET_BUILD)
  set(WALL "")
else()
  set(WALL "-Wall")
endif()

set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread -funroll-loops -fPIC -fomit-frame-pointer -O4 -DHAVE_ANSI_TERM ${WALL} -std=c++11")

set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads REQUIRED)

##
# OSX is strange (some might say, stupid in this regard).  Deal with it's quirkines here.
##
if (APPLE)
    # To allow ourselves to build a dynamic library, we have to tell the compiler
    # that, yes, the symbols will be around at runtime.
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -undefined dynamic_lookup")
    set (LIBSALMON_LINKER_FLAGS "-all_load")
    # In order to "think different", we also have to use non-standard suffixes
    # for our shared libraries
    set(SHARED_LIB_EXTENSION "dylib")
else()
    # We're in sane linux world
   set (SHARED_LIB_EXTENSION "so")
   set (LIBSALMON_LINKER_FLAGS "")
endif()

set( BOOST_EXTRA_FLAGS "--layout=tagged" )
## this get's set differently below if we
## are on clang & apple
set (NON_APPLECLANG_LIBS gomp rt)
set (PTHREAD_LIB)

##
# Compiler-specific C++11 activation.
# http://stackoverflow.com/questions/10984442/how-to-detect-c11-support-of-a-compiler-with-cmake
##
##
# First take care of what to do if we have gcc
##
set (JELLYFISH_CXX_FLAGS "-fPIC") 
if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU")
    execute_process(
        COMMAND ${CMAKE_CXX_COMPILER} -dumpversion OUTPUT_VARIABLE GCC_VERSION)
    # If we're on OSX
    if (APPLE AND NOT (GCC_VERSION VERSION_GREATER 4.8.2 OR GCC_VERSION VERSION_EQUAL 4.8.2))
        message(FATAL_ERROR "When building under OSX, ${PROJECT_NAME} requires "
                            "either clang or g++ >= 4.8.2")
    elseif (NOT (GCC_VERSION VERSION_GREATER 4.7 OR GCC_VERSION VERSION_EQUAL 4.7))
        message(FATAL_ERROR "${PROJECT_NAME} requires g++ 4.7 or greater.")
    endif ()

    set (GCC TRUE)

    # Put complete static linking on hold for the time-being
    # If we're not on OSX, make an attempt to compile everything statically
    #if (NOT APPLE)
    #set (CMAKE_CXX_FLAGS "-static ${CMAKE_CXX_FLAGS}")
    #set (CMAKE_EXE_LINK_FLAGS "-static")
    set (PTHREAD_LIB "pthread")
    #endif()

    # If we're on Linux (i.e. not OSX) and we're using
    # gcc, then set the -static-libstdc++ flag
    if (NOT APPLE)
        set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libstdc++")
    endif()

    set (WARNING_IGNORE_FLAGS "${WARNING_IGNORE_FLAGS} -Wno-unused-local-typedefs")
# Tentatively, we support clang now
elseif ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    set(CLANG TRUE)
    # If we have libc++, then try and use it
    include(CheckCXXCompilerFlag)
    check_cxx_compiler_flag(-stdlib=libc++ HAVE_LIBCPP)
    if (HAVE_LIBCPP)
        message ("It appears that you're compiling with clang and that libc++ is available, so I'll use that")
        set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
	set (JELLYFISH_CXX_FLAGS "${JELLYFISH_CXX_FLAGS} -stdlib=libc++")
    # Otherwise, use libstdc++ (and make it static)
    else()
        set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libstdc++")
    endif()

    if (APPLE)
        set (NON_APPLECLANG_LIBS "")
    else()
        set (PTHREAD_LIB "pthread")
    endif()
else ()
    message(FATAL_ERROR "Your C++ compiler does not support C++11.")
endif ()

include(ExternalProject)

##
#  Update the CXX flags according to the system and compiler
##
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${WARNING_IGNORE_FLAGS}")

if (CMAKE_BUILD_TYPE MATCHES Debug)
    message ("Making Debug build")
    set (CMAKE_CXX_FLAGS_DEBUG "-g ${CMAKE_CXX_FLAGS}")
elseif (CMAKE_BUILD_TYPE MATCHES Release)
    message ("Making Release build")
    set (CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS}")
else ()
    message ("Making Default build type")
endif ()

##
# Record this top-level path
##
set (GAT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})

# Have CMake tell us what it's doing
set (CMAKE_VERBOSE_MAKEFILE true)

find_package (ZLIB REQUIRED)
if (NOT ZLIB_FOUND)
	message (FATAL_ERROR "zlib must be installed before configuration & building can proceed")
endif()


message("Build system will build libdivsufsort")
message("==================================================================")
include(ExternalProject)
ExternalProject_Add(libdivsufsort
    DOWNLOAD_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external
    URL ${CMAKE_CURRENT_SOURCE_DIR}/external/libdivsufsort.tar.gz
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/libdivsufsort-master
    INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install
    UPDATE_COMMAND sh -c "mkdir -p <SOURCE_DIR>/build"
    BINARY_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/libdivsufsort-master/build
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR> -DBUILD_DIVSUFSORT64=TRUE -DUSE_OPENMP=TRUE -DBUILD_SHARED_LIBS=FALSE
)
set(SUFFARRAY_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/external/install/include)

if (NOT CEREAL_ROOT)
	set(CEREAL_ROOT ${GAT_SOURCE_DIR}/external/install)
endif()

find_package (Cereal 1.2.2)
if (NOT CEREAL_FOUND)

	message("Build system will fetch and build the Cereal serialization library")
	message("==================================================================")
	include(ExternalProject)
	ExternalProject_Add(libcereal
	    DOWNLOAD_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external
	    DOWNLOAD_COMMAND curl -k -L https://github.com/USCiLab/cereal/archive/v1.2.2.tar.gz -o cereal-v1.2.2.tar.gz &&
		tar -xzvf cereal-v1.2.2.tar.gz
	    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/cereal-1.2.2
	    INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install
	    UPDATE_COMMAND sh -c "mkdir -p <SOURCE_DIR>/build"
	    BINARY_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/cereal-1.2.2/build
	    CONFIGURE_COMMAND ""
	    BUILD_COMMAND ""
	    INSTALL_COMMAND sh -c "mkdir -p <INSTALL_DIR>/include && cp -r <SOURCE_DIR>/include/cereal <INSTALL_DIR>/include"
	)

	set(CEREAL_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/external/install/include)

endif()


if (NOT JELLYFISH_ROOT)
	set(JELLYFISH_ROOT ${GAT_SOURCE_DIR}/external/install)
endif()

find_package(Jellyfish 2.2.6)

if (NOT JELLYFISH_FOUND)
message("Build system will fetch and build Jellyfish")
message("==================================================================")
ExternalProject_Add(libjellyfish
    DOWNLOAD_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external
    DOWNLOAD_COMMAND curl -k -L https://github.com/gmarcais/Jellyfish/releases/download/v2.2.6/jellyfish-2.2.6.tar.gz -o jellyfish-2.2.6.tgz &&
    	rm -fr jellyfish-2.2.6 &&
     	tar -xzvf jellyfish-2.2.6.tgz
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/jellyfish-2.2.6
    INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install
    CONFIGURE_COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/external/jellyfish-2.2.6/configure --enable-shared=no --prefix=<INSTALL_DIR> CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER} CXXFLAGS=${JELLYFISH_CXX_FLAGS}
    BUILD_COMMAND make CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER} CXXFLAGS=${JELLYFISH_CXX_FLAGS}
    BUILD_IN_SOURCE 1
    INSTALL_COMMAND make install
)
endif()

set (FAST_MALLOC_LIB "")
set (HAVE_FAST_MALLOC FALSE)

# See if we have Jemalloc
find_package(Jemalloc)
if (Jemalloc_FOUND)
    message("Found Jemalloc library --- using this memory allocator")
    set (FAST_MALLOC_LIB ${JEMALLOC_LIBRARIES})
    set (HAVE_FAST_MALLOC TRUE)
endif()

if (NOT HAVE_FAST_MALLOC)
    # See if we have Tcmalloc
    find_package(Tcmalloc)
    if (Tcmalloc_FOUND)
        message("Fount TCMalloc library --- using this memory allocator")
        set (TCMALLOC_LIB ${Tcmalloc_LIBRARIES})
        set (FAST_MALLOC_LIB ${TCMALLOC_LIB})
        set (HAVE_FAST_MALLOC TRUE)
    endif()
endif()

if (NOT HAVE_FAST_MALLOC)
    message("Build system will fetch and use JEMalloc")
    message("==================================================================")
    ExternalProject_Add(libjemalloc
        DOWNLOAD_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external
        DOWNLOAD_COMMAND curl -k -L https://github.com/COMBINE-lab/jemalloc/archive/4.5.0.tar.gz -o jemalloc-4.5.0.tar.gz &&
                         tar -xzf jemalloc-4.5.0.tar.gz
        SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/jemalloc-4.5.0
        BUILD_IN_SOURCE TRUE
        INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/install
        CONFIGURE_COMMAND sh -c "${JEMALLOC_FLAGS} ./autogen.sh --disable-debug --prefix=<INSTALL_DIR>"
        INSTALL_COMMAND cp -r lib <INSTALL_DIR>/ && cp -r include <INSTALL_DIR>/
        )

    set (FAST_MALLOC_LIB ${CMAKE_CURRENT_SOURCE_DIR}/external/install/lib/libjemalloc.a)
    set (HAVE_FAST_MALLOC TRUE)
endif ()

###
#
# Done building external dependencies.
#
###

set (CPACK_SOURCE_IGNORE_FILES
".git/")

message("CPACK_SOURCE_IGNORE_FILES = ${CPACK_SOURCE_IGNORE_FILES}")

# Recurse into Salmon source directory
add_subdirectory ( src )

# build a CPack driven installer package
include (CPack)

set(ARCHIVE_NAME ${CMAKE_PROJECT_NAME}-${PROJECT_VERSION})
add_custom_target(dist
        COMMAND git archive --prefix=${ARCHIVE_NAME}/ HEAD
        | gzip > ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar.gz
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

