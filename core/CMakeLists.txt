#
# libev library
#
add_library(ev tarantool_ev.c)

check_c_compiler_flag ("-Wno-unused-result" gcc_has_wno_unused_result)

if (gcc_has_wno_unused_result)
    set_source_files_properties(tarantool_ev.c
        PROPERTIES COMPILE_FLAGS "-Wno-unused-result")
endif()

if (TARGET_OS_LINUX)
#
# Enable Linux-specific event notification API (man inotify)
  set_target_properties(ev PROPERTIES COMPILE_FLAGS "-DEV_USE_INOTIFY")
elseif (TARGET_OS_FREEBSD)
#
# On FreeBSD build libev loop on top of
  set_target_properties(ev PROPERTIES COMPILE_FLAGS "-DEV_USE_KQUEUE")
endif()
#
# libev uses ceil and floor from the standard math library
#
target_link_libraries(ev m)

#
# Build admin.m from admin.rl, but only if admin.rl was changed.
# The same applies to memcached.m/memcached.rl.
# We track admin.m and memcached.m in revision control, and thus do not
# require engineers who do not modify .rl files to have Ragel
# installed.
#
add_custom_command(OUTPUT ${CMAKE_SOURCE_DIR}/core/admin.m
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    COMMAND ${RAGEL} -G2 core/admin.rl -o core/admin.m
    DEPENDS ${CMAKE_SOURCE_DIR}/core/admin.rl)

add_custom_target(generate_admin_m DEPENDS ${CMAKE_SOURCE_DIR}/core/admin.m)
# do not randomly try to re-generate admin.m after a fresh checkout
execute_process(COMMAND ${CMAKE_COMMAND} -E touch_nocreate
    ${CMAKE_SOURCE_DIR}/core/admin.m)

#
# Do not clean admin.m, memcached.m or other
# generated files in 'make clean' -- they are under
# revision control.
#
set_property(DIRECTORY PROPERTY CLEAN_NO_CUSTOM 1)

#
# Used by modules.
#
set (recompiled_core_sources
     ${CMAKE_SOURCE_DIR}/core/tarantool.m
     ${CMAKE_SOURCE_DIR}/core/tarantool_lua.m
     ${CMAKE_SOURCE_DIR}/core/say.m
     ${CMAKE_SOURCE_DIR}/core/assoc.m
     ${CMAKE_SOURCE_DIR}/core/admin.m
     ${CMAKE_SOURCE_DIR}/core/replication.m
     ${CMAKE_SOURCE_DIR}/core/fiber.m PARENT_SCOPE)

set (common_sources tbuf.m palloc.m util.m
    salloc.m pickle.m coro.m stat.m log_io.m
    log_io_remote.m iproto.m exception.m errcode.c latch.m)

if (ENABLE_TRACE)
  set (common_sources ${common_sources} trace.m)
endif()

add_library(core STATIC ${common_sources})
add_dependencies(core generate_headers luajit)
set_target_properties(core PROPERTIES COMPILE_FLAGS "${core_cflags}")

set (common_libraries cfg core ev coro gopt misc objc luajit)
if (TARGET_OS_LINUX)
  set (common_libraries ${common_libraries} dl)
endif()

if (ENABLE_GCOV)
  set (common_libraries ${common_libraries} gcov)
endif()

if (ENABLE_BACKTRACE AND HAVE_BFD)
  set (common_libraries ${common_libraries} bfd)
  if (ENABLE_STATIC)
     include (FindZLIB)
     set (common_libraries ${common_libraries} ${ZLIB_LIBRARIES} iberty)
  endif()
endif()

set (common_libraries ${common_libraries} PARENT_SCOPE) 
