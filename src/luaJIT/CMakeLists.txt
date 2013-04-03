#
# LuaJIT 2.0.x
#
cmake_minimum_required(VERSION 2.4 FATAL_ERROR)

project(luaJIT C ASM)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

set(COMMON_CFLAGS)
set(COMMON_LDFLAGS)
set(LIBS)

include_directories ( ${CMAKE_CURRENT_BINARY_DIR} dynasm src src/host )

if(APPLE)
	set(DEFAULT_POSIX TRUE)
	set(DEFAULT_DLOPEN ON)
	# use this on Mac OS X 10.3-
	option(LUA_USE_MACOSX "Mac OS X 10.3-" OFF)
elseif(CYGWIN)
	set(DEFAULT_POSIX TRUE)
elseif(UNIX)
	set(DEFAULT_POSIX TRUE)
elseif(WIN32)
	set(LUA_WIN TRUE)
	set(COMMON_CFLAGS "${COMMON_CFLAGS} -DLUA_BUILD_AS_DLL")
else()
	set(DEFAULT_ANSI TRUE)
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
	set(COMMON_LDFLAGS "${COMMON_LDFLAGS} -Wl,-E -lm")
	set(DEFAULT_DLOPEN ON)
endif()

if(WIN32)
	#set(BUILD_STATIC OFF)
	set(BUILD_STATIC ON)
else()
	#option(BUILD_STATIC "build static library" ON)
	set(BUILD_STATIC ON)
endif()

if(DEFAULT_DLOPEN)
	option(LUA_USE_DLOPEN "Enable dlopen support." ON)
else()
	option(LUA_USE_DLOPEN "Enable dlopen support." OFF)
endif()
mark_as_advanced(LUA_USE_DLOPEN)

if(DEFAULT_POSIX)
else()
endif()

if(DEFAULT_ANSI)
	option(LUA_ANSI "Disable non-ansi features." ON)
else()
	option(LUA_ANSI "Disable non-ansi features." OFF)
endif()
mark_as_advanced(LUA_ANSI)

#
# Lua version
#
set(LUA_VERSION_MAJOR 2)
set(LUA_VERSION_MINOR 0)
set(LUA_VERSION_PATCH 1)
set(LUA_VERSION
				"${LUA_VERSION_MAJOR}.${LUA_VERSION_MINOR}.${LUA_VERSION_PATCH}")
set(LUA_SOVERSION
				"${LUA_VERSION_MAJOR}")

#
# libs & cflags
#
set(COMMON_LDFLAGS "${COMMON_LDFLAGS}")

# For "Mac OS X 10.3-"
if(LUA_USE_MACOSX)
	set(COMMON_CFLAGS "${COMMON_CFLAGS} -DLUA_USE_MACOSX")
	set(LUA_USE_DLOPEN FALSE)
endif(LUA_USE_MACOSX)

if(LUA_USE_DLOPEN)
	set(COMMON_CFLAGS "${COMMON_CFLAGS} -DLUA_USE_DLOPEN")
	if(NOT APPLE)
		set(COMMON_LDFLAGS "${COMMON_LDFLAGS} -ldl ")
	endif(NOT APPLE)
endif(LUA_USE_DLOPEN)

if(LUA_ANSI)
	set(COMMON_CFLAGS "${COMMON_CFLAGS} -DLUA_ANSI")
endif(LUA_ANSI)

#
# COMMON_CFLAGS has no effect without this line
#
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${COMMON_CFLAGS}")

#
# standard flags to use for each build type.
#
if(CMAKE_COMPILER_IS_GNUCC)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pipe -Wall -Wextra -Wshadow -W -pedantic -std=gnu99")
	set(CMAKE_C_FLAGS_RELEASE        "${CMAKE_C_FLAGS_RELEASE}     -O2")
	set(CMAKE_C_FLAGS_DEBUG          "${CMAKE_C_FLAGS_DEBUG}       -O0 -g")
	set(CMAKE_C_FLAGS_PROFILE        "${CMAKE_C_FLAGS_PROFILE}     -O1 -g")
	set(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_WITHDEBINFO} -O2 -g")
endif(CMAKE_COMPILER_IS_GNUCC)

