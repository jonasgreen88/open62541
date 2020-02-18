# Load CMake Packages
find_package(Check REQUIRED)
set(LIBS ${CHECK_LIBRARIES} ${open62541_LIBRARIES})
if(NOT WIN32 AND NOT APPLE)
  list(APPEND LIBS subunit)
endif()

include_directories(${CHECK_INCLUDE_DIR})
#find_package(Threads REQUIRED)
if(NOT MSVC AND UA_ENABLE_UNIT_TESTS_MEMCHECK)
    find_package(Valgrind REQUIRED)
endif()

if(CMAKE_HOST_SYSTEM_NAME MATCHES "OpenBSD")
    link_directories(/usr/local/lib)
    add_definitions(-Wno-gnu-zero-variadic-macro-arguments)
endif()

if(APPLE)
    # CLang on Apple complains about many cases like this:
    # /Users/travis/build/open62541/open62541/tests/check_chunking.c:148:22: warning: implicit conversion changes signedness: 'size_t' (aka 'unsigned long') to 'intmax_t' (aka 'long') [-Wsign-conversion]
    #    ck_assert_int_eq(counter,9); //10 chunks allocated - callback called 4 times
    add_definitions(-Wno-sign-conversion)
endif()

get_property(open62541_BUILD_INCLUDE_DIRS TARGET open62541 PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
include_directories(${open62541_BUILD_INCLUDE_DIRS})
# ua_server_internal.h
include_directories("${PROJECT_SOURCE_DIR}/src")
include_directories("${PROJECT_SOURCE_DIR}/src/server")
# testing_clock.h
include_directories("${CMAKE_CURRENT_SOURCE_DIR}/testing-plugins")
# #include <src_generated/<...>.h>
include_directories("${PROJECT_BINARY_DIR}")
include_directories("${PROJECT_BINARY_DIR}/../plugins")

if(UA_ENABLE_ENCRYPTION)
    # mbedtls includes
    include_directories(${MBEDTLS_INCLUDE_DIRS})
endif()

add_definitions(-DUA_sleep_ms=UA_comboSleep)

#############################
# Compiled binaries folders #
#############################

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin/tests)
if (MSVC)
    set(TESTS_BINARY_DIR ${CMAKE_BINARY_DIR}/bin/tests/${CMAKE_BUILD_TYPE})
else()
    set(TESTS_BINARY_DIR ${CMAKE_BINARY_DIR}/bin/tests)
endif()


# Use different plugins for testing
set(test_plugin_sources ${PROJECT_SOURCE_DIR}/arch/network_tcp.c
    ${PROJECT_SOURCE_DIR}/tests/testing-plugins/testing_clock.c
    ${PROJECT_SOURCE_DIR}/plugins/ua_log_stdout.c
    ${PROJECT_SOURCE_DIR}/plugins/ua_config_default.c
    ${PROJECT_SOURCE_DIR}/plugins/ua_accesscontrol_default.c
    ${PROJECT_SOURCE_DIR}/plugins/ua_pki_default.c
    ${PROJECT_SOURCE_DIR}/plugins/ua_nodestore_ziptree.c
    ${PROJECT_SOURCE_DIR}/plugins/ua_nodestore_hashmap.c
    ${PROJECT_SOURCE_DIR}/plugins/securityPolicies/ua_securitypolicy_none.c
    ${PROJECT_SOURCE_DIR}/tests/testing-plugins/testing_policy.c
    ${PROJECT_SOURCE_DIR}/tests/testing-plugins/testing_networklayers.c
    )

if(UA_ENABLE_HISTORIZING)
    set(test_plugin_sources ${test_plugin_sources}
        ${PROJECT_SOURCE_DIR}/plugins/historydata/ua_history_data_backend_memory.c
        ${PROJECT_SOURCE_DIR}/plugins/historydata/ua_history_data_gathering_default.c
        ${PROJECT_SOURCE_DIR}/plugins/historydata/ua_history_database_default.c)
endif()

