cmake_minimum_required(VERSION 3.10.2)

add_compile_options(
        -std=c11
        -Weverything
        -Werror)

# import xdl by local-project
string(TOLOWER ${CMAKE_BUILD_TYPE} BUILD_TYPE_DIRNAME)
set(XDL_BASE ${CMAKE_CURRENT_SOURCE_DIR}/../../../../xdl)
add_library(xdl::xdl SHARED IMPORTED)
set_target_properties(xdl::xdl PROPERTIES
        IMPORTED_LOCATION ${XDL_BASE}/build/intermediates/stripped_native_libs/${BUILD_TYPE_DIRNAME}/out/lib/${ANDROID_ABI}/libxdl.so
        INTERFACE_INCLUDE_DIRECTORIES ${XDL_BASE}/src/main/cpp/include
        )

# import xdl by maven
#find_package(xdl REQUIRED CONFIG)

add_library(sample SHARED
        sample.c)

target_link_libraries(sample
        xdl::xdl
        log)
