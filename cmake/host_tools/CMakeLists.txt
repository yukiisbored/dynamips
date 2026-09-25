# cmake/host_tools/CMakeLists.txt
#
# Minimal sub-project that builds rom2c with the native (host) compiler.
# Invoked via ExternalProject_Add from stable/CMakeLists.txt when
# cross-compiling (e.g. with Emscripten) so that rom2c can actually run
# on the build host to generate the microcode .inc files.
#
# Required cache variables (passed via CMAKE_ARGS):
#   COMMON_DIR          path to dynamips common/ directory
#   LOCAL_DIR           path to dynamips stable/ directory (provides utils.h)
#   USE_VENDORED_LIBELF ON/OFF
#   THIRD_PARTY_DIR     path to dynamips third_party/ directory (needed when
#                       USE_VENDORED_LIBELF is ON)
#   CMAKE_MODULE_DIR    path to dynamips cmake/ module directory (needed when
#                       USE_VENDORED_LIBELF is OFF, for FindLibElf.cmake)

cmake_minimum_required ( VERSION 3.10 )
project ( rom2c_host_tools C )

set ( COMMON_DIR       "" CACHE PATH "Path to the dynamips common/ directory" )
set ( LOCAL_DIR        "" CACHE PATH "Path to the dynamips stable/ directory (provides utils.h)" )
set ( USE_VENDORED_LIBELF OFF CACHE BOOL "Use the vendored libelf from third_party/libelf" )
set ( THIRD_PARTY_DIR  "" CACHE PATH "Path to the dynamips third_party/ directory" )
set ( CMAKE_MODULE_DIR "" CACHE PATH "Path to the dynamips cmake/ module directory" )

if ( NOT COMMON_DIR )
   message ( FATAL_ERROR "COMMON_DIR must be set to the dynamips common/ directory" )
endif ()
if ( NOT LOCAL_DIR )
   message ( FATAL_ERROR "LOCAL_DIR must be set to the dynamips stable/ directory" )
endif ()

if ( USE_VENDORED_LIBELF )
   if ( NOT THIRD_PARTY_DIR )
      message ( FATAL_ERROR "THIRD_PARTY_DIR must be set when USE_VENDORED_LIBELF is ON" )
   endif ()
   # Build the vendored libelf with the host compiler.
   # VENDORED_LIBELF_INCLUDE_DIR is exported into this scope by the subdirectory.
   add_subdirectory (
      "${THIRD_PARTY_DIR}/libelf"
      "${CMAKE_CURRENT_BINARY_DIR}/libelf_build"
      )
   set ( _libelf_includes "${VENDORED_LIBELF_INCLUDE_DIR}" )
   set ( _libelf_libs libelf_vendored )
else ()
   list ( APPEND CMAKE_MODULE_PATH "${CMAKE_MODULE_DIR}" )
   find_package ( LibElf REQUIRED )
   set ( _libelf_includes "${LIBELF_INCLUDE_DIRS}" )
   set ( _libelf_libs "${LIBELF_LIBRARIES}" )
endif ()

add_executable ( rom2c "${COMMON_DIR}/rom2c.c" )
target_include_directories ( rom2c PRIVATE
   "${LOCAL_DIR}"       # utils.h
   "${COMMON_DIR}"      # dynamips_common.h
   ${_libelf_includes}
   )
target_link_libraries ( rom2c PRIVATE ${_libelf_libs} )
