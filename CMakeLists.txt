cmake_minimum_required(VERSION 3.16)

project(bpf VERSION 0.6.0 LANGUAGES C)

option(BUILD_SHARED_LIBS "The option allows you to switch between shared and static library" OFF)

if (NOT CMAKE_BUILD_TYPE)
    set (CMAKE_BUILD_TYPE RelWithDebInfo)
endif (NOT CMAKE_BUILD_TYPE)

set (CMAKE_POSITION_INDEPENDENT_CODE ON)
set(is-clang $<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>>)
set(is-gnu $<CXX_COMPILER_ID:GNU>)
set(compiler-id-clang-or-gnu $<OR:${is-clang},${is-gnu}>)

set(BPF_SRC_FILES
    src/bpf.c
    src/btf.c
    src/libbpf.c
    src/libbpf_errno.c
    src/netlink.c
    src/nlattr.c
    src/str_error.c
    src/libbpf_probes.c
    src/bpf_prog_linfo.c
    src/btf_dump.c
    src/hashmap.c
    src/ringbuf.c
    src/strset.c
    src/linker.c
    src/gen_loader.c
    src/relo_core.c
    src/xsk.c
)
add_library(bpf ${BPF_SRC_FILES})
add_library(bpf::bpf ALIAS bpf)

target_include_directories(
    bpf
    PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/usr/include>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
)

target_include_directories(bpf PRIVATE
    ${CMAKE_CURRENT_LIST_DIR}/include
    ${CMAKE_CURRENT_LIST_DIR}/include/uapi
    ${CMAKE_CURRENT_LIST_DIR}/src
)

set_target_properties(bpf PROPERTIES OUTPUT_NAME ${PROJECT_NAME})
set_target_properties(bpf PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})

target_compile_options(
    bpf
    PRIVATE
    -c -pipe -fomit-frame-pointer -Werror -Wall -D_LARGEFILE64_SOURCE -D_FILE_OFFSET_BITS=64
)
include(GNUInstallDirs)

set(LIBBPF_INCLUDE_DIR ${CMAKE_BINARY_DIR}/usr/include/bpf)
set(LINUX_INCLUDE_DIR ${CMAKE_BINARY_DIR}/usr/include/linux)

add_custom_command(
    TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/bpf.h ${LIBBPF_INCLUDE_DIR}/bpf.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/libbpf.h ${LIBBPF_INCLUDE_DIR}/libbpf.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/btf.h ${LIBBPF_INCLUDE_DIR}/btf.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/libbpf_common.h ${LIBBPF_INCLUDE_DIR}/libbpf_common.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/libbpf_legacy.h ${LIBBPF_INCLUDE_DIR}/libbpf_legacy.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/bpf_helpers.h ${LIBBPF_INCLUDE_DIR}/bpf_helpers.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/bpf_helper_defs.h ${LIBBPF_INCLUDE_DIR}/bpf_helper_defs.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/bpf_tracing.h ${LIBBPF_INCLUDE_DIR}/bpf_tracing.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/bpf_endian.h ${LIBBPF_INCLUDE_DIR}/bpf_endian.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/bpf_core_read.h ${LIBBPF_INCLUDE_DIR}/bpf_core_read.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/skel_internal.h ${LIBBPF_INCLUDE_DIR}/skel_internal.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/libbpf_version.h ${LIBBPF_INCLUDE_DIR}/libbpf_version.h
    COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/src/xsk.h ${LIBBPF_INCLUDE_DIR}/xsk.h
)
#COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/include/linux/err.h ${LINUX_INCLUDE_DIR}/err.h
#COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/include/linux/list.h ${LINUX_INCLUDE_DIR}/list.h
#COMMAND install -m 0644 -D ${CMAKE_CURRENT_LIST_DIR}/include/linux/types.h ${LINUX_INCLUDE_DIR}/types.h

message(STATUS "Generating install")
install(DIRECTORY include/ DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
install(
    TARGETS bpf
    EXPORT bpf
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

set(project_config_in "${CMAKE_CURRENT_LIST_DIR}/cmake/bpfConfig.cmake.in")
set(project_config_out "${CMAKE_CURRENT_BINARY_DIR}/bpfConfig.cmake")
set(config_targets_file "bpfConfigTargets.cmake")
set(version_config_file "${CMAKE_CURRENT_BINARY_DIR}/bpfConfigVersion.cmake")
set(export_dest_dir "${CMAKE_INSTALL_LIBDIR}/cmake/bpf")

# ---------------------------------------------------------------------------------------
# Install CMake config files
# ---------------------------------------------------------------------------------------
install(EXPORT bpf DESTINATION ${export_dest_dir} NAMESPACE bpf:: FILE ${config_targets_file})

include(CMakePackageConfigHelpers)
configure_file("${project_config_in}" "${project_config_out}" @ONLY)

write_basic_package_version_file("${version_config_file}" COMPATIBILITY SameMajorVersion)
install(FILES "${project_config_out}" "${version_config_file}" DESTINATION "${export_dest_dir}")
