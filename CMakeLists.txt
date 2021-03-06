cmake_minimum_required(VERSION 2.8)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")
set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE INTERNAL "limit build types" FORCE)
set(CMAKE_BUILD_TYPE "Release" CACHE STRING "default build type")

set(CMAKE_INSTALL_SO_NO_EXE False)
set(CMAKE_LEGACY_CYGWIN_WIN32 False)

project(arib_std_b25 C)
enable_language(CXX)

include(GitRevision)
include(GenerateExportHeader)
find_package(PCSC REQUIRED)

if (UNIX OR MSYS)
	find_program(LDCONFIG_EXECUTABLE "ldconfig")
	set(WITH_PCSC_PACKAGE "libpcsclite" CACHE STRING "pkg-config package name for PC/SC")
	set(WITH_PCSC_LIBRARY "" CACHE STRING "library name for PC/SC")
endif()

if(WIN32)
	option(USE_UNICODE "enable unicode support" ON)
endif()

set(ARIB25_LIB_NAME "arib25")
set(ARIB25_CMD_NAME "b25")

set(ARIB25_URL "https://github.com/stz2012/libarib25")
set(ARIB25_DESCRIPTION "Reference implementation of ARIB STD-B25")
set(ARIB25_COMPANY "㋲製作所")
set(ARIB25_COPYRIGHT "©2012 茂木 和洋")
set(ARIB25_PRODUCT "ARIB STD-B25")

set(ARIB25_MAJOR_VERSION "0")
set(ARIB25_MINOR_VERSION "2")
set(ARIB25_PATCH_VERSION "5")
set(ARIB25_VERSION_NUMBER ${ARIB25_MAJOR_VERSION}.${ARIB25_MINOR_VERSION}.${ARIB25_PATCH_VERSION})
set(ARIB25_VERSION_STRING stz-${ARIB25_VERSION_NUMBER})

if(CMAKE_C_COMPILER_ID MATCHES "(GNU|Clang)")
	set(CMAKE_C_FLAGS "-Wall")
	set(CMAKE_C_FLAGS_DEBUG "-O2 -g")
	set(CMAKE_C_FLAGS_RELEASE "-O2")
	set(CMAKE_SHARED_LINKER_FLAGS "-fvisibility=hidden")

	if(UNIX AND NOT CYGWIN)
		include(ElfInterp)
		set(CMAKE_C_FLAGS "-fPIC ${CMAKE_C_FLAGS}")
		if(NOT APPLE)
			set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-eshow_version")
		endif()
	else(MINGW)
		set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--enable-stdcall-fixup -static-libgcc")
	endif()

	if(MINGW AND USE_UNICODE)
		set(CMAKE_C_FLAGS "-municode ${CMAKE_C_FLAGS}")
	endif()
elseif(CMAKE_C_COMPILER_ID MATCHES "(MSVC)")
	add_definitions("-D_CRT_SECURE_NO_WARNINGS")
	set(CMAKE_STATIC_LIBRARY_PREFIX lib)
	set(CMAKE_SHARED_LIBRARY_PREFIX lib)
	set(CMAKE_SUPPRESS_REGENERATION TRUE)
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
	add_definitions("-DDEBUG")
endif()

if(USE_UNICODE)
	add_definitions("-DUNICODE" "-D_UNICODE")
endif()

add_definitions("-D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64")
include_directories(${CMAKE_CURRENT_BINARY_DIR})
if(PCSC_INCLUDE_DIRS)
	include_directories(${PCSC_INCLUDE_DIRS})
endif()
link_directories(${PCSC_LIBRARY_DIRS})

add_library(arib25-objlib OBJECT src/arib_std_b25.c src/b_cas_card.c src/multi2.c src/ts_section_parser.c src/version.c)
set_target_properties(arib25-objlib PROPERTIES COMPILE_DEFINITIONS ARIB25_DLL)

add_library(arib25-static STATIC $<TARGET_OBJECTS:arib25-objlib>)
set_target_properties(arib25-static PROPERTIES OUTPUT_NAME ${ARIB25_LIB_NAME})
target_link_libraries(arib25-static PRIVATE ${PCSC_LIBRARIES})

add_library(arib25-shared SHARED $<TARGET_OBJECTS:arib25-objlib> ${CMAKE_CURRENT_BINARY_DIR}/version.rc)
set_target_properties(arib25-shared PROPERTIES MACOSX_RPATH ON)
set_target_properties(arib25-shared PROPERTIES DEFINE_SYMBOL ARIB25_DLL)
set_target_properties(arib25-shared PROPERTIES COMPILE_DEFINITIONS ARIB25_DLL)
set_target_properties(arib25-shared PROPERTIES OUTPUT_NAME ${ARIB25_LIB_NAME})
set_target_properties(arib25-shared PROPERTIES SOVERSION ${ARIB25_MAJOR_VERSION})
set_target_properties(arib25-shared PROPERTIES VERSION ${ARIB25_VERSION_NUMBER})
target_link_libraries(arib25-shared PRIVATE ${PCSC_LIBRARIES})
generate_export_header(arib25-shared BASE_NAME arib25_api EXPORT_FILE_NAME arib25_api.h)

add_executable(b25 src/td.c ${CMAKE_CURRENT_BINARY_DIR}/version.rc)
set_target_properties(b25 PROPERTIES OUTPUT_NAME ${ARIB25_CMD_NAME})
target_link_libraries(b25 PRIVATE ${PCSC_LIBRARIES})
target_link_libraries(b25 PRIVATE arib25-shared)

configure_file(src/config.h.in config.h @ONLY)
configure_file(src/version.rc.in version.rc @ONLY)

if(UNIX AND NOT CYGWIN)
	configure_file(src/libarib25.pc.in ${CMAKE_SHARED_LIBRARY_PREFIX}${ARIB25_LIB_NAME}.pc @ONLY)

	install(TARGETS b25 RUNTIME DESTINATION bin)
	install(TARGETS arib25-static arib25-shared ARCHIVE DESTINATION lib LIBRARY DESTINATION lib)
	install(FILES src/arib_std_b25.h src/b_cas_card.h src/multi2.h src/ts_section_parser.h src/portable.h ${CMAKE_CURRENT_BINARY_DIR}/arib25_api.h DESTINATION include/arib25)
	install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}${ARIB25_LIB_NAME}.pc DESTINATION lib/pkgconfig)
	install(CODE "execute_process(COMMAND ${CMAKE_COMMAND} -DLDCONFIG_EXECUTABLE=${LDCONFIG_EXECUTABLE} -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/PostInstall.cmake)")
	
	add_custom_target(uninstall ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Uninstall.cmake)
endif()

