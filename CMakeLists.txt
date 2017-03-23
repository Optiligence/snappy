cmake_policy(SET CMP0048 NEW)
project(Snappy VERSION 1.1.4 LANGUAGES C CXX)
cmake_minimum_required(VERSION 3.0)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

option(BUILD_SHARED_LIBS "Enable to build the shared version of the library or disable to build the static version" ON)

include(CheckIncludeFiles)
include(CheckLibraryExists)
include(CheckCXXSourceCompiles)
include(TestBigEndian)
include(GenerateExportHeader)
include(CMakePackageConfigHelpers)

test_big_endian(WORDS_BIG_ENDIAN)
if (WORDS_BIG_ENDIAN)
    message(STATUS "Building on big endian system")
    add_definitions(-DWORDS_BIGENDIAN=1)
endif (WORDS_BIG_ENDIAN)

check_include_files("windows.h" HAVE_WINDOWS_H)
check_include_files("byteswap.h" HAVE_BYTESWAP_H)
check_include_files("dlfcn.h" HAVE_DLFCN_H)
check_include_files("inttypes.h" HAVE_INTTYPES_H)
check_include_files("memory.h" HAVE_MEMORY_H)
check_include_files("stddef.h" HAVE_STDDEF_H)
check_include_files("stdint.h" HAVE_STDINT_H)
check_include_files("stdlib.h" HAVE_STDLIB_H)
check_include_files("strings.h" HAVE_STRINGS_H)
check_include_files("string.h" HAVE_STRING_H)
check_include_files("sys/byteswap.h" HAVE_SYS_BYTESWAP_H)
check_include_files("sys/endian.h" HAVE_SYS_ENDIAN_H)
check_include_files("stdint.h" HAVE_STDINT_H)
check_include_files("sys/mman.h" HAVE_SYS_MMAN_H)
check_include_files("sys/resource.h" HAVE_SYS_RESOURCE_H)
check_include_files("sys/stat.h" HAVE_SYS_STAT_H)
check_include_files("sys/time.h" HAVE_SYS_TIME_H)
check_include_files("sys/types.h" HAVE_SYS_TYPES_H)
check_include_files("unistd.h" HAVE_UNISTD_H)
check_include_files("sys/uio.h" HAVE_SYS_UIO_H)

if (NOT HAVE_SYS_UIO_H)
    set(HAVE_SYS_UIO_H 0)
endif (NOT HAVE_SYS_UIO_H)

if (NOT HAVE_STDINT_H)
    set(HAVE_STDINT_H 0)
endif (NOT HAVE_STDINT_H)

if (NOT HAVE_STDDEF_H)
    set(HAVE_STDDEF_H 0)
endif (NOT HAVE_STDDEF_H)

check_library_exists(z zlibVersion "" HAVE_LIBZ)
check_library_exists(lzo2 lzo1x_1_15_compress "" HAVE_LIBLZO2)
check_library_exists(lzf lzf_compress "" HAVE_LIBLZF)
check_library_exists(fastlz fastlz_compress "" HAVE_LIBFASTLZ)
check_library_exists(quicklz qlz_compress "" HAVE_LIBQUICKLZ)

check_cxx_source_compiles("int main(void) { return __builtin_expect(0, 1) ? 1 : 0; }"
        HAVE_BUILTIN_EXPECT)

check_cxx_source_compiles("int main(void) { return __builtin_ctzll(0) ? 1 : 0; }"
        HAVE_BUILTIN_CTZ)


configure_file(${Snappy_SOURCE_DIR}/cmake/config.h.in config.h)
configure_file(${Snappy_SOURCE_DIR}/cmake/snappy-stubs-public.h.in snappy-stubs-public.h)

if (WIN32)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif (WIN32)

# Define the main library.
add_library(Snappy
        snappy-c.cc
        snappy-c.h
        snappy-sinksource.cc
        snappy-sinksource.h
        snappy-stubs-internal.cc
        snappy-stubs-public.h
        snappy.cc
        snappy.h)

target_compile_definitions(Snappy PRIVATE -DHAVE_CONFIG_H)

set_target_properties(Snappy PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON OUTPUT_NAME snappy)

target_include_directories(Snappy INTERFACE $<INSTALL_INTERFACE:include>)
install(FILES snappy.h
        snappy-c.h
        snappy-sinksource.h
        ${Snappy_BINARY_DIR}/snappy-stubs-public.h
        DESTINATION include)

if (BUILD_SHARED_LIBS)
    set(ExportSet SnappySharedTargets)
else()
    set(ExportSet SnappyStaticTargets)
endif()
install(TARGETS Snappy
        EXPORT ${ExportSet}
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)
install(EXPORT ${ExportSet} NAMESPACE Snappy:: DESTINATION lib/cmake/Snappy)

set(INCLUDE_INSTALL_DIR include)
set(LIBRARY_INSTALL_DIR lib)
set(BINARY_INSTALL_DIR bin)

configure_package_config_file(cmake/SnappyConfig.cmake.in
        ${Snappy_BINARY_DIR}/SnappyConfig.cmake
        INSTALL_DESTINATION lib/cmake/Snappy
        PATH_VARS INCLUDE_INSTALL_DIR LIBRARY_INSTALL_DIR BINARY_INSTALL_DIR
        )

write_basic_package_version_file(${Snappy_BINARY_DIR}/SnappyConfigVersion.cmake
        COMPATIBILITY SameMajorVersion)
install(FILES ${Snappy_BINARY_DIR}/SnappyConfig.cmake
        ${Snappy_BINARY_DIR}/SnappyConfigVersion.cmake
        DESTINATION lib/cmake/Snappy)

enable_testing()

if (HAVE_LIBZ)
    list(APPEND COMPRESSION_LIBS z)
endif (HAVE_LIBZ)

if (HAVE_LIBLZO2)
    list(APPEND COMPRESSION_LIBS lzo2)
endif (HAVE_LIBLZO2)

if (HAVE_LIBLZF)
    list(APPEND COMPRESSION_LIBS lzf)
endif (HAVE_LIBLZF)

if (HAVE_LIBFASTLZ)
    list(APPEND COMPRESSION_LIBS fastlz)
endif (HAVE_LIBFASTLZ)

if (HAVE_LIBQUICKLZ)
    list(APPEND COMPRESSION_LIBS quicklz)
endif (HAVE_LIBQUICKLZ)

add_executable(snappy-unittest snappy_unittest.cc snappy-test.cc)
target_compile_definitions(snappy-unittest PRIVATE -DHAVE_CONFIG_H)
target_link_libraries(snappy-unittest Snappy ${COMPRESSION_LIBS})
target_include_directories(snappy-unittest BEFORE PRIVATE ${Snappy_SOURCE_DIR})

add_test(NAME snappy-unittest
         WORKING_DIRECTORY ${Snappy_SOURCE_DIR}
         COMMAND ${Snappy_BINARY_DIR}/snappy-unittest)
