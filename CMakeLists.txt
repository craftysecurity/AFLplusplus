# CMake for compiling afl-fuzz and afl-frida-trace for android arm64

project(AFLppAndroid)
cmake_minimum_required(VERSION 3.13)
add_compile_options(
    -Wno-pointer-sign
    -Wno-pointer-arith
    -Wno-sign-compare
    -Wno-unused-parameter
    -Wno-unused-function
    -Wno-format
    -Wno-user-defined-warnings
    -Wno-macro-redefined
    -Wno-deprecated
)


file(GLOB afl-fuzz-c src/afl-fuzz*.c)
add_executable(
    afl-fuzz
    ${afl-fuzz-c}
    "src/afl-common.c"
    "src/afl-forkserver.c"
    "src/afl-sharedmem.c"
    "src/afl-performance.c"
)
target_include_directories(afl-fuzz AFTER PRIVATE ${CMAKE_SOURCE_DIR}/include)
target_compile_definitions(
    afl-fuzz
    PRIVATE DOC_PATH=\"/usr/local/share/doc/afl\"
    PRIVATE BIN_PATH=\"/usr/local/bin\"
    PRIVATE AFL_PATH=\"/usr/local/lib/afl\"
    PRIVATE _FORTIFY_SOURCE=2
)
set_target_properties(afl-fuzz PROPERTIES POSITION_INDEPENDENT_CODE ON)
set_target_properties(afl-fuzz PROPERTIES COMPILE_FLAGS "-O2")

set(FRIDA_DIR "/home/<username>/AFL-34/FRIDA-MODIFIED/frida/build/subprojects/frida-gum/bindings/gumjs/devkit")

set(API_JS "${CMAKE_SOURCE_DIR}/frida_mode/src/js/api.js")
set(API_C "api.c")
execute_process(
    COMMAND
    bash -c "echo 'unsigned char api_js[] = {' > ${API_C}; \
    xxd -p -c 12 ${API_JS} | sed -e \"s/\\([0-9a-f]\\{2\\}\\)/0x\\1, /g\" \
                           | sed -e \"s/^/  /\" >> ${API_C}; \
    echo '};' >> ${API_C}; \
    echo \"unsigned int api_js_len = $(stat --printf='%s' ${API_JS});\" \
    >> ${API_C}"
)
# Generates frida-afl-trace library
file(GLOB_RECURSE FRIDA_MODE_SRCS frida_mode/src/*.c)
add_library(
    afl-frida-trace
    SHARED
    ${FRIDA_MODE_SRCS}
    api.c
    ${CMAKE_SOURCE_DIR}/instrumentation/afl-compiler-rt.o.c
    ${CMAKE_SOURCE_DIR}/src/afl-performance.c
)
target_include_directories(
    afl-frida-trace
    AFTER
    PRIVATE ${CMAKE_SOURCE_DIR}/include
    PRIVATE ${CMAKE_SOURCE_DIR}/frida_mode/include
    PRIVATE ${FRIDA_DIR}
)
target_link_directories(
    afl-frida-trace
    PRIVATE ${FRIDA_DIR}
)
set_target_properties(afl-frida-trace PROPERTIES PREFIX "")
set_target_properties(afl-frida-trace PROPERTIES LINKER_LANGUAGE CXX)
set_target_properties(afl-frida-trace PROPERTIES POSITION_INDEPENDENT_CODE ON)
set_target_properties(
    afl-frida-trace PROPERTIES LINK_FLAGS
    "-static-libstdc++ -DANDROID -llog -shared -z noexecstack \
    -Wl,--gc-sections -Wl,--exclude-libs,ALL -ldl \
    -Wl,--version-script=${CMAKE_SOURCE_DIR}/frida_mode/frida.map"
)
target_link_libraries(afl-frida-trace ${FRIDA_DIR}/libfrida-gumjs.a)
