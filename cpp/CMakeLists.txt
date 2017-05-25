cmake_minimum_required( VERSION 3.1 )
project( securemessage )

enable_language(CXX)

set( EXECUTABLE_OUTPUT_PATH bin )
set( LIBRARY_OUTPUT_PATH lib )

set( CMAKE_BUILD_TYPE Release )
set( CMAKE_SKIP_BUILD_RPATH true )

set( CMAKE_CXX_COMPILER g++ )
set( CMAKE_CXX_STANDARD 11 )
set( CMAKE_CXX_FLAGS "-std=gnu++0x -Wall -Wnarrowing -Wextra -pthread" )

set(OPENSSL_ROOT_DIR /usr/local/opt/openssl)
INCLUDE_DIRECTORIES(/usr/local/opt/openssl/include)
LINK_DIRECTORIES(/usr/local/opt/openssl/lib)

set( PROJECT_LINK_LIBS ${PROJECT_LINK_LIBS} pthread crypto protobuf-lite )

# Architecture detection courtesy of BoringSSL
if (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86_64")
    set(ARCH "x86_64")
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "amd64")
    set(ARCH "x86_64")
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "AMD64")
    # cmake reports AMD64 on Windows, but we might be building for 32-bit.
    if (CMAKE_CL_64)
	set(ARCH "x86_64")
    else()
	set(ARCH "x86")
    endif()
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86")
    set(ARCH "x86")
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "i386")
    set(ARCH "x86")
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "i686")
    set(ARCH "x86")
elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "^arm")
    set(ARCH "arm")
else()
    message(FATAL_ERROR "Unknown processor:" ${CMAKE_SYSTEM_PROCESSOR})
endif()

if( ${CMAKE_SYSTEM_NAME} MATCHES "Darwin" )
    set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DOS_UNIX" )
    link_directories( lib/macos ./ )
elseif( ${CMAKE_SYSTEM_NAME} MATCHES "Linux" )
    set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DOS_UNIX -D_POSIX_C_SOURCES=200809L" )
    set( PROJECT_LINK_LIBS ${PROJECT_LINK_LIBS} usb )
    if( ${ARCH} STREQUAL "arm" )
	link_directories( /lib/arm-linux-gnueabihf lib/rpi ./ )
    elseif( ${ARCH} STREQUAL "x86_64" )
        link_directories( lib/linux_64bit ./ )
    else()
	link_directories( lib/linux_32bit ./ )
    endif(${ARCH} STREQUAL "arm" )
endif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")

message(STATUS "Building for ${CMAKE_SYSTEM_NAME}/${ARCH}")

find_package(Openssl REQUIRED)
find_package(Protobuf REQUIRED)
include_directories(${PROTOBUF_INCLUDE_DIRS})
protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ../proto/securemessage.proto)
set_source_files_properties(${PROTO_SRCS} ${PROTO_HDRS} PROPERTIES GENERATED TRUE)

include_directories(include /usr/local/opt/openssl/include ${PROTOBUF_INCLUDE_DIRS} ${CMAKE_CURRENT_BINARY_DIR})

set(LIB_SOURCES src/byte_buffer.cc src/crypto_ops_openssl.cc
    src/raw_secure_message_parser.cc src/secure_message_parser.cc
    src/util.cc src/crypto_ops.cc src/public_key_proto_util.cc
    src/secure_message_builder.cc src/secure_message_wrapper.cc)

add_library( securemessage STATIC ${LIB_SOURCES} ${PROTO_SRCS} )
target_link_libraries( securemessage ${PROJECT_LINK_LIBS} ${PROTOBUF_LIBRARIES} )
