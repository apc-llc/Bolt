############################################################################                                                                                     
#   © 2012,2014 Advanced Micro Devices, Inc. All rights reserved.                                     
#                                                                                    
#   Licensed under the Apache License, Version 2.0 (the "License");   
#   you may not use this file except in compliance with the License.                 
#   You may obtain a copy of the License at                                          
#                                                                                    
#       http://www.apache.org/licenses/LICENSE-2.0                      
#                                                                                    
#   Unless required by applicable law or agreed to in writing, software              
#   distributed under the License is distributed on an "AS IS" BASIS,              
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.         
#   See the License for the specific language governing permissions and              
#   limitations under the License.                                                   

############################################################################                                                                                     

cmake_minimum_required( VERSION 2.8.12 )

# uncomment these to debug nmake and borland makefiles
#SET(CMAKE_START_TEMP_FILE "")
#SET(CMAKE_END_TEMP_FILE "")
#SET(CMAKE_VERBOSE_MAKEFILE 1)

# This becomes the name of the solution file
project( Bolt )

# Define a version for the code
if( NOT DEFINED Bolt_VERSION_MAJOR )
   set( Bolt_VERSION_MAJOR 1 )
endif( )

if( NOT DEFINED Bolt_VERSION_MINOR )
   set( Bolt_VERSION_MINOR 3 )
endif( )

if( NOT DEFINED Bolt_VERSION_PATCH )
   set( Bolt_VERSION_PATCH 0 )
endif( )

set( Bolt_VERSION "${Bolt_VERSION_MAJOR}.${Bolt_VERSION_MINOR}.${Bolt_VERSION_PATCH}")

message( STATUS "Bolt_VERSION_MAJOR=${Bolt_VERSION_MAJOR}" )
message( STATUS "Bolt_VERSION_MINOR=${Bolt_VERSION_MINOR}" )
message( STATUS "Bolt_VERSION_PATCH=${Bolt_VERSION_PATCH}" )

list( APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR} )
set( BOLT_LIBRARY_DIR ${PROJECT_SOURCE_DIR}/bolt )

# The binary directory is in the list of includes because files may be configured at build time, such as version.h
set( BOLT_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/include )

# Set the test folder directory. This is common to cl and amp. 
set( BOLT_TEST_DIR ${PROJECT_SOURCE_DIR}/test )

# On windows, it's convenient to change the default install prefix such that it does NOT point to 'program files'
# Need to check out CMAKE_RUNTIME_OUTPUT_DIRECTORY variable, and see if that eliminates the need to modify install path
if( CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT )
    set( CMAKE_INSTALL_PREFIX "${PROJECT_BINARY_DIR}/package" CACHE PATH "Install path prefix, prepended onto install directories" FORCE )
endif( )

# Set the default of CMAKE_BUILD_TYPE to be release, unless user specifies with -D.  MSVC_IDE does not use CMAKE_BUILD_TYPE
#if( NOT MSVC_IDE AND NOT CMAKE_BUILD_TYPE )
#  set( CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE )
#endif()

set_property( GLOBAL PROPERTY USE_FOLDERS TRUE )

set( CMAKE_BUILD_TYPE ${BOLT_BUILD_TYPE} CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE )   
message( STATUS "Build Type R/D:" ${CMAKE_BUILD_TYPE} )

# Various options below on configuring the build, and how to generate the solution files

if ( MSVC )
    set(Bolt_ampDefault ON)
else ()
    set(Bolt_ampDefault OFF)
endif() 

option( BUILD_ampBolt "Create a solution that compiles Bolt for AMP"  ${Bolt_ampDefault})
option( BUILD_clBolt "Create a solution that compiles Bolt for OpenCL" ON )
option( BUILD_StripSymbols "When making debug builds, remove symbols and program database files" OFF )
 
if( IS_DIRECTORY "${PROJECT_SOURCE_DIR}/test" )
    option( BUILD_tests "Add projects for testing Bolt" ON )
endif( )

if( IS_DIRECTORY "${PROJECT_SOURCE_DIR}/bench" )
    option( BUILD_benchmarks "Add projects for benchmarking Bolt" OFF )
endif( )

