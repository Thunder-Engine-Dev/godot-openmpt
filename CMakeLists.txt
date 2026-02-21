# WARNING:
# This build system is already a mess... CMake is horrible.
# The problem is that I really don't want the binaries to be more that 100MB
# And apparently CMake insists on making the binary (through hidden compile flags)
# over 100MB. If I don't do anything at all that goes for both Debug AND Release.
# Hopefully this won't need much modification into the future.

cmake_minimum_required(VERSION 3.24 FATAL_ERROR)
project(godot-openmpt)

set(CMAKE_STATIC_LIBRARY_PREFIX "")
set(CMAKE_SHARED_LIBRARY_PREFIX "")

set(GODOT_GDEXTENSION_DIR godot-cpp/gdextension/ CACHE STRING "Path to GDExtension interface header directory")
set(CPP_BINDINGS_PATH godot-cpp/ CACHE STRING "Path to C++ bindings")
set(GODOT_LIB_PATH custom-godotcpp-build/bin CACHE STRING "Path to the built Godot C++ libraries")
set(GODOT_GEN_INCLUDE_PATH custom-godotcpp-build/gen/include CACHE STRING "Path to generated Godot includes")
set(EXAMPLE_DIR ${CMAKE_SOURCE_DIR}/example/ CACHE STRING "Path to example project")

# Change the output directory to the bin directory
set(BUILD_PATH ${CMAKE_SOURCE_DIR}/bin)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${BUILD_PATH}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${BUILD_PATH}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${BUILD_PATH}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG "${BUILD_PATH}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE "${BUILD_PATH}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_DEBUG "${BUILD_PATH}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_RELEASE "${BUILD_PATH}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG "${BUILD_PATH}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE "${BUILD_PATH}")

# Set the c++ standard to c++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(GODOT_COMPILE_FLAGS )
set(GODOT_LINKER_FLAGS )

# Get OpenMPT Sources
file(GLOB LIBOPENMPT_SOURCES openmpt/libopenmpt/*.cpp)
file(GLOB_RECURSE OPENMPT_SOURCES openmpt/sounddsp/*.cpp** openmpt/soundlib/*.cpp** openmpt/common/*.cpp**)

# Define the libopenmpt library
add_library(libopenmpt STATIC
	${LIBOPENMPT_SOURCES} ${OPENMPT_SOURCES})

if(CMAKE_BUILD_TYPE MATCHES Debug)
	target_compile_definitions(libopenmpt
		PRIVATE
			MPT_BUILD_DEBUG)
endif()

target_compile_definitions(libopenmpt
	PRIVATE
		LIBOPENMPT_BUILD)

target_include_directories(libopenmpt SYSTEM
	PUBLIC
		openmpt
	PRIVATE
		openmpt/libopenmpt
		openmpt/common
		openmpt/sounddsp
		openmpt/soundlib
		openmpt/src
)

# Define the Godot Extension library
add_library(${PROJECT_NAME} SHARED
	src/register_types.cpp
	src/audio_stream_mpt.cpp
    src/openmpt_singleton.cpp
)

target_include_directories(${PROJECT_NAME} SYSTEM
	PRIVATE
		${CPP_BINDINGS_PATH}/include
		${GODOT_GEN_INCLUDE_PATH}
		${GODOT_GDEXTENSION_DIR}
)

set(GODOT_LINKER_OPTIONS -flto -static-libgcc -static-libstdc++ "-Wl,--gc-sections,-rpath,$ORIGIN")
set(GODOT_COMPILE_OPTIONS -flto -O3 -fPIC -fvisibility=hidden -fvisibility-inlines-hidden -ffunction-sections -fdata-sections -fexceptions -frtti -pthread)

set(CMAKE_AR, "gcc-ar")
set(CMAKE_NM, "gcc-nm")
set(CMAKE_RANLIB "gcc-ranlib")

# Add the compile flags
set_property(TARGET ${PROJECT_NAME} APPEND PROPERTY COMPILE_OPTIONS ${GODOT_COMPILE_OPTIONS})
set_property(TARGET libopenmpt APPEND PROPERTY COMPILE_OPTIONS ${GODOT_COMPILE_OPTIONS})
set_property(TARGET ${PROJECT_NAME} APPEND PROPERTY LINK_OPTIONS ${GODOT_LINKER_OPTIONS})
set_property(TARGET libopenmpt APPEND PROPERTY LINK_OPTIONS ${GODOT_LINKER_OPTIONS})

# Create the correct name (godot.os.build_type.system_bits)
# Synchronized with godot-cpp's CMakeLists.txt

set(BITS .32)
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
	set(BITS .64)
endif(CMAKE_SIZEOF_VOID_P EQUAL 8)

if(CMAKE_BUILD_TYPE MATCHES Debug)
	set(GODOT_CPP_BUILD_TYPE Debug)
else()
	set(GODOT_CPP_BUILD_TYPE Release)
endif()

string(TOLOWER ${CMAKE_SYSTEM_NAME} SYSTEM_NAME)
string(TOLOWER ${GODOT_CPP_BUILD_TYPE} BUILD_TYPE)

if(ANDROID)
	# Added the android abi after system name
	set(SYSTEM_NAME ${SYSTEM_NAME}.${ANDROID_ABI})
	set(BITS "")
endif()

target_link_directories(${PROJECT_NAME}
	PRIVATE
		${GODOT_LIB_PATH}
)

target_link_libraries(${PROJECT_NAME}
	godot-cpp.${SYSTEM_NAME}.${BUILD_TYPE}${BITS}
	libopenmpt
)

set(MAIN_EXT_OUTPUT_NAME libgdmpt-${SYSTEM_NAME}.${BUILD_TYPE}${BITS})
set_property(TARGET ${PROJECT_NAME} PROPERTY OUTPUT_NAME ${MAIN_EXT_OUTPUT_NAME})

set(REAL_OUT ${BUILD_PATH}/${MAIN_EXT_OUTPUT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX})
if(ANDROID)
	set(REAL_OUT ${BUILD_PATH}/androed.so)
	add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
		COMMAND ${ANDROID_TOOLCHAIN_ROOT}/bin/llvm-strip --strip-debug ${BUILD_PATH}/${MAIN_EXT_OUTPUT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}
		COMMAND ${CMAKE_COMMAND} -E copy ${BUILD_PATH}/${MAIN_EXT_OUTPUT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX} ${REAL_OUT})
endif()

add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy ${REAL_OUT} ${CMAKE_SOURCE_DIR}/addons/godot-openmpt/bin/${MAIN_EXT_OUTPUT_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}
    COMMAND ${CMAKE_COMMAND} -E rm -rf ${EXAMPLE_DIR}/addons/godot-openmpt
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/addons/godot-openmpt ${EXAMPLE_DIR}/addons/godot-openmpt)