if(UA_ENABLE_ENCRYPTION)
  list(APPEND test_plugin_sources
       ${PROJECT_SOURCE_DIR}/plugins/securityPolicies/securitypolicy_mbedtls_common.c
              ${PROJECT_SOURCE_DIR}/plugins/securityPolicies/ua_securitypolicy_basic128rsa15.c
              ${PROJECT_SOURCE_DIR}/plugins/securityPolicies/ua_securitypolicy_basic256.c
              ${PROJECT_SOURCE_DIR}/plugins/securityPolicies/ua_securitypolicy_basic256sha256.c)
endif()

add_library(open62541-testplugins OBJECT ${test_plugin_sources} ${PROJECT_SOURCE_DIR}/arch/${UA_ARCHITECTURE}/ua_architecture_functions.c)
add_dependencies(open62541-testplugins open62541)
target_compile_definitions(open62541-testplugins PRIVATE -DUA_DYNAMIC_LINKING_EXPORT)

if(NOT MSVC)
    add_definitions(-Wno-deprecated-declarations)
endif()
# Workaround some clang warnings in the uni tests
if((NOT ${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD") AND (CMAKE_COMPILER_IS_GNUCC OR "x${CMAKE_C_COMPILER_ID}" STREQUAL "xClang"))
    add_definitions(-Wno-sign-conversion)
endif()

# Unit Test Definition Macro
# For now we need to disable the libc freeres. See https://github.com/open62541/open62541/pull/1003#issuecomment-315045143
# This also requires to disable the phtread cache with no-nptl-pthread-stackcache
set(VALGRIND_FLAGS --quiet --trace-children=yes --leak-check=full --run-libc-freeres=no --sim-hints=no-nptl-pthread-stackcache --track-fds=yes)
macro(add_test_valgrind TEST_NAME)
    if(UA_ENABLE_UNIT_TESTS_MEMCHECK)
        if(MSVC)
            add_test(${TEST_NAME} drmemory -batch -exit_code_if_errors 1 -results_to_stderr -summary -- ${ARGN})
        else()
            set(VALGRIND_LOG ${TESTS_BINARY_DIR}/${TEST_NAME}.log)
            set(VALGRIND_CMD valgrind --error-exitcode=1 --suppressions=${PROJECT_SOURCE_DIR}/tools/valgrind_suppressions.supp ${VALGRIND_FLAGS} --log-file=${VALGRIND_LOG} ${ARGN})
            add_test(${TEST_NAME} ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/tools/valgrind_check_error.py ${VALGRIND_LOG} ${VALGRIND_CMD})
        endif()
    else()
        add_test(${TEST_NAME} ${ARGN})
    endif()
    if(UA_BUILD_FUZZING_CORPUS)
        target_sources(check_${TEST_NAME} PRIVATE ${PROJECT_SOURCE_DIR}/tests/fuzz/ua_debug_dump_pkgs_file.c)
        file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/corpus/${TEST_NAME})
        target_compile_definitions(check_${TEST_NAME} PRIVATE UA_CORPUS_OUTPUT_DIR="${PROJECT_BINARY_DIR}/corpus/${TEST_NAME}")
    endif()
endmacro()

macro(add_test_no_valgrind TEST_NAME)
    if(NOT UA_ENABLE_UNIT_TESTS_MEMCHECK)
        add_test(${TEST_NAME} ${ARGN})
    endif()
    if(UA_BUILD_FUZZING_CORPUS)
        target_sources(check_${TEST_NAME} PRIVATE ${PROJECT_SOURCE_DIR}/tests/fuzz/ua_debug_dump_pkgs_file.c)
        file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/corpus/${TEST_NAME})
        target_compile_definitions(check_${TEST_NAME} PRIVATE UA_CORPUS_OUTPUT_DIR="${PROJECT_BINARY_DIR}/corpus/${TEST_NAME}")
    endif()
endmacro()

add_custom_target(test-verbose COMMAND ${CMAKE_CTEST_COMMAND} --verbose)

# the unit test are built directly on the open62541 object files. so they can
# access symbols that are hidden/not exported to the shared library

if(UA_ENABLE_SUBSCRIPTIONS)
  add_executable(check_local_monitored_item server/check_local_monitored_item.c $<TARGET_OBJECTS:open62541-object> $<TARGET_OBJECTS:open62541-testplugins>)
  target_link_libraries(check_local_monitored_item ${LIBS})
  add_test_valgrind(local_monitored_item ${TESTS_BINARY_DIR}/check_local_monitored_item)
endif()