if( IS_DIRECTORY "${PROJECT_SOURCE_DIR}/doxy" )
    option( BUILD_documentation "Add project for generating Bolt documentation" OFF )
endif( )

# Building the examples from the main Bolt project does not work yet
#if( IS_DIRECTORY "${PROJECT_SOURCE_DIR}/examples" )
#    option( BUILD_examples "Generate example projects to demonstrate Bolt usage" OFF )
#endif( )
set( BUILD_examples OFF )

# Currently, linux has a problem outputing both narrow and wide characters,
# which happens in our client because openCL only supports narrow characters
if( WIN32 )
    option( BUILD_UNICODE "Create a solution that compiles Bolt with Unicode Support" ON )
    if( BUILD_UNICODE )
        message( STATUS "UNICODE build" )
    endif( )
else()
    set( BUILD_UNICODE OFF )
    message( STATUS "UNICODE disabled on linux" )
endif()

# FFLAGS depend on the compiler, grab the compiler name from the path
get_filename_component( C_COMPILER_NAME ${CMAKE_C_COMPILER} NAME_WE )
# message( "C_COMPILER_NAME: " ${C_COMPILER_NAME} )
# message( "CMAKE_C_COMPILER: " ${CMAKE_C_COMPILER} )

# Set common compile and link options
if( C_COMPILER_NAME STREQUAL "cl" )
    # Following options for nMake
    message( STATUS "Detected MSVS Ver: " ${MSVC_VERSION} )
    
    # CMake uses huge stack frames for windows, for some reason.  We remove.
    string( REGEX REPLACE "/STACK:[0-9]+" "" CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}" )
    string( REGEX REPLACE "/STACK:[0-9]+" "" CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS}" )
    string( REGEX REPLACE "/STACK:[0-9]+" "" CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS}" ) 
    
    if( BUILD_StripSymbols )
        string( REGEX REPLACE "/Zi" "" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}" )
    endif()
elseif( ( C_COMPILER_NAME STREQUAL "gcc" ) OR ( CMAKE_COMPILER_IS_GNUCXX GREATER  0 ))
    message( STATUS "Detected GNU fortran compiler." )
    set( CMAKE_CXX_FLAGS "-std=c++0x ${CMAKE_CXX_FLAGS}" )
    
    set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}" )
    set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS}" )
    execute_process(COMMAND lsb_release -irc 
        OUTPUT_VARIABLE suse-ubuntu)
        STRING(FIND $(suse_ubuntu) "Ubuntu" pos)
    SET(com -1)
    if($(pos) EQUAL com)
        set( CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS}" )
        set( CMAKE_EXE_LINKER_FLAGS "-lpthread ${CMAKE_EXE_LINKER_FLAGS}" )
    else()
        set( CMAKE_EXE_LINKER_FLAGS " ${CMAKE_EXE_LINKER_FLAGS}" )
        set( CMAKE_SHARED_LINKER_FLAGS "-lpthread ${CMAKE_SHARED_LINKER_FLAGS}" )
    endif()
else( )
    message( AUTHOR_WARNING "Compiler not recognized.  Using default flags." )
endif( )

# If UNICODE is defined, pass extra definitions into 
if( BUILD_UNICODE )
    add_definitions( "/DUNICODE /D_UNICODE" )
endif( )

# Print out compiler flags for viewing/debug
message( STATUS "CMAKE_CXX_COMPILER flags: " ${CMAKE_CXX_FLAGS} )
message( STATUS "CMAKE_CXX_COMPILER debug flags: " ${CMAKE_CXX_FLAGS_DEBUG} )
message( STATUS "CMAKE_CXX_COMPILER release flags: " ${CMAKE_CXX_FLAGS_RELEASE} )
message( STATUS "CMAKE_CXX_COMPILER relwithdebinfo flags: " ${CMAKE_CXX_FLAGS_RELWITHDEBINFO} )
message( STATUS "CMAKE_EXE_LINKER link flags: " ${CMAKE_EXE_LINKER_FLAGS} )

