# CMakeLists.txt for the vendored libelf
# Builds libelf_vendored as a static library using the autoconf-based source
# in lib/ without running ./configure.
#
# Generated headers (config.h and sys_elf.h) are produced by configure_file
# using the CMake-specific templates config.h.cmake.in and sys_elf.h.cmake.in.
#
# Public include directory layout (in CMAKE_CURRENT_BINARY_DIR/include/):
#   libelf.h           <- thin wrapper: #include <libelf/libelf.h>
#   libelf/
#     libelf.h         <- copy of lib/libelf.h
#     gelf.h           <- copy of lib/gelf.h
#     nlist.h          <- copy of lib/nlist.h
#     elf_repl.h       <- copy of lib/elf_repl.h
#     sys_elf.h        <- generated from sys_elf.h.cmake.in
#
# LIBELF_INCLUDE_DIRS is set to CMAKE_CURRENT_BINARY_DIR/include so that
# consumers can use #include <libelf.h> and the library can satisfy
# #include <libelf/sys_elf.h> and #include <libelf/elf_repl.h>.

cmake_minimum_required(VERSION 3.5)

include(CheckIncludeFile)
include(CheckFunctionExists)
include(CheckTypeSize)
include(CheckCSourceCompiles)

# ---------------------------------------------------------------------------
# Platform detection
# ---------------------------------------------------------------------------

check_include_file("elf.h"      HAVE_ELF_H)
check_include_file("sys/elf.h"  HAVE_SYS_ELF_H)
check_include_file("ar.h"       HAVE_AR_H)
check_include_file("unistd.h"   HAVE_UNISTD_H)
check_include_file("stdint.h"   HAVE_STDINT_H)
check_include_file("fcntl.h"    HAVE_FCNTL_H)

check_function_exists(mmap       HAVE_MMAP)
check_function_exists(ftruncate  HAVE_FTRUNCATE)
check_function_exists(getpagesize HAVE_GETPAGESIZE)
check_function_exists(memcmp     HAVE_MEMCMP)
check_function_exists(memcpy     HAVE_MEMCPY)
check_function_exists(memmove    HAVE_MEMMOVE)
check_function_exists(memset     HAVE_MEMSET)

check_type_size("short"     LIBELF_SIZEOF_SHORT)
check_type_size("int"       LIBELF_SIZEOF_INT)
check_type_size("long"      LIBELF_SIZEOF_LONG)
check_type_size("long long" LIBELF_SIZEOF_LONG_LONG)

# Determine integer type aliases used in sys_elf.h / config.h
if(LIBELF_SIZEOF_LONG EQUAL 8)
  set(LIBELF_I64_T "long")
elseif(LIBELF_SIZEOF_LONG_LONG EQUAL 8)
  set(LIBELF_I64_T "long long")
else()
  message(FATAL_ERROR "vendored libelf: cannot find a 64-bit integer type")
endif()

if(LIBELF_SIZEOF_INT EQUAL 4)
  set(LIBELF_I32_T "int")
elseif(LIBELF_SIZEOF_LONG EQUAL 4)
  set(LIBELF_I32_T "long")
else()
  message(FATAL_ERROR "vendored libelf: cannot find a 32-bit integer type")
endif()

if(LIBELF_SIZEOF_SHORT EQUAL 2)
  set(LIBELF_I16_T "short")
elseif(LIBELF_SIZEOF_INT EQUAL 2)
  set(LIBELF_I16_T "int")
else()
  message(FATAL_ERROR "vendored libelf: cannot find a 16-bit integer type")
endif()

# ---------------------------------------------------------------------------
# Determine whether a usable system elf.h is present.
# If so, set __LIBELF_HEADER_ELF_H to the include path.
# ---------------------------------------------------------------------------
set(LIBELF_HEADER_ELF_H_DEFINE "/* __LIBELF_HEADER_ELF_H not defined - using elf_repl.h */")
set(_LIBELF_USING_SYSTEM_ELF_H FALSE)
set(_LIBELF_SYSTEM_ELF_INCLUDE "")  # e.g. "<elf.h>"

if(HAVE_ELF_H)
  check_c_source_compiles(
    "#include <elf.h>\nint main(void) { Elf32_Ehdr e; (void)e; return 0; }"
    LIBELF_ELF_H_WORKS)
  if(LIBELF_ELF_H_WORKS)
    set(LIBELF_HEADER_ELF_H_DEFINE "#define __LIBELF_HEADER_ELF_H <elf.h>")
    set(_LIBELF_USING_SYSTEM_ELF_H TRUE)
    set(_LIBELF_SYSTEM_ELF_INCLUDE "<elf.h>")
  endif()