file ( GLOB SRC_LIB src/lib_*.c )
if ( LUA_USE_RELATIVE_LOADLIB )
  list ( REMOVE_ITEM SRC_LIB ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_package.c )
else ()
  list ( REMOVE_ITEM SRC_LIB ${CMAKE_CURRENT_SOURCE_DIR}/src/lib_package_rel.c )
endif ()

# Build minilua
add_executable ( minilua src/host/minilua.c )
target_link_libraries ( minilua ${LIBS} )

# Dynasm
set ( DASM ${CMAKE_CURRENT_SOURCE_DIR}/dynasm/dynasm.lua )
set ( DASM_T ${CMAKE_CURRENT_SOURCE_DIR}/src/host/buildvm_arch.h )

set ( DASM_FLAGS "-LN" "-D" "WIN" "-D" "JIT" "-D" "FFI" "-D" "P64")
set ( DASM_DASC ${CMAKE_CURRENT_SOURCE_DIR}/src/vm_x86.dasc )

# Generate buildvm arch header

add_custom_command(OUTPUT ${DASM_T}
  COMMAND minilua ${DASM} ${DASM_FLAGS} -o ${DASM_T} ${DASM_DASC}
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/dynasm
  DEPENDS minilua
)

# Buildvm
file ( GLOB SRC_BUILDVM src/host/buildvm*.c )
add_executable ( buildvm ${SRC_BUILDVM} ${DASM_T} )

macro(add_buildvm_target _target _mode)
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${_target}
    COMMAND buildvm ARGS -m ${_mode} -o ${CMAKE_CURRENT_BINARY_DIR}/${_target} ${ARGN}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS buildvm ${ARGN}
  )
endmacro(add_buildvm_target)

if (MSVC)
  add_buildvm_target ( lj_vm.obj peobj )
  set (LJ_VM_SRC ${CMAKE_CURRENT_BINARY_DIR}/lj_vm.obj)
else ()
  add_buildvm_target ( lj_vm.s ${LJVM_MODE} )
  set (LJ_VM_SRC ${CMAKE_CURRENT_BINARY_DIR}/lj_vm.s)
endif ()
add_buildvm_target ( lj_ffdef.h   ffdef   ${SRC_LIB} )
add_buildvm_target ( lj_bcdef.h  bcdef  ${SRC_LIB} )
add_buildvm_target ( lj_folddef.h folddef src/lj_opt_fold.c )
add_buildvm_target ( lj_recdef.h  recdef  ${SRC_LIB} )
add_buildvm_target ( lj_libdef.h  libdef  ${SRC_LIB} )
add_buildvm_target ( jit/vmdef.lua  libvm  ${SRC_LIB} )

set(LUAJIT_LIB_HEADERS
	${CMAKE_CURRENT_BINARY_DIR}/lj_ffdef.h
	${CMAKE_CURRENT_BINARY_DIR}/lj_bcdef.h
	${CMAKE_CURRENT_BINARY_DIR}/lj_libdef.h
	${CMAKE_CURRENT_BINARY_DIR}/lj_recdef.h
	${CMAKE_CURRENT_BINARY_DIR}/lj_folddef.h
)

set(LUAJIT_CORE_SRC
	src/lib_base.c
	src/lib_math.c
	src/lib_bit.c
	src/lib_string.c
	src/lib_table.c
	src/lib_io.c
	src/lib_os.c
	src/lib_package.c
	src/lib_debug.c
	src/lib_jit.c
	src/lib_ffi.c
)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}
				${CMAKE_CURRENT_BINARY_DIR} dynasm src src/host )

#
# Lua library.
#
add_library(luajit STATIC ${LUAJIT_CORE_SRC} ${SRC_LIB} ${LUAJIT_LIB_HEADERS})
target_link_libraries(luajit ${LIBS})
set(LUA_STATIC_LIB luajit)
set(LUA_LIBS luajit)

set_target_properties(${LUA_LIBS} PROPERTIES
	VERSION ${LUA_VERSION}
	SOVERSION ${LUA_SOVERSION}
	CLEAN_DIRECT_OUTPUT 1
)

#
# sub-folders
#


