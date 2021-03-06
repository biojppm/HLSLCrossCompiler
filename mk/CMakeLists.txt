CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

PROJECT(HLSLCrossCompilerProj C CXX)

IF(NOT CMAKE_BUILD_TYPE)
	SET(CMAKE_BUILD_TYPE "Release")
ENDIF(NOT CMAKE_BUILD_TYPE)

IF( ${CMAKE_BUILD_TYPE} STREQUAL Debug )
    # CMAKE_C_FLAGS_DEBUG will also be added to CMAKE_C_FLAGS
    MESSAGE( "Debug build" )
    ADD_DEFINITIONS(-DDEBUG -D_DEBUG -D__DEBUG__)
ENDIF()

IF( BUILD_SHARED_LIBS)
   MESSAGE("Dynamic libs")
   ADD_DEFINITIONS(-DHLSLCC_DYNLIB)
ENDIF( BUILD_SHARED_LIBS )

#SET( CMAKE_RUNTIME_OUTPUT_DIRECTORY "../bin" )
#SET( CMAKE_ARCHIVE_OUTPUT_DIRECTORY "../lib" )

include(${CMAKE_ROOT}/Modules/CheckIncludeFile.cmake)
check_include_file("stdint.h" HAVE_STDINT_H)
IF(HAVE_STDINT_H)
    add_definitions(-DEXCLUDE_PSTDINT)
ENDIF()

#INCLUDE(TestBigEndian)

#TEST_BIG_ENDIAN(BIG_ENDIAN_SYSTEM)

IF(BIG_ENDIAN_SYSTEM)
    MESSAGE("SETTING Big endian")
    ADD_DEFINITIONS(-D__BIG_ENDIAN__)
ELSE()
    MESSAGE("SETTING Little endian")
    ADD_DEFINITIONS(-D__LITTLE_ENDIAN__)
ENDIF()

IF(MSVC)
	OPTION(USE_MSVC_FAST_FLOATINGPOINT "Use MSVC /fp:fast option" ON)
	IF(USE_MSVC_FAST_FLOATINGPOINT)
		ADD_DEFINITIONS(/fp:fast)
	ENDIF(USE_MSVC_FAST_FLOATINGPOINT)
ENDIF(MSVC)

IF(WIN32)
    ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS /WX /W3 /arch:SSE2)
ENDIF()

get_filename_component(incdir ../include ABSOLUTE)
get_filename_component(incdir_int ../src/internal_includes ABSOLUTE)
get_filename_component(srcdir ../src ABSOLUTE)
get_filename_component(cbstringdir ../src/cbstring ABSOLUTE)

#INCLUDE_DIRECTORIES( ../include ../src/cbstring/ ../src/ )

file(GLOB CBSTRING_CFILES ../src/cbstring/*.c)
file(GLOB CBSTRING_HFILES ../src/cbstring/*.h)
source_group("CBString Files" FILES ${CBSTRING_CFILES} ${CBSTRING_HFILES})

SET(libHLSLcc_SOURCES
../src/internal_includes/debug.h
../src/internal_includes/decode.h
../src/internal_includes/languages.h
../include/pstdint.h
../src/internal_includes/reflect.h
../src/internal_includes/structs.h
../include/hlslcc.h
../src/internal_includes/toGLSLDeclaration.h
../src/internal_includes/toGLSLInstruction.h
../src/internal_includes/toGLSLOperand.h
../src/internal_includes/tokens.h
../src/internal_includes/tokensDX9.h
../src/internal_includes/shaderLimits.h
../src/internal_includes/hlslcc_malloc.h
../src/internal_includes/hlslcc_malloc.c
${CBSTRING_HFILES}
../src/decode.c
../src/decodeDX9.c
../src/reflect.c
../src/toGLSL.c
../src/toGLSLInstruction.c
../src/toGLSLOperand.c
../src/toGLSLDeclaration.c
${CBSTRING_CFILES}
)
ADD_LIBRARY( libHLSLcc ${libHLSLcc_SOURCES} )
LIST(APPEND targets libHLSLcc)

file(GLOB pub_headers "${incdir}/*.h" "${srcdir}/*.h" "${cbstringdir}/*.h")
file(GLOB prv_headers "${incdir_int}/*.h")
message(STATUS "pub_headers=${pub_headers}")
message(STATUS "prv_headers=${prv_headers}")

set_target_properties(libHLSLcc PROPERTIES
    PUBLIC_HEADER "${pub_headers}"
    PRIVATE_HEADER "${prv_headers}")

# Define headers for this library. PUBLIC headers are used for
# compiling the library, and will be added to consumers' build
# paths.
target_include_directories(libHLSLcc
    PUBLIC
    $<BUILD_INTERFACE:${incdir}>
    $<BUILD_INTERFACE:${srcdir}>
    $<BUILD_INTERFACE:${cbstringdir}>
    $<INSTALL_INTERFACE:include>
    PRIVATE ${incdir_int}
    )


file(GLOB CJSON_CFILES ../offline/cjson/*.c)
file(GLOB CJSON_HFILES ../offline/cjson/*.h)
source_group("cJSON Files" FILES ${CJSON_CFILES} ${CJSON_HFILES})

SET(HLSLcc_SOURCES
../offline/toGLSLStandalone.cpp
../offline/timer.h
../offline/timer.cpp
../offline/hash.h
../offline/serializeReflection.cpp
../offline/serializeReflection.h
${CJSON_CFILES}
${CJSON_HFILES}
)

ADD_EXECUTABLE( HLSLcc ${HLSLcc_SOURCES} )
target_include_directories(HLSLcc
    PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../offline/cjson>
    )
LIST(APPEND targets HLSLcc)
TARGET_LINK_LIBRARIES( HLSLcc libHLSLcc)

# Install target (will install the library to specified CMAKE_INSTALL_PREFIX variable)
install(TARGETS ${targets}
  EXPORT HLSLccConfig
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
  PUBLIC_HEADER DESTINATION include/HLSLcc)

# This makes the project importable from the install directory
# Put config file in per-project dir (name MUST match), can also
# just go into <prefix>/cmake.
install(EXPORT HLSLccConfig DESTINATION cmake)

# This makes the project importable from the build directory
export(TARGETS libHLSLcc FILE HLSLccConfig.cmake)

# force variables that could be defined on the cmdline
# to be written to the cache

SET( CMAKE_BUILD_TYPE "${CMAKE_BUILD_TYPE}" CACHE STRING
"One of None Debug Release RelWithDebInfo MinSizeRel" FORCE )