endif()
if(NOT _LIBELF_USING_SYSTEM_ELF_H AND HAVE_SYS_ELF_H)
  check_c_source_compiles(
    "#include <sys/elf.h>\nint main(void) { Elf32_Ehdr e; (void)e; return 0; }"
    LIBELF_SYS_ELF_H_WORKS)
  if(LIBELF_SYS_ELF_H_WORKS)
    set(LIBELF_HEADER_ELF_H_DEFINE "#define __LIBELF_HEADER_ELF_H <sys/elf.h>")
    set(_LIBELF_USING_SYSTEM_ELF_H TRUE)
    set(_LIBELF_SYSTEM_ELF_INCLUDE "<sys/elf.h>")
  endif()
endif()

# ---------------------------------------------------------------------------
# Symbol versioning detection.
#
# Mirror the configure.in logic:
#   - When using elf_repl.h (no system elf.h): it defines both SHT_SUNW_* and
#     SHT_GNU_*, so prefer Sun-style (same as autoconf default).
#   - When using a system elf.h: probe which constants it actually provides.
#
# We store the full #define / /* #undef */ line as CMake variables so the
# @VAR@ substitution in the .cmake.in templates emits the right C text.
# ---------------------------------------------------------------------------
set(LIBELF_SYMBOL_VERSIONS_DEFINE    "/* #undef __LIBELF_SYMBOL_VERSIONS */")
set(LIBELF_SUN_SYMBOL_VERSIONS_DEFINE "/* #undef __LIBELF_SUN_SYMBOL_VERSIONS */")
set(LIBELF_GNU_SYMBOL_VERSIONS_DEFINE "/* #undef __LIBELF_GNU_SYMBOL_VERSIONS */")

if(NOT _LIBELF_USING_SYSTEM_ELF_H)
  # elf_repl.h defines both SHT_SUNW_* and SHT_GNU_*; prefer Sun-style.
  set(LIBELF_SYMBOL_VERSIONS_DEFINE    "#define __LIBELF_SYMBOL_VERSIONS 1")
  set(LIBELF_SUN_SYMBOL_VERSIONS_DEFINE "#define __LIBELF_SUN_SYMBOL_VERSIONS 1")
else()
  # Probe the system elf.h for SHT_SUNW_* (Sun/Solaris style).
  string(CONCAT _src
    "#include ${_LIBELF_SYSTEM_ELF_INCLUDE}\n"
    "int main(void) {\n"
    "  unsigned x = SHT_SUNW_verdef + SHT_SUNW_verneed + SHT_SUNW_versym;\n"
    "  (void)x; return 0;\n}\n")
  check_c_source_compiles("${_src}" LIBELF_HAVE_SHT_SUNW_VERDEF)

  if(LIBELF_HAVE_SHT_SUNW_VERDEF)
    set(LIBELF_SYMBOL_VERSIONS_DEFINE    "#define __LIBELF_SYMBOL_VERSIONS 1")
    set(LIBELF_SUN_SYMBOL_VERSIONS_DEFINE "#define __LIBELF_SUN_SYMBOL_VERSIONS 1")
  else()
    # Probe for SHT_GNU_* (Linux/glibc style).
    string(CONCAT _src
      "#include ${_LIBELF_SYSTEM_ELF_INCLUDE}\n"
      "int main(void) {\n"
      "  unsigned x = SHT_GNU_verdef + SHT_GNU_verneed + SHT_GNU_versym;\n"
      "  (void)x; return 0;\n}\n")
    check_c_source_compiles("${_src}" LIBELF_HAVE_SHT_GNU_VERDEF)
    if(LIBELF_HAVE_SHT_GNU_VERDEF)
      set(LIBELF_SYMBOL_VERSIONS_DEFINE    "#define __LIBELF_SYMBOL_VERSIONS 1")
      set(LIBELF_GNU_SYMBOL_VERSIONS_DEFINE "#define __LIBELF_GNU_SYMBOL_VERSIONS 1")
    endif()
  endif()
endif()

message(STATUS "vendored libelf: ${LIBELF_HEADER_ELF_H_DEFINE}")
message(STATUS "vendored libelf: ${LIBELF_SYMBOL_VERSIONS_DEFINE}")
message(STATUS "vendored libelf: __libelf_i64_t=${LIBELF_I64_T}  __libelf_i32_t=${LIBELF_I32_T}  __libelf_i16_t=${LIBELF_I16_T}")