set( CPACK_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${Bolt_VERSION}-${CMAKE_HOST_SYSTEM_NAME}" )
set( INCLUDE_DIR include )
set( BIN_DIR bin )
set( LIB_DIR lib )

# Build AMP library (only header files for now)
if( BUILD_ampBolt )
    if( MSVC_VERSION VERSION_LESS 1700 )
        message( FATAL_ERROR "Visual Studio 11 or higher is required because Bolt uses AMP C++ features" )
    endif( )

    if( IS_DIRECTORY "${BOLT_LIBRARY_DIR}/amp" AND WIN32 )
        add_subdirectory( ${BOLT_LIBRARY_DIR}/amp )
    endif( )
endif( )
    
# Build OpenCL library
if( BUILD_clBolt )
    # This will define OPENCL_FOUND
    set(OpenCL_INCLUDE_DIR /opt/rocm/opencl/include)
    set(OpenCL_LIBRARY /opt/rocm/opencl/lib/libOpenCL.so) 
    find_package( OpenCL REQUIRED )
    set(OPENCL_INCLUDE_DIRS ${OpenCL_INCLUDE_DIR})

    if( OPENCL_FOUND AND IS_DIRECTORY "${BOLT_LIBRARY_DIR}/cl" AND IS_DIRECTORY "${PROJECT_SOURCE_DIR}/tools" )
        add_subdirectory( tools )
        add_subdirectory( ${BOLT_LIBRARY_DIR}/cl )
    endif( )
endif( )

set(TBB_BUILD_TESTS OFF)
add_subdirectory(ThirdParty/tbb)

# After libraries are built, built tests
if( BUILD_tests )
    add_subdirectory( test )
endif( )    

# After libraries are built, built benchmarks
if( BUILD_benchmarks )
    add_subdirectory( bench )
endif( )

# After libraries are built, built documentation
if( BUILD_documentation )
    add_subdirectory( doxy )
endif( )

# After libraries are built, built examples
if( BUILD_examples )
    add_subdirectory( examples EXCLUDE_FROM_ALL )
endif( )

# configure a header file to pass the CMake version settings to the source, and package the header files in the output archive
configure_file( "${PROJECT_SOURCE_DIR}/include/bolt/BoltVersion.h.in" "${PROJECT_BINARY_DIR}/include/bolt/BoltVersion.h" @ONLY )
install( FILES 
            "${PROJECT_BINARY_DIR}/include/bolt/BoltVersion.h" 
        DESTINATION 
            ${INCLUDE_DIR}/bolt )

configure_file( "${PROJECT_SOURCE_DIR}/examples/CMakeLists.txt.in" "${PROJECT_BINARY_DIR}/examples/CMakeLists.txt" @ONLY )

# Right now, always install the examples folder
install( DIRECTORY 
            examples
        DESTINATION
            .
        PATTERN "Hessian" EXCLUDE
        PATTERN "*.in" EXCLUDE )
        
install( FILES 
            "${PROJECT_BINARY_DIR}/examples/CMakeLists.txt" 
        DESTINATION 
            examples )

# Copy over the documentation related files
install( FILES 
            doxy/README.html
            LICENSE.txt
        DESTINATION
            . )

# The following code is setting variables to control the behavior of CPack to generate our 
if( WIN32 )
    set( CPACK_SOURCE_GENERATOR "ZIP" )
    set( CPACK_GENERATOR "ZIP" )
else( )
    set( CPACK_SOURCE_GENERATOR "TGZ" )
    set( CPACK_GENERATOR "TGZ" )
endif( )

set( CPACK_SOURCE_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${Bolt_VERSION}-${CMAKE_HOST_SYSTEM_NAME}-Source")
#set( CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/ReadMeBuild.html")
set( CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.txt")
set( CPACK_PACKAGE_VERSION_MAJOR ${Bolt_VERSION_MAJOR} )
set( CPACK_PACKAGE_VERSION_MINOR ${Bolt_VERSION_MINOR} )
set( CPACK_PACKAGE_VERSION_PATCH ${Bolt_VERSION_PATCH} )
set( CPACK_PACKAGE_DESCRIPTION_SUMMARY "Bolt library SDK package")
set( CPACK_PACKAGE_VENDOR "AMD")
set( CPACK_SOURCE_IGNORE_FILES "/\\\\.hg/;/\\\\.svn/;" )

# Define all variables that influence CPack before including CPack, such as install targets
include( CPack )
