# We need this as a SOURCES array and not _just_ an OBJECT library because
# these sources will be compiled multiple times (with name-prefix jet-* and
# without (regular distribution)).  The regular distribution does use OBJECT
# library so we don't build the same sources twice for shared and static.
set(JEMALLOC_CMAKE_SOURCES
    arena.c
    base.c
    bitmap.c
    ckh.c
    ctl.c
    extent.c
    extent_dss.c
    extent_mmap.c
    jemalloc.c
    large.c
    mutex.c
    nstime.c
    pages.c
    prof.c
    rtree.c
    stats.c
    tcache.c
    tsd.c
    util.c
    witness.c
    )

if(CMAKE_SYSTEM_NAME MATCHES "Darwin")
    list(APPEND JEMALLOC_CMAKE_SOURCES zone.c)
endif()

# These objects only have code in debug builds (due to macros)
if(CMAKE_BUILD_TYPE MATCHES Debug)
    list(APPEND JEMALLOC_CMAKE_SOURCES
        atomic.c
        hash.c
        prng.c
        spin.c
        ticker.c)
    # No symbols even in debug mode
    # mb.c
endif()

if(enable_valgrind)
    list(APPEND C_SRCS valgrind.c)
endif()

# Bring sources into an object library so we can include elsewhere
# and also so we don't end up compiling the same code twice for no reason.
add_library(${PROJECT_NAME}-obj OBJECT ${JEMALLOC_CMAKE_SOURCES})
target_include_directories(${PROJECT_NAME}-obj PUBLIC "$<BUILD_INTERFACE:${JEMALLOC_INC_DIR}>")

# The original library, delivery product: static or shared via BUILD_SHARED_LIBS
add_library(${PROJECT_NAME} $<TARGET_OBJECTS:${PROJECT_NAME}-obj>)
target_include_directories(${PROJECT_NAME} PUBLIC "$<BUILD_INTERFACE:${JEMALLOC_INC_DIR}>")

if (JEMALLOC_BUILD_TESTS)
    # Now add shared library. Needed for integration tests and a benchmark  
    add_library(${PROJECT_NAME}-shared SHARED $<TARGET_OBJECTS:${PROJECT_NAME}-obj>)
    target_include_directories(${PROJECT_NAME}-shared PUBLIC "$<BUILD_INTERFACE:${JEMALLOC_INC_DIR}>")
endif()

if(MSVC)
  target_include_directories(${PROJECT_NAME} PUBLIC "$<BUILD_INTERFACE:${JEMALLOC_INC_DIR}>")
endif()

# Need to export from DLL
target_compile_definitions(${PROJECT_NAME} PRIVATE DLLEXPORT)

if(with-jemalloc-prefix)
    target_compile_definitions(${PROJECT_NAME} PRIVATE JEMALLOC_MANGLE)
endif()

if(enable-prefixed-jemalloc-library)
   ############################################################################
   # JET prefixed version of jemalloc library
   # necessary to compliment non-exported symbols
   # Note: we must references the SOURCES here and not the OBJECT library
   #       because we must re-build with different namespace options.
   add_library(${PROJECT_NAME}-jet-static STATIC ${JEMALLOC_CMAKE_SOURCES})
   target_compile_definitions(${PROJECT_NAME}-jet-static PRIVATE "${C_JETLIB_DEFS}")
   target_include_directories(${PROJECT_NAME}-jet-static PUBLIC "$<BUILD_INTERFACE:${JEMALLOC_INC_DIR}>")
endif()

### Install ###
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${version_config}"
  VERSION ${jemalloc_version}
  COMPATIBILITY SameMajorVersion
  )

# Note: use 'targets_export_name'
configure_package_config_file(
  "${PROJECT_SOURCE_DIR}/cmake/Config.cmake.in"
  "${project_config}"
  INSTALL_DESTINATION "${config_install_dir}"
  )

install(
  TARGETS ${PROJECT_NAME}
  EXPORT "${targets_export_name}"
  LIBRARY DESTINATION "lib"
  ARCHIVE DESTINATION "lib"
  RUNTIME DESTINATION "bin"
  INCLUDES DESTINATION "${include_install_dir}"
  )

install(
  DIRECTORY "${JEMALLOC_TOP_DIR}/include/jemalloc" # No trailing slash
  DESTINATION "${include_install_dir}"
  FILES_MATCHING PATTERN "*.h"
  PATTERN "*internal*" EXCLUDE
  )

install(
  FILES "${project_config}" "${version_config}"
  DESTINATION "${config_install_dir}"
  )

install(
  EXPORT "${targets_export_name}"
  NAMESPACE "${namespace}"
  DESTINATION "${config_install_dir}"
  )