# ---------------------------------------------------------------------------
# Generate config.h  (private, internal compilation only)
# ---------------------------------------------------------------------------
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/config.h"
  @ONLY
)

# ---------------------------------------------------------------------------
# Generate the public include tree under include/
#
#   include/libelf.h               <- thin wrapper
#   include/libelf/sys_elf.h       <- generated
#   include/libelf/libelf.h        <- copy
#   include/libelf/gelf.h          <- copy
#   include/libelf/nlist.h         <- copy
#   include/libelf/elf_repl.h      <- copy
# ---------------------------------------------------------------------------
set(_INC "${CMAKE_CURRENT_BINARY_DIR}/include")
set(_LIB_INC "${_INC}/libelf")

# Generated sys_elf.h
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/sys_elf.h.cmake.in"
  "${_LIB_INC}/sys_elf.h"
  @ONLY
)

# Static public headers (copied verbatim)
foreach(_hdr libelf.h gelf.h nlist.h elf_repl.h)
  configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/lib/${_hdr}"
    "${_LIB_INC}/${_hdr}"
    COPYONLY
  )
endforeach()

# Compatibility wrapper at include/libelf.h so that
# #include <libelf.h> finds this file and then resolves
# #include <libelf/sys_elf.h> etc. from the same include/ root.
file(WRITE "${_INC}/libelf.h"
  "/* Compatibility wrapper generated by CMake - do not edit */\n"
  "#include <libelf/libelf.h>\n"
)

# ---------------------------------------------------------------------------
# Source files (from lib/Makefile.in)
# ---------------------------------------------------------------------------
set(LIBELF_SRCS
  # generic
  lib/begin.c     lib/cntl.c      lib/end.c       lib/errmsg.c
  lib/errno.c     lib/fill.c      lib/flag.c      lib/getarhdr.c
  lib/getarsym.c  lib/getbase.c   lib/getdata.c   lib/getident.c
  lib/getscn.c    lib/hash.c      lib/kind.c      lib/ndxscn.c
  lib/newdata.c   lib/newscn.c    lib/next.c      lib/nextscn.c
  lib/rand.c      lib/rawdata.c   lib/rawfile.c   lib/strptr.c
  lib/update.c    lib/version.c   lib/checksum.c  lib/getaroff.c
  # 32-bit
  lib/32.fsize.c   lib/32.getehdr.c  lib/32.getphdr.c  lib/32.getshdr.c
  lib/32.newehdr.c lib/32.newphdr.c  lib/32.xlatetof.c
  # support
  lib/cook.c      lib/data.c      lib/input.c     lib/assert.c
  # nlist
  lib/nlist.c
  # optional extensions
  lib/opt.delscn.c  lib/x.remscn.c  lib/x.movscn.c  lib/x.elfext.c
  # 64-bit
  lib/64.xlatetof.c  lib/gelfehdr.c  lib/gelfphdr.c
  lib/gelfshdr.c     lib/gelftrans.c lib/swap64.c
  # symbol versioning
  lib/verdef_32_tof.c  lib/verdef_32_tom.c
  lib/verdef_64_tof.c  lib/verdef_64_tom.c
)

# Only add the memset fallback when memset is missing (very unlikely on modern systems)
if(NOT HAVE_MEMSET)
  list(APPEND LIBELF_SRCS lib/memset.c)
endif()

# ---------------------------------------------------------------------------
# Static library target
# ---------------------------------------------------------------------------
add_library(libelf_vendored STATIC ${LIBELF_SRCS})

target_compile_definitions(libelf_vendored PRIVATE HAVE_CONFIG_H=1)

target_include_directories(libelf_vendored
  # Public: consumers use #include <libelf.h> or #include <libelf/...>
  PUBLIC
    "$<BUILD_INTERFACE:${_INC}>"
  # Private: internal compilation needs:
  #   config.h          from CMAKE_CURRENT_BINARY_DIR
  #   <sys_elf.h>       as a bare include (not <libelf/sys_elf.h>)
  #   <elf_repl.h>      as a bare include
  #   private headers   (private.h, errors.h, byteswap.h, ext_types.h, …)
  PRIVATE
    "${CMAKE_CURRENT_BINARY_DIR}"         # config.h
    "${_LIB_INC}"                         # sys_elf.h, elf_repl.h as bare names
    "${CMAKE_CURRENT_SOURCE_DIR}/lib"     # private.h, errors.h, byteswap.h, …
)

# Expose the include root so parent scope can reference it
set(VENDORED_LIBELF_INCLUDE_DIR "${_INC}" PARENT_SCOPE)
