#
# project: fips-uSockets
# Miniscule cross-platform eventing, networking & crypto for async applications
#
cmake_minimum_required(VERSION 3.21)

project(fips-uSockets)

get_filename_component(FIPS_ROOT_DIR "../fips" ABSOLUTE)
include("${FIPS_ROOT_DIR}/cmake/fips.cmake")

fips_setup()

# Options
option(USOCKETS_NO_SSL "Build without SSL support" ON)
option(USOCKETS_USE_OPENSSL "Use OpenSSL for SSL" OFF)
option(USOCKETS_USE_LIBUV "Use libuv event loop" OFF)

fips_begin_lib(uSockets)

    # Core source files
    fips_dir(src)
    fips_files(
        bsd.c
        context.c
        loop.c
        socket.c
        udp.c
        quic.c
        quic.h
        libusockets.h
    )

    # Eventing layer - platform specific event loop implementations
    fips_dir(src/eventing)
    if (USOCKETS_USE_LIBUV)
        fips_files(libuv.c)
    elseif (FIPS_MACOS OR FIPS_IOS)
        fips_files(epoll_kqueue.c)
    elseif (FIPS_LINUX)
        fips_files(epoll_kqueue.c)
    elseif (FIPS_WINDOWS)
        fips_files(epoll_kqueue.c)
    else()
        fips_files(epoll_kqueue.c)
    endif()

    # Crypto layer
    fips_dir(src/crypto)
    fips_files(
        openssl.c
        sni_tree.cpp
    )

    # Internal headers
    fips_dir(src/internal)
    fips_files(
        internal.h
        networking/bsd.h
        eventing/libuv.h
        eventing/epoll_kqueue.h
        eventing/gcd.h
    )

fips_end_lib()

# Include directories
target_include_directories(uSockets PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/src)

# Compiler definitions
if (USOCKETS_NO_SSL)
    target_compile_definitions(uSockets PUBLIC LIBUS_NO_SSL)
elseif (USOCKETS_USE_OPENSSL)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_OPENSSL)
    find_package(OpenSSL REQUIRED)
    target_link_libraries(uSockets OpenSSL::SSL OpenSSL::Crypto)
endif()

if (USOCKETS_USE_LIBUV)
    target_compile_definitions(uSockets PUBLIC LIBUS_USE_LIBUV)
    find_package(libuv CONFIG REQUIRED)
    target_link_libraries(uSockets $<IF:$<TARGET_EXISTS:libuv::uv_a>,libuv::uv_a,libuv::uv>)
endif()

# Platform-specific settings
if (FIPS_WINDOWS)
    target_compile_definitions(uSockets PRIVATE WIN32_LEAN_AND_MEAN _CRT_SECURE_NO_WARNINGS)
endif()

# C11 standard for C files
set_target_properties(uSockets PROPERTIES C_STANDARD 11)

# Disable warnings for third-party code
if (FIPS_MSVC)
    target_compile_options(uSockets PRIVATE /wd4100 /wd4127 /wd4244 /wd4267 /wd4702)
elseif (FIPS_CLANG OR FIPS_GCC)
    target_compile_options(uSockets PRIVATE -Wno-unused-parameter -Wno-sign-compare)
endif()

# Link threading on Unix
if (FIPS_LINUX OR FIPS_OSX OR FIPS_IOS)
    find_package(Threads REQUIRED)
    target_link_libraries(uSockets ${CMAKE_THREAD_LIBS_INIT})
endif()
