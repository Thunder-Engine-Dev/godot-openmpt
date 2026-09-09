cmake_minimum_required(VERSION 3.16)

project(godot-openmpt LANGUAGES CXX)

# Default debug flags inject -g /Zi and can push binaries well over 100MB,
# especially once libopenmpt is included. Debug info is added per-target below.
option(LIBOPENMPT_DEBUG_SYMBOLS "Compile libopenmpt with debug symbols (very large binaries)" OFF)

set(GODOT_GDEXTENSION_DIR "${CMAKE_CURRENT_SOURCE_DIR}/godot-cpp/gdextension" CACHE PATH "Path to GDExtension interface headers")
set(CPP_BINDINGS_PATH "${CMAKE_CURRENT_SOURCE_DIR}/godot-cpp" CACHE PATH "Path to godot-cpp")
set(GODOT_LIB_PATH "${CMAKE_CURRENT_SOURCE_DIR}/custom-godotcpp-build/bin" CACHE PATH "Path to built godot-cpp libraries")
set(GODOT_GEN_INCLUDE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/custom-godotcpp-build/gen/include" CACHE PATH "Path to generated godot-cpp headers")
set(EXAMPLE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/example" CACHE PATH "Path to the example Godot project")

set(BUILD_PATH "${CMAKE_CURRENT_SOURCE_DIR}/bin")
set(ADDON_DIR "${CMAKE_CURRENT_SOURCE_DIR}/addons/godot-openmpt")
set(ADDON_BIN_DIR "${ADDON_DIR}/bin")

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Build type" FORCE)
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS Debug Release RelWithDebInfo MinSizeRel)
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# GDExtension output names already include a "lib" prefix (libgdmpt-...).
set(CMAKE_STATIC_LIBRARY_PREFIX "")
set(CMAKE_SHARED_LIBRARY_PREFIX "")

foreach(_output_var ARCHIVE LIBRARY RUNTIME)
	set(CMAKE_${_output_var}_OUTPUT_DIRECTORY "${BUILD_PATH}")
	foreach(_config DEBUG RELEASE RELWITHDEBINFO MINSIZEREL)
		set(CMAKE_${_output_var}_OUTPUT_DIRECTORY_${_config} "${BUILD_PATH}")
	endforeach()
endforeach()

if(MSVC)
	set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
	string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
else()
	set(CMAKE_C_FLAGS_DEBUG "")
	set(CMAKE_CXX_FLAGS_DEBUG "")

	# LTO objects in a static lib must be archived with gcc-ar/llvm-ar.
	# Plain ar/ranlib drop the symbol index ("plugin needed to handle lto object"),
	# which then shows up as undefined openmpt:: references at DLL link time.
	get_filename_component(_compiler_dir "${CMAKE_CXX_COMPILER}" DIRECTORY)
	if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
		set(_ar_names gcc-ar gcc-ar.exe)
		set(_ranlib_names gcc-ranlib gcc-ranlib.exe)
		set(_nm_names gcc-nm gcc-nm.exe)
	elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
		set(_ar_names llvm-ar llvm-ar.exe gcc-ar)
		set(_ranlib_names llvm-ranlib llvm-ranlib.exe gcc-ranlib)
		set(_nm_names llvm-nm llvm-nm.exe gcc-nm)
	endif()
	if(DEFINED _ar_names)
		if(NOT CMAKE_CXX_COMPILER_AR)
			find_program(CMAKE_CXX_COMPILER_AR NAMES ${_ar_names} HINTS "${_compiler_dir}")
		endif()
		if(NOT CMAKE_CXX_COMPILER_RANLIB)
			find_program(CMAKE_CXX_COMPILER_RANLIB NAMES ${_ranlib_names} HINTS "${_compiler_dir}")
		endif()
		if(NOT CMAKE_CXX_COMPILER_NM)
			find_program(CMAKE_CXX_COMPILER_NM NAMES ${_nm_names} HINTS "${_compiler_dir}")
		endif()
		if(CMAKE_CXX_COMPILER_AR)
			set(CMAKE_AR "${CMAKE_CXX_COMPILER_AR}" CACHE FILEPATH "LTO-aware archiver" FORCE)
		endif()
		if(CMAKE_CXX_COMPILER_RANLIB)
			set(CMAKE_RANLIB "${CMAKE_CXX_COMPILER_RANLIB}" CACHE FILEPATH "LTO-aware ranlib" FORCE)
		endif()
		if(CMAKE_CXX_COMPILER_NM)
			set(CMAKE_NM "${CMAKE_CXX_COMPILER_NM}" CACHE FILEPATH "LTO-aware nm" FORCE)
		endif()
		if(CMAKE_AR MATCHES "gcc-ar|llvm-ar")
			message(STATUS "LTO archiver: ${CMAKE_AR}")
		else()
			# Slim LTO objects cannot be indexed by plain ar; keep a regular object too.
			set(GODOT_OPENMPT_FAT_LTO ON)
			message(WARNING "LTO-aware archiver not found (using ${CMAKE_AR}); enabling -ffat-lto-objects")
		endif()
		unset(_ar_names)
		unset(_ranlib_names)
		unset(_nm_names)
	endif()
	unset(_compiler_dir)
endif()

string(TOLOWER "${CMAKE_SYSTEM_NAME}" GODOT_SYSTEM_NAME)
if(ANDROID)
	# Match godot-cpp: android.<abi> and no bits suffix.
	set(GODOT_SYSTEM_NAME "${GODOT_SYSTEM_NAME}.${ANDROID_ABI}")
	set(GODOT_BITS_SUFFIX "")
elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
	set(GODOT_BITS_SUFFIX ".64")
else()
	set(GODOT_BITS_SUFFIX ".32")
endif()

set(GODOT_CONFIG_SUFFIX "$<IF:$<CONFIG:Debug>,debug,release>")
set(GODOT_CPP_LIB_NAME "godot-cpp.${GODOT_SYSTEM_NAME}.${GODOT_CONFIG_SUFFIX}${GODOT_BITS_SUFFIX}")
set(GDEXTENSION_OUTPUT_NAME "libgdmpt-${GODOT_SYSTEM_NAME}.${GODOT_CONFIG_SUFFIX}${GODOT_BITS_SUFFIX}")

file(GLOB LIBOPENMPT_SOURCES CONFIGURE_DEPENDS
	"${CMAKE_CURRENT_SOURCE_DIR}/openmpt/libopenmpt/*.cpp"
)
file(GLOB_RECURSE OPENMPT_SOURCES CONFIGURE_DEPENDS
	"${CMAKE_CURRENT_SOURCE_DIR}/openmpt/sounddsp/*.cpp"
	"${CMAKE_CURRENT_SOURCE_DIR}/openmpt/soundlib/*.cpp"
	"${CMAKE_CURRENT_SOURCE_DIR}/openmpt/common/*.cpp"
)

add_library(libopenmpt STATIC
	${LIBOPENMPT_SOURCES}
	${OPENMPT_SOURCES}
)

target_compile_definitions(libopenmpt PRIVATE
	LIBOPENMPT_BUILD
	$<$<CONFIG:Debug>:MPT_BUILD_DEBUG>
)

target_include_directories(libopenmpt SYSTEM
	PUBLIC
		"${CMAKE_CURRENT_SOURCE_DIR}/openmpt"
	PRIVATE
		"${CMAKE_CURRENT_SOURCE_DIR}/openmpt/libopenmpt"
		"${CMAKE_CURRENT_SOURCE_DIR}/openmpt/common"
		"${CMAKE_CURRENT_SOURCE_DIR}/openmpt/sounddsp"
		"${CMAKE_CURRENT_SOURCE_DIR}/openmpt/soundlib"
		"${CMAKE_CURRENT_SOURCE_DIR}/openmpt/src"
)

add_library(${PROJECT_NAME} SHARED
	src/register_types.cpp
	src/audio_stream_mpt.cpp
	src/openmpt_singleton.cpp
)

target_include_directories(${PROJECT_NAME} SYSTEM PRIVATE
	"${CPP_BINDINGS_PATH}/include"
	"${GODOT_GEN_INCLUDE_PATH}"
	"${GODOT_GDEXTENSION_DIR}"
)

set_target_properties(${PROJECT_NAME} PROPERTIES
	OUTPUT_NAME "${GDEXTENSION_OUTPUT_NAME}"
	PREFIX ""
)

function(godot_openmpt_configure_target target)
	target_compile_features(${target} PRIVATE cxx_std_17)
	set_target_properties(${target} PROPERTIES
		CXX_EXTENSIONS OFF
		POSITION_INDEPENDENT_CODE ON
		CXX_VISIBILITY_PRESET hidden
		VISIBILITY_INLINES_HIDDEN ON
	)

	if(MSVC)
		target_compile_options(${target} PRIVATE
			/WX
			/EHsc
			/wd4244
			/wd4305
			/wd4101
			/wd4018
			/wd4267
			$<$<NOT:$<CONFIG:Debug>>:/O2>
		)
	else()
		target_compile_options(${target} PRIVATE
			-fPIC
			-flto
			$<$<BOOL:${GODOT_OPENMPT_FAT_LTO}>:-ffat-lto-objects>
			-ffunction-sections
			-fdata-sections
			-fexceptions
			-frtti
			-pthread
			$<$<NOT:$<CONFIG:Debug>>:-O3>
		)
		target_link_options(${target} PRIVATE
			-flto
			-Wl,--gc-sections
		)
		set_target_properties(${target} PROPERTIES INTERPROCEDURAL_OPTIMIZATION ON)
		# Android NDK uses libc++, not libgcc/libstdc++.
		if(NOT APPLE AND NOT ANDROID)
			target_link_options(${target} PRIVATE
				-static-libgcc
				-static-libstdc++
			)
		endif()
	endif()
endfunction()

godot_openmpt_configure_target(libopenmpt)
godot_openmpt_configure_target(${PROJECT_NAME})

if(MSVC)
	target_compile_definitions(libopenmpt PRIVATE
		MPT_BUILD_MSVC
		WIN32
		NOMINMAX
		_CRT_NONSTDC_NO_WARNINGS
		_CRT_SECURE_NO_WARNINGS
		_CRT_SECURE_CPP_OVERLOAD_STANDARD_NAMES=1
		_CRT_SECURE_CPP_OVERLOAD_STANDARD_NAMES_COUNT=1
		_WIN32_WINNT=0x0A00
		NTDDI_VERSION=0x0A00000A
	)
	target_compile_definitions(${PROJECT_NAME} PRIVATE TYPED_METHOD_BIND)
else()
	# Keep libopenmpt optimized and without debug symbols unless explicitly requested.
	if(LIBOPENMPT_DEBUG_SYMBOLS)
		target_compile_options(libopenmpt PRIVATE
			$<$<CONFIG:Debug>:-fno-omit-frame-pointer>
			$<$<CONFIG:Debug>:-O0>
			$<$<CONFIG:Debug>:-g>
		)
	else()
		target_compile_options(libopenmpt PRIVATE
			$<$<CONFIG:Debug>:-O3>
		)
	endif()

	target_compile_options(${PROJECT_NAME} PRIVATE
		$<$<CONFIG:Debug>:-fno-omit-frame-pointer>
		$<$<CONFIG:Debug>:-O0>
		$<$<CONFIG:Debug>:-g>
	)
	if(UNIX AND NOT APPLE AND NOT ANDROID)
		target_link_options(${PROJECT_NAME} PRIVATE "-Wl,-rpath,\$ORIGIN")
	endif()
endif()

target_link_directories(${PROJECT_NAME} PRIVATE "${GODOT_LIB_PATH}")
target_link_libraries(${PROJECT_NAME} PRIVATE
	"${GODOT_CPP_LIB_NAME}"
	libopenmpt
)

if(ANDROID AND CMAKE_STRIP)
	add_custom_command(
		TARGET ${PROJECT_NAME} POST_BUILD
		COMMAND "${CMAKE_STRIP}" --strip-debug "$<TARGET_FILE:${PROJECT_NAME}>"
		COMMENT "Stripping debug symbols from Android library"
	)
endif()

add_custom_command(
	TARGET ${PROJECT_NAME} POST_BUILD
	COMMAND ${CMAKE_COMMAND} -E make_directory "${ADDON_BIN_DIR}"
	COMMAND ${CMAKE_COMMAND} -E copy
		"$<TARGET_FILE:${PROJECT_NAME}>"
		"${ADDON_BIN_DIR}/$<TARGET_FILE_NAME:${PROJECT_NAME}>"
	COMMAND ${CMAKE_COMMAND} -E remove_directory "${EXAMPLE_DIR}/addons/godot-openmpt"
	COMMAND ${CMAKE_COMMAND} -E copy_directory
		"${ADDON_DIR}"
		"${EXAMPLE_DIR}/addons/godot-openmpt"
	COMMENT "Installing GDExtension into addons/ and example/"
)
