CMAKE_MINIMUM_REQUIRED( VERSION 3.9.0 )
PROJECT( luamsgpack_c C )

INCLUDE(CheckCCompilerFlag)
IF( CMAKE_C_COMPILER_ID MATCHES "Clang" )
  SET(CMAKE_COMPILER_IS_CLANG 1) # using Clang or AppleClang
ENDIF()

# Lua
IF( NOT LUA_INCLUDE_DIR OR (WIN32 AND NOT LUA_LIBRARIES) )
  SET( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/" )
  FIND_PACKAGE( Lua REQUIRED )
ENDIF()

################################################################################
# Path/Library Setup.
################################################################################

SET(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${CMAKE_CURRENT_SOURCE_DIR}/etc/")
IF( NOT CMAKE_BUILD_TYPE )
  SET(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, \
  options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
ENDIF()

SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/" CACHE STRING "Library build output directory")
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/")
SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/")

SET(CMAKE_INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE PATH "Installation directory for libraries")
SET(CMAKE_INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "Installation directory for headers")

################################################################################
# Compiler Setup
################################################################################

# Enables the use of Intel AVX2 if/when possible
OPTION(LUA_NATIVE_ARCH "Enable compiler optimizations for the native processor architecture" ON)

SET(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Windows systems
IF( MSVC )
  OPTION(LUA_USE_WINDOWS "Windows specific build" ON)
  OPTION(LUA_BUILD_AS_DLL "Build Lua library as DLL" ON)
  IF( NOT( ${CMAKE_COMPILER_IS_CLANG} ) )
    MESSAGE(WARNING "MSVC detected! For MSVC/msbuild environments, it is advised to use the LLVM/ClangCL toolset")
  ENDIF()

  IF(MSVC AND NOT "${CMAKE_BUILD_TYPE}" MATCHES "Debug")
    SET(MSVC_OPT_FLAG "/Ox /DNDEBUG")
  ELSE()
    SET(MSVC_OPT_FLAG "")
  ENDIF()

  IF( LUA_NATIVE_ARCH )
    CHECK_C_COMPILER_FLAG("/arch:AVX2" COMPILER_OPT_ARCH_AVX2_SUPPORTED)
    IF( LUA_NATIVE_ARCH AND COMPILER_OPT_ARCH_AVX2_SUPPORTED )
      ADD_COMPILE_OPTIONS(/arch:AVX2)
    ENDIF()
  ENDIF()

  ADD_COMPILE_OPTIONS(/nologo /Zc:__cplusplus)
  ADD_COMPILE_DEFINITIONS(NOMINMAX)
  ADD_COMPILE_DEFINITIONS(_CRT_SECURE_NO_DEPRECATE)
  ADD_COMPILE_DEFINITIONS(_CRT_NONSTDC_NO_DEPRECATE)

  SET(LUA_C_FLAGS "${MSVC_OPT_FLAG} /Ob2 /Oi /Ot /GT /GF /GS- /MP /EHsc /GL")
  SET(LUA_REL_FLAGS "${LUA_REL_FLAGS} /GL")
  SET(LUA_LINKFLAGS "${LUA_LINKFLAGS} /LTCG")

  # Visual Studio 2019 feature
  CHECK_C_COMPILER_FLAG("/Ob3" COMPILER_OPT_OB3)
  IF( COMPILER_OPT_OB3 )
    STRING(REGEX REPLACE "[-/]Ob[1-4]" "" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
    STRING(REGEX REPLACE "[-/]Ob[1-4]" "" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
    STRING(REPLACE "/Ob2" "/Ob3" LUA_C_FLAGS "${LUA_C_FLAGS}")

    SET(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /Ob3")
    SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Ob3")
  ENDIF()

  SET(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${LUA_C_FLAGS} ${LUA_REL_FLAGS} /Oy")
  SET(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO} ${LUA_C_FLAGS} ${LUA_REL_FLAGS} /Zi")

  SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${LUA_C_FLAGS} ${LUA_REL_FLAGS} /Oy")
  SET(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} ${LUA_C_FLAGS} ${LUA_REL_FLAGS} /Zi")

  SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS_RELEASE} ${LUA_LINKFLAGS}")
  SET(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} ${LUA_LINKFLAGS}")
  SET(CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO} ${LUA_LINKFLAGS}")

  SET(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} ${LUA_LINKFLAGS}")
  SET(CMAKE_SHARED_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_SHARED_LINKER_FLAGS_RELWITHDEBINFO} ${LUA_LINKFLAGS}")

  SET(CMAKE_MODULE_LINKER_FLAGS_RELEASE "${CMAKE_MODULE_LINKER_FLAGS_RELEASE} ${LUA_LINKFLAGS}")
  SET(CMAKE_MODULE_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_MODULE_LINKER_FLAGS_RELWITHDEBINFO} ${LUA_LINKFLAGS}")

  # W4 to turn compiler warnings all the way up...
  STRING(REGEX REPLACE "[-/]W[1-4]" "" CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
  STRING(REGEX REPLACE "[-/]W[1-4]" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
  SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /W3")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W3")

# Posix systems (including msys2/cygwin)
ELSE()
  OPTION(LUA_BIT32 "32bit compile" OFF)
  OPTION(LUA_USE_POSIX "Use POSIX functionality" ON)

  IF( APPLE )
    SET(PLAT "macosx")
    SET(LINK_FLAGS "-bundle -undefined dynamic_lookup -all_load")
  ENDIF()

  IF( LUA_BIT32 )
    IF( "${CMAKE_C_COMPILER_ID}" STREQUAL "GNU" OR CMAKE_COMPILER_IS_CLANG )
      SET(CMAKE_C_FLAGS "-m32 ${CMAKE_C_FLAGS}")
      SET(CMAKE_CXX_FLAGS "-m32 ${CMAKE_CXX_FLAGS}")
    ENDIF()
  ENDIF()

  # Compile everything with -march=native
  IF( LUA_NATIVE_ARCH )
    CHECK_C_COMPILER_FLAG("-march=native" COMPILER_OPT_ARCH_NATIVE_SUPPORTED)
    IF( LUA_NATIVE_ARCH AND COMPILER_OPT_ARCH_NATIVE_SUPPORTED )
      ADD_COMPILE_OPTIONS(-march=native)
    ENDIF()
  ENDIF()

  # Temporary fix: CMake no longer defines WIN32 on MSYS.
  IF( CYGWIN OR MSYS )
    SET(LUA_BUILD_AS_DLL ON)
    ADD_COMPILE_DEFINITIONS(_WIN32)
  ENDIF()

  # Compiler options taken from Lua: lua/blob/master/makefile
  ADD_COMPILE_OPTIONS(
    -Wall -Wextra -Wsign-compare -Wundef -Wwrite-strings -Wdisabled-optimization
    -Wno-inline -Wunused-parameter -Wno-comment -Wno-unused-function
    -Wno-array-bounds -Wno-sign-conversion -Wformat=2
    # -Wshadow
    # -Werror
    # -Wconversion
    # -Wsign-conversion
    # -Wcast-qual
  )

  IF (NOT CMAKE_C_COMPILER_ID STREQUAL "Intel")
    ADD_COMPILE_OPTIONS(
        -Wredundant-decls -Wdouble-promotion -Wstrict-overflow=2
        -Wno-format-nonliteral
    )
  ENDIF()

  IF( CMAKE_COMPILER_IS_CLANG )
    ADD_COMPILE_OPTIONS(-ferror-limit=5 -fno-common -Wno-deprecated
      # Clang Everything support;
      #-Weverything
      #-Wno-gnu-zero-variadic-macro-arguments
      #-Wno-covered-switch-default
      #-Wno-reserved-id-macro
      #-Wno-unused-macros
      #-Wno-padded
      #-Wno-old-style-cast
      #-Wno-extra-semi-stmt
      #-Wno-c++98-compat-pedantic
      #-Wno-zero-as-null-pointer-constant
      #-Wno-gnu-anonymous-struct
      #-Wno-gnu-label-as-value
      #-Wno-nested-anon-types
      #-Wno-enum-enum-conversion
    )
  ELSEIF( CMAKE_COMPILER_IS_GNUCC )
    ADD_COMPILE_OPTIONS(-fmax-errors=5 -Wduplicated-cond -Wrestrict -Wlogical-op
      -Wno-aggressive-loop-optimizations
      -Wno-ignored-qualifiers
      #-Wnull-dereference
      #-Wduplicated-branches
    )
  ENDIF()
ENDIF()

SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
SET(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
SET(CMAKE_CXX_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL}")

################################################################################
# Build Options
################################################################################

OPTION(LUA_COMPILED_AS_HPP "Library compiled for Lua with C++ linkage" OFF)
OPTION(LUA_INCLUDE_TEST "Include ltests.h" OFF)

OPTION(LUA_MSGPACK_SAFE "Wrap API functions in pcall" OFF)
OPTION(LUA_MSGPACK_COMPAT "Compatible pack()" OFF)
OPTION(LUA_MSGPACK_BIT32 "32bit compile" OFF)
OPTION(LUA_MSGPACK_ERROR_NESTING "Throw an error when exceeding the maximum table nesting depth" OFF)

SET(MP_MAX_NESTING CACHE STRING "Maximum recursive-depth/table-nesting")
SET(MP_BUFFER_INITSIZE CACHE STRING "Initial intermediate string buffer size")
SET(MP_ZONE_CHUNK_SIZE CACHE STRING "Default chunk msgpack_zone chunk size")
SET(MP_TABLE_CUTOFF CACHE STRING
  "Threshold for mp_table_is_an_array. If a table of only integer keys has a \
  key greater than this value: ensure at least half of the keys within the \
  table have non-nil objects to be encoded as an array."
)

IF( CMAKE_BUILD_TYPE STREQUAL Debug )
  SET(LUA_INCLUDE_TEST ON)
ENDIF()

IF( LUA_COMPILED_AS_HPP )
  ADD_COMPILE_DEFINITIONS(LUA_COMPILED_AS_HPP)
ENDIF()

IF( LUA_INCLUDE_TEST )
  ADD_COMPILE_DEFINITIONS(LUA_USER_H="ltests.h")
ENDIF()

IF( LUA_MSGPACK_COMPAT )
  ADD_COMPILE_DEFINITIONS(LUA_MSGPACK_COMPAT)
ENDIF()

IF( LUA_MSGPACK_SAFE )
  ADD_COMPILE_DEFINITIONS(LUA_MSGPACK_SAFE)
ENDIF()

IF( LUA_MSGPACK_BIT32  )
  ADD_COMPILE_DEFINITIONS(LUA_MSGPACK_BIT32)
ENDIF()

IF( LUA_MSGPACK_ERROR_NESTING  )
  ADD_COMPILE_DEFINITIONS(LUA_MSGPACK_ERROR_NESTING)
ENDIF()

IF( MP_MAX_NESTING )
  ADD_COMPILE_DEFINITIONS(MP_MAX_NESTING=${MP_MAX_NESTING})
ENDIF()

IF( MP_BUFFER_INITSIZE )
  ADD_COMPILE_DEFINITIONS(MP_BUFFER_INITSIZE=${MP_BUFFER_INITSIZE})
ENDIF()

IF( MP_ZONE_CHUNK_SIZE )
  ADD_COMPILE_DEFINITIONS(MP_ZONE_CHUNK_SIZE=${MP_ZONE_CHUNK_SIZE})
ENDIF()

IF( MP_TABLE_CUTOFF )
  ADD_COMPILE_DEFINITIONS(MP_TABLE_CUTOFF=${MP_TABLE_CUTOFF})
ENDIF()

################################################################################
# msgpack-c external dependency
################################################################################

ENABLE_LANGUAGE(CXX)

SET(MSGPACK_DIR "${CMAKE_BINARY_DIR}/msgpack" CACHE PATH "msgpack directory" FORCE)
SET(MSGPACK_INSTALL_DIR "${MSGPACK_DIR}/src/msgpack/" CACHE PATH "msgpack installation directory" FORCE)
IF (LUA_MSGPACK_BIT32)
  SET(MSGPACK_C_32BIT "-DMSGPACK_32BIT=ON")
ELSE ()
  SET(MSGPACK_C_32BIT "")
ENDIF()

include(ExternalProject)
ExternalProject_Add(msgpack
  PREFIX msgpack
  GIT_REPOSITORY https://github.com/msgpack/msgpack-c.git
  GIT_TAG "4d36456799183a59654a4fae243af4c19e49d871"
  BUILD_IN_SOURCE 1
  # SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/etc/msgpack-c"

  TEST_COMMAND ""
  UPDATE_COMMAND ""
  PATCH_COMMAND ""
  INSTALL_COMMAND ""
  EXCLUDE_FROM_ALL TRUE
  CMAKE_ARGS +=
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=true
    -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DMSGPACK_BUILD_EXAMPLES=OFF
    -DMSGPACK_BUILD_TESTS=OFF
    -DBUILD_SHARED_LIBS=OFF
    -DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}
    -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
    -DCMAKE_EXE_LINKER_FLAGS=${CMAKE_EXE_LINKER_FLAGS}
    ${MSGPACK_C_32BIT}
)

SET(MSGPACK_INCLUDE_DIR ${MSGPACK_INSTALL_DIR}/include CACHE PATH "msgpack include directory" FORCE)
IF( WIN32 )
  # MSVC hack
  SET(MSGPACK_LIB_PATH "${MSGPACK_DIR}/src/msgpack/${CMAKE_BUILD_TYPE}/msgpackc.lib")
  SET(MSGPACK_LIBRARY ${MSGPACK_LIB_PATH} CACHE FILEPATH "msgpack library" FORCE)
ELSE()
  SET(MSGPACK_LIBRARY ${MSGPACK_DIR}/src/msgpack/libmsgpackc.a CACHE FILEPATH "msgpack library" FORCE)
ENDIF()

################################################################################
# Compilation
################################################################################

INCLUDE_DIRECTORIES(${LUA_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${MSGPACK_INCLUDE_DIR})

SET(SRC_LUAMSGPACK_C src/lua_cmsgpack.c )
SET(PUB_LUAMSGPACK_C src/lua_cmsgpacklib.h )
IF( LUA_COMPILED_AS_HPP )
  SET_SOURCE_FILES_PROPERTIES(${SRC_LUAMSGPACK_C} PROPERTIES LANGUAGE CXX )
ELSE()
  SET_SOURCE_FILES_PROPERTIES(${SRC_LUAMSGPACK_C} PROPERTIES LANGUAGE C )
ENDIF()

ADD_LIBRARY(luamsgpack_c MODULE ${SRC_LUAMSGPACK_C})
ADD_DEPENDENCIES(luamsgpack_c msgpack)

SET_TARGET_PROPERTIES(luamsgpack_c PROPERTIES PREFIX "")
SET_TARGET_PROPERTIES(luamsgpack_c PROPERTIES OUTPUT_NAME "cmsgpack")
IF( LUA_BIT32 )
  SET_TARGET_PROPERTIES(luamsgpack_c PROPERTIES COMPILE_FLAGS "-m32" LINK_FLAGS "-m32")
ENDIF()

IF( LINK_FLAGS )
  SET_TARGET_PROPERTIES(luamsgpack_c PROPERTIES LINK_FLAGS "${LINK_FLAGS}")
ENDIF()

IF( LUA_BUILD_AS_DLL )
  TARGET_COMPILE_DEFINITIONS(luamsgpack_c PUBLIC LUA_BUILD_AS_DLL)
ENDIF()

# Win32 modules need to be linked to the Lua library.
IF( WIN32 OR CYGWIN OR MSYS )
  TARGET_INCLUDE_DIRECTORIES(luamsgpack_c PRIVATE ${INCLUDE_DIRECTORIES})
  TARGET_LINK_LIBRARIES(luamsgpack_c ${LUA_LIBRARIES})
ENDIF()

TARGET_LINK_LIBRARIES(luamsgpack_c ${MSGPACK_LIBRARY})
IF( CMAKE_INSTALL_PREFIX )
  INSTALL(TARGETS luamsgpack_c DESTINATION "${CMAKE_INSTALL_PREFIX}/lib")
ENDIF(CMAKE_INSTALL_PREFIX)

################################################################################
# Install
################################################################################

# Don't allow installation if including test builds
SET(SKIP_INSTALL SKIP_INSTALL_ALL OR LUA_INCLUDE_TEST)

IF( NOT SKIP_INSTALL_HEADERS AND NOT ${SKIP_INSTALL} )
  INSTALL(FILES ${PUB_LUAMSGPACK_C} DESTINATION "${CMAKE_INSTALL_INC_DIR}")
ENDIF()

IF( NOT SKIP_INSTALL_LIBRARIES AND NOT ${SKIP_INSTALL} )
  IF( UNIX AND NOT APPLE )
    SET_TARGET_PROPERTIES(luamsgpack_c PROPERTIES INSTALL_RPATH "${CMAKE_INSTALL_LIB_DIR}" )
  ENDIF()

  INSTALL(TARGETS luamsgpack_c
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIB_DIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIB_DIR}
  )
ENDIF()
