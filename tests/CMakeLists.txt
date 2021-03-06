
if(CMAKE_SYSTEM_NAME STREQUAL Windows)
    execute_process(COMMAND
                      "${CMAKE_COMMAND}" -E copy_directory "${PROJECT_SOURCE_DIR}/private"
                      "${CMAKE_CURRENT_BINARY_DIR}/dispatch")
    execute_process(COMMAND
                      "${CMAKE_COMMAND}" -E copy "${CMAKE_CURRENT_SOURCE_DIR}/leaks-wrapper.sh"
                      "${CMAKE_CURRENT_BINARY_DIR}/leaks-wrapper")
else()
    execute_process(COMMAND
                      "${CMAKE_COMMAND}" -E create_symlink "${PROJECT_SOURCE_DIR}/private"
                      "${CMAKE_CURRENT_BINARY_DIR}/dispatch")
    execute_process(COMMAND
                      "${CMAKE_COMMAND}" -E create_symlink "${CMAKE_CURRENT_SOURCE_DIR}/leaks-wrapper.sh"
                      "${CMAKE_CURRENT_BINARY_DIR}/leaks-wrapper")
endif()

if(CMAKE_SYSTEM_NAME STREQUAL Linux)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lrt")
endif()

add_library(bsdtests
            STATIC
              bsdtests.c
              dispatch_test.c)
target_include_directories(bsdtests
                           PRIVATE
                             ${CMAKE_CURRENT_BINARY_DIR}
                             ${CMAKE_CURRENT_SOURCE_DIR}
                             ${PROJECT_SOURCE_DIR}
                           PUBLIC
                             # bsdtests.h needs config_ac.h
                             ${PROJECT_BINARY_DIR})
if(BSD_OVERLAY_FOUND)
  target_compile_options(bsdtests
                         PRIVATE
                           ${BSD_OVERLAY_CFLAGS})
endif()
if (WIN32)
  target_sources(bsdtests
                 PRIVATE
                   generic_win_port.c)
  target_compile_definitions(bsdtests
                             PUBLIC
                               _CRT_NONSTDC_NO_WARNINGS
                               _CRT_SECURE_NO_WARNINGS)
endif ()

add_executable(bsdtestharness
               bsdtestharness.c)
dispatch_set_linker(bsdtestharness)
target_include_directories(bsdtestharness
                           PRIVATE
                             ${CMAKE_CURRENT_BINARY_DIR}
                             ${CMAKE_CURRENT_SOURCE_DIR}
                             ${PROJECT_SOURCE_DIR})
if(BSD_OVERLAY_FOUND)
  target_compile_options(bsdtestharness
                         PRIVATE
                           ${BSD_OVERLAY_CFLAGS})
endif()
target_link_libraries(bsdtestharness
                      PRIVATE
                        bsdtests
                        dispatch)
if(BSD_OVERLAY_FOUND)
  target_link_libraries(bsdtestharness
                        PRIVATE
                          ${BSD_OVERLAY_LDFLAGS})
endif()

function(add_unit_test name)
  set(options DISABLED_TEST;NO_BSD_OVERLAY)
  set(single_value_args)
  set(multiple_value_args SOURCES)
  cmake_parse_arguments(AUT "${options}" "${single_value_args}" "${multiple_value_args}" ${ARGN})

  if(AUT_DISABLED_TEST)
    return()
  endif()

  add_executable(${name} ${AUT_SOURCES})
  target_include_directories(${name}
                             PRIVATE
                               ${CMAKE_CURRENT_BINARY_DIR}
                               ${CMAKE_CURRENT_SOURCE_DIR}
                               ${PROJECT_SOURCE_DIR})
  if(ENABLE_SWIFT)
    # For testing in swift.org CI system; make deadlines lenient by default
    # to reduce probability of test failures due to machine load.
    target_compile_options(${name} PRIVATE -DLENIENT_DEADLINES=1)
  endif()
  target_include_directories(${name}
                             SYSTEM BEFORE PRIVATE
                               "${BlocksRuntime_INCLUDE_DIR}")
  if(BSD_OVERLAY_FOUND AND NOT AUT_NO_BSD_OVERLAY)
    target_compile_options(${name}
                           PRIVATE
                             ${BSD_OVERLAY_CFLAGS})
  endif()
  if("${CMAKE_C_SIMULATE_ID}" STREQUAL "MSVC")
    target_compile_options(${name} PRIVATE -Xclang -fblocks)
    target_compile_options(${name} PRIVATE /W3 -Wno-deprecated-declarations)
  else()
    target_compile_options(${name} PRIVATE -fblocks)
    target_compile_options(${name} PRIVATE -Wall -Wno-deprecated-declarations)
  endif()
  dispatch_set_linker(${name})
  target_link_libraries(${name}
                        PRIVATE
                          dispatch
                          Threads::Threads
                          BlocksRuntime::BlocksRuntime)
  if(BSD_OVERLAY_FOUND AND NOT AUT_NO_BSD_OVERLAY)
    target_link_libraries(${name}
                          PRIVATE
                            ${BSD_OVERLAY_LDFLAGS})
  endif()
  target_link_libraries(${name} PRIVATE bsdtests)
  add_test(NAME ${name}
           COMMAND bsdtestharness $<TARGET_FILE:${name}>)
  set_tests_properties(${name}
                       PROPERTIES
                         TIMEOUT 120
                         DEPENDS bsdtestharness
                         WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
  if(NOT leaks_EXECUTABLE)
    set_tests_properties(${name}
                         PROPERTIES
                           ENVIRONMENT NOLEAKS=1)
  endif()
endfunction()

# Tests that reliably pass on all platforms
set(DISPATCH_C_TESTS
    apply
    api
    debug
    queue_finalizer
    group
    overcommit
    context_for_key
    after
    timer
    timer_short
    timer_timeout
    sema
    timer_bit31
    timer_bit63
    timer_set_time
    starfish
    data
    io_net
    select)

# Tests that usually pass, but occasionally fail.
# Excluded by default for purposes of Swift CI
if(EXTENDED_TEST_SUITE)
  list(APPEND DISPATCH_C_TESTS
       priority
       concur
       read
       read2
       suspend_timer
       pingpong
       drift
       readsync
       cascade
       io)
  # an oddball; dispatch_priority.c compiled with -DUSE_SET_TARGET_QUEUE=1
  add_unit_test(dispatch_priority2 SOURCES dispatch_priority.c)
  target_compile_options(dispatch_priority2 PRIVATE -DUSE_SET_TARGET_QUEUE=1)
endif()

# add C tests for platform-specific functionality when applicable
if(CMAKE_SYSTEM_NAME STREQUAL Darwin)
  list(APPEND DISPATCH_C_TESTS
       deadname
       proc
       vm
       vnode)
endif()

foreach(test ${DISPATCH_C_TESTS})
  add_unit_test(dispatch_${test}
                SOURCES
                  dispatch_${test}.c)
endforeach()

# test dispatch API for various C/CXX language variants
add_unit_test(dispatch_c99 NO_BSD_OVERLAY SOURCES dispatch_c99.c)
add_unit_test(dispatch_plusplus SOURCES dispatch_plusplus.cpp)

# test-specific link options
target_link_libraries(dispatch_group PRIVATE m)
target_link_libraries(dispatch_timer_short PRIVATE m)

# test-specific compile options
set_target_properties(dispatch_c99 PROPERTIES C_STANDARD 99)
