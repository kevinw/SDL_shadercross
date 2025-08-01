# CMake Project for SDL3_shadercross - Simple DirectMedia Layer Shader Cross Compiler
# Written by @thatcosmonaut
cmake_minimum_required(VERSION 3.22)

# Version
set(MAJOR_VERSION 3)
set(MINOR_VERSION 0)
set(MICRO_VERSION 0)
set(SDL_REQUIRED_VERSION "3.1.3")

# option() honors normal variables.
set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)

project(SDL3_shadercross LANGUAGES C VERSION "${MAJOR_VERSION}.${MINOR_VERSION}.${MICRO_VERSION}")

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include(CMakeDependentOption)
include("${CMAKE_CURRENT_LIST_DIR}/cmake/GetGitRevisionDescription.cmake")
include("${CMAKE_CURRENT_LIST_DIR}/cmake/PrivateSdlFunctions.cmake")
include("${CMAKE_CURRENT_LIST_DIR}/cmake/sdlcpu.cmake")
include("${CMAKE_CURRENT_LIST_DIR}/cmake/sdlplatform.cmake")
include("${CMAKE_CURRENT_LIST_DIR}/cmake/sdlmanpages.cmake")

if(NOT TARGET SDL3::SDL3-static)
    find_package(SDL3 ${SDL_REQUIRED_VERSION})
endif()

if(BUILD_SHARED_LIBS)
	set(SDLSHADERCROSS_SHARED_DEFAULT ON)
	set(SDLSHADERCROSS_STATIC_DEFAULT OFF)
else()
	set(SDLSHADERCROSS_SHARED_DEFAULT OFF)
	set(SDLSHADERCROSS_STATIC_DEFAULT ON)
endif()

# Options
option(SDLSHADERCROSS_DXC "Enable HLSL compilation via DXC" ON)
option(SDLSHADERCROSS_SHARED "Build shared SDL_shadercross library" ${SDLSHADERCROSS_SHARED_DEFAULT})
option(SDLSHADERCROSS_STATIC "Build static SDL_shadercross library" ${SDLSHADERCROSS_STATIC_DEFAULT})
option(SDLSHADERCROSS_SPIRVCROSS_SHARED "Link to shared library variants of dependencies" ON)
option(SDLSHADERCROSS_VENDORED "Use vendored dependencies" OFF)
option(SDLSHADERCROSS_CLI "Build command line executable" ON)
cmake_dependent_option(SDLSHADERCROSS_CLI_STATIC "Link CLI with static libraries" OFF "SDLSHADERCROSS_CLI;SDLSHADERCROSS_STATIC;TARGET SDL3::SDL3-static" OFF)
cmake_dependent_option(SDLSHADERCROSS_CLI_LEAKCHECK "Check shadercross for memory leaks" OFF "SDLSHADERCROSS_CLI;TARGET SDL3::SDL3_test" OFF)
option(SDLSHADERCROSS_WERROR "Enable Werror" OFF)
option(SDLSHADERCROSS_INSTALL "Enable installation" OFF)
cmake_dependent_option(SDLSHADERCROSS_INSTALL_CPACK "Enable CPack installation" OFF "SDLSHADERCROSS_INSTALL" OFF)
cmake_dependent_option(SDLSHADERCROSS_INSTALL_MAN "Install man pages for SDL3_shadercross" OFF "SDLSHADERCROSS_INSTALL" OFF)
cmake_dependent_option(SDLSHADERCROSS_INSTALL_RUNTIME "Download, build and install runtime dependencies" OFF "SDLSHADERCROSS_INSTALL" OFF)

sdl_calculate_derived_version_variables(${MAJOR_VERSION} ${MINOR_VERSION} ${MICRO_VERSION})
SDL_DetectTargetCPUArchitectures(SDL_CPU_NAMES)
SDL_DetectCMakePlatform()

if(SDLSHADERCROSS_INSTALL)
	include(GNUInstallDirs)
endif()

if(NOT SDLSHADERCROSS_SHARED AND NOT SDLSHADERCROSS_STATIC)
	message(FATAL_ERROR "SDLSHADERCROSS_SHARED and SDLSHADERCROSS_STATIC cannot be both disabled")
endif()

# Platform Flags
if(APPLE)
	set(CMAKE_MACOSX_RPATH ON)
	set(CMAKE_OSX_DEPLOYMENT_TARGET 11.0)
endif()

if(NOT MSVC)
	add_compile_options(-pedantic) # -Wno-strict-aliasing
endif()

set(spirv_static_targets spirv-cross-c spirv-cross-glsl spirv-cross-hlsl spirv-cross-msl spirv-cross-cpp spirv-cross-reflect spirv-cross-core)

set(pc_requires )
set(vendored_targets )
if(SDLSHADERCROSS_VENDORED)
	set(SPIRV_CROSS_SKIP_INSTALL ON)
	set(SPIRV_CROSS_CLI OFF)
	set(SPIRV_CROSS_ENABLE_TESTS OFF)
	if(SDLSHADERCROSS_SPIRVCROSS_SHARED)
		set(SPIRV_CROSS_SHARED ON)
		set(SPIRV_CROSS_STATIC OFF)
	else()
		set(SPIRV_CROSS_SHARED OFF)
		set(SPIRV_CROSS_STATIC ON)
	endif()

	sdl_check_project_in_subfolder(external/SPIRV-Cross SPIRV-Cross SDLSHADERCROSS_VENDORED)
	set(SPIRV_CROSS_ENABLE_TESTS ON)
	set(CMAKE_POSITION_INDEPENDENT_CODE ON)
	add_subdirectory(external/SPIRV-Cross EXCLUDE_FROM_ALL)

	if(SDLSHADERCROSS_SPIRVCROSS_SHARED)
		list(APPEND vendored_targets spirv-cross-c-shared)
		list(APPEND pc_requires "spirv-cross-c-shared")
	else()
		enable_language(CXX)
		if(SDLSHADERCROSS_STATIC)
			foreach(extra IN LISTS spirv_static_targets)
				if(TARGET ${extra})
					list(APPEND vendored_targets ${extra})
					list(APPEND pc_requires "${extra}")
				endif()
			endforeach()
		endif()
	endif()

	sdl_check_project_in_subfolder(external/SPIRV-Headers SPIRV-Headers SDLSHADERCROSS_VENDORED)
	add_subdirectory(external/SPIRV-Headers EXCLUDE_FROM_ALL)
	sdl_check_project_in_subfolder(external/SPIRV-Tools SPIRV-Tools SDLSHADERCROSS_VENDORED)
	add_subdirectory(external/SPIRV-Tools EXCLUDE_FROM_ALL)

	sdl_check_project_in_subfolder(external/DirectXShaderCompiler DirectXShaderCompiler SDLSHADERCROSS_VENDORED)
	if(MINGW)
		set(CMAKE_C_COMPILER_ARCHITECTURE_ID "${SDL_CPU_NAMES}")
	endif()

	if(SDLSHADERCROSS_DXC)
		# LLVM dependencies of dxc does not support building as a shared library
		set(BUILD_SHARED_LIBS OFF)
		# Enabling HLSL_ENABLE_DEBUG_ITERATORS blocks DirectXShaderCompiler from adding /D_ITERATOR_DEBUG_LEVEL=0
		set(HLSL_ENABLE_DEBUG_ITERATORS ON)
		set(DXC_COVERAGE OFF)
		set(HLSL_INCLUDE_TESTS OFF)
		set(LLVM_INCLUDE_TESTS OFF)
		set(HLSL_DISABLE_SOURCE_GENERATION TRUE)
		include(external/DirectXShaderCompiler/cmake/caches/PredefinedParams.cmake)
		add_subdirectory(external/DirectXShaderCompiler EXCLUDE_FROM_ALL)
		add_library(DirectXShaderCompiler::dxcompiler ALIAS dxcompiler)
		add_library(DirectXShaderCompiler::dxil ALIAS dxildll)
		list(APPEND vendored_targets dxcompiler dxildll)
		add_dependencies(dxcompiler dxildll)
	endif()

	export(TARGETS ${vendored_targets} NAMESPACE "SDL3_shadercross::" FILE "SDL3_shadercross-vendored-targets.cmake")
else()
	if(SDLSHADERCROSS_SPIRVCROSS_SHARED)
		if(NOT TARGET spirv_cross_c_shared)
			find_package(spirv_cross_c_shared REQUIRED)
		endif()
	else()
		enable_language(CXX)
		if(NOT TARGET spirv_cross_c)
			find_package(spirv_cross_core QUIET)
			find_package(spirv_cross_glsl QUIET)
			find_package(spirv_cross_hlsl QUIET)
			find_package(spirv_cross_msl QUIET)
			find_package(spirv_cross_cpp QUIET)
			find_package(spirv_cross_reflect QUIET)
			find_package(spirv_cross_c REQUIRED)
		endif()
	endif()

	if(SDLSHADERCROSS_DXC)
		set(DirectXShaderCompiler_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/external/DirectXShaderCompiler-binaries")
		find_package(DirectXShaderCompiler REQUIRED)
	endif()
endif()

# Source lists
set(SOURCE_FILES
	# Public Headers
	include/SDL3_shadercross/SDL_shadercross.h
	# Source Files
	src/SDL_shadercross.c
)

set(SDL3_shadercross_targets)
if(SDLSHADERCROSS_SHARED)
	list(APPEND SDL3_shadercross_targets SDL3_shadercross-shared)
	if(NOT TARGET SDL3::Headers OR NOT TARGET SDL3::SDL3-shared)
		find_package(SDL3 ${SDL_REQUIRED_VERSION} REQUIRED COMPONENTS Headers SDL3-shared)
	endif()

	add_library(SDL3_shadercross-shared SHARED ${SOURCE_FILES})
	add_library(SDL3_shadercross::SDL3_shadercross ALIAS SDL3_shadercross-shared)

	set_property(TARGET SDL3_shadercross-shared PROPERTY DEFINE_SYMBOL DLL_EXPORT)
	sdl_target_link_option_version_file(SDL3_shadercross-shared "${CMAKE_CURRENT_SOURCE_DIR}/src/SDL_shadercross.sym")
	sdl_target_link_options_no_undefined(SDL3_shadercross-shared)

	# Build flags
	if(WIN32)
		target_sources(SDL3_shadercross-shared PRIVATE "src/version.rc")
		set_property(TARGET SDL3_shadercross-shared PROPERTY PREFIX "")
	endif()

	# SDL3_shadercross folders as includes, for other targets to consume
	target_include_directories(SDL3_shadercross-shared PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>")
	target_include_directories(SDL3_shadercross-shared PUBLIC "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")

	# Soname
	set_target_properties(SDL3_shadercross-shared PROPERTIES
		OUTPUT_NAME "SDL3_shadercross"
		SOVERSION "${SO_VERSION_MAJOR}"
		VERSION "${SO_VERSION}"
	)

	target_link_libraries(SDL3_shadercross-shared PRIVATE
		SDL3::SDL3-shared
	)
	export(TARGETS SDL3_shadercross-shared NAMESPACE "SDL3_shadercross::" FILE "SDL3_shadercross-shared-targets.cmake")
endif()


if(SDLSHADERCROSS_STATIC)
	list(APPEND SDL3_shadercross_targets SDL3_shadercross-static)
	if(NOT TARGET SDL3::Headers)
		find_package(SDL3 ${SDL_REQUIRED_VERSION} REQUIRED COMPONENTS Headers)
	endif()

	add_library(SDL3_shadercross-static STATIC ${SOURCE_FILES})
	add_library(SDL3_shadercross::SDL3_shadercross-static ALIAS SDL3_shadercross-static)

	if(NOT MSVC)
		set_property(TARGET SDL3_shadercross-static PROPERTY OUTPUT_NAME "SDL3_shadercross")
	endif()

	# SDL3_shadercross folders as includes, for other targets to consume
	target_include_directories(SDL3_shadercross-static PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>")
	target_include_directories(SDL3_shadercross-static PUBLIC "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")

	target_link_libraries(SDL3_shadercross-static PUBLIC
		SDL3::Headers
	)
	export(TARGETS SDL3_shadercross-static NAMESPACE "SDL3_shadercross::" FILE "SDL3_shadercross-static-targets.cmake")
endif()

# Copy dependencies of SDL3_shadercross-config.cmake to build directory.
# This allows usage of the package without installation.
file(COPY
	"${CMAKE_CURRENT_SOURCE_DIR}/cmake/sdlcpu.cmake"
	"${CMAKE_CURRENT_SOURCE_DIR}/cmake/FindDirectXShaderCompiler.cmake"
	DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")

foreach(target IN LISTS SDL3_shadercross_targets)
	sdl_add_warning_options(${target} WARNING_AS_ERROR ${SDLSHADERCROSS_WERROR})
	target_compile_features(${target} PRIVATE c_std_99)

	if(SDLSHADERCROSS_DXC)
		add_compile_definitions(SDL_SHADERCROSS_DXC)
	endif()

	if(SDLSHADERCROSS_SPIRVCROSS_SHARED)
		target_link_libraries(${target} PRIVATE spirv-cross-c-shared)
	else()
		target_link_libraries(${target} PRIVATE spirv-cross-c)
	endif()
	if(SDLSHADERCROSS_DXC)
		target_link_libraries(${target} PRIVATE DirectXShaderCompiler::dxcompiler)
	endif()
	if(NOT SDLSHADERCROSS_SPIRVCROSS_SHARED)
		# spirv-cross uses C++
		set_property(TARGET ${target} PROPERTY LINKER_LANGUAGE CXX)
	endif()
endforeach()

if(NOT TARGET SDL3_shadercross::SDL3_shadercross)
	if(TARGET SDL3_shadercross-shared)
		add_library(SDL3_shadercross::SDL3_shadercross ALIAS SDL3_shadercross-shared)
	else()
		add_library(SDL3_shadercross::SDL3_shadercross ALIAS SDL3_shadercross-static)
	endif()
endif()

if(SDLSHADERCROSS_CLI)
	add_executable(shadercross src/cli.c)
	sdl_add_warning_options(shadercross WARNING_AS_ERROR ${SDLSHADERCROSS_WERROR})
	sdl_target_link_options_no_undefined(shadercross)

	if(SDLSHADERCROSS_CLI_LEAKCHECK)
		target_link_libraries(shadercross PRIVATE SDL3::SDL3_test)
		target_compile_definitions(shadercross PRIVATE LEAKCHECK)
	endif()
	if(SDLSHADERCROSS_CLI_STATIC)
		target_link_libraries(shadercross PRIVATE SDL3_shadercross::SDL3_shadercross-static)
		target_link_libraries(shadercross PRIVATE SDL3::SDL3-static)
	else()
		if(NOT TARGET SDL3::SDL3)
			find_package(SDL3 ${SDL_REQUIRED_VERSION} REQUIRED CONFIG COMPONENTS SDL3)
		endif()
		target_link_libraries(shadercross PRIVATE SDL3_shadercross::SDL3_shadercross)
		target_link_libraries(shadercross PRIVATE SDL3::SDL3)
	endif()
endif()

if(SDLSHADERCROSS_INSTALL)
	if(WIN32 AND NOT MINGW)
		set(INSTALL_CMAKEDIR_ROOT_DEFAULT "cmake")
	else()
		set(INSTALL_CMAKEDIR_ROOT_DEFAULT "${CMAKE_INSTALL_LIBDIR}/cmake")
	endif()
	set(SDLSHADERCROSS_INSTALL_CMAKEDIR_ROOT "${INSTALL_CMAKEDIR_ROOT_DEFAULT}" CACHE STRING "Root folder where to install SDL3_shadercross cmake related files (SDL3_shadercross subfolder for MSVC projects)")
	set(SDLSHADERCROSS_PKGCONFIG_INSTALLDIR "${CMAKE_INSTALL_LIBDIR}/pkgconfig")

	if(WIN32 AND NOT MINGW)
		set(SDLSHADERCROSS_INSTALL_CMAKEDIR "${SDLSHADERCROSS_INSTALL_CMAKEDIR_ROOT}")
	else()
		set(SDLSHADERCROSS_INSTALL_CMAKEDIR "${SDLSHADERCROSS_INSTALL_CMAKEDIR_ROOT}/SDL3_shadercross")
	endif()

	if(TARGET SDL3_shadercross-shared)
		install(TARGETS SDL3_shadercross-shared EXPORT SDL3_shadercross-shared-export
			ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT devel
			LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT library
			RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT library
		)
		install(EXPORT SDL3_shadercross-shared-export
			FILE SDL3_shadercross-shared-targets.cmake
			NAMESPACE SDL3_shadercross::
			DESTINATION "${SDLSHADERCROSS_INSTALL_CMAKEDIR}"
			COMPONENT devel
		)
	endif()
	if(TARGET SDL3_shadercross-static)
		install(TARGETS SDL3_shadercross-static EXPORT SDL3_shadercross-static-export
			ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT devel
			LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT library
		)
		install(EXPORT SDL3_shadercross-static-export
			FILE SDL3_shadercross-static-targets.cmake
			NAMESPACE SDL3_shadercross::
			DESTINATION "${SDLSHADERCROSS_INSTALL_CMAKEDIR}"
			COMPONENT devel
		)
	endif()
	if(vendored_targets)
		install(TARGETS ${vendored_targets} EXPORT SDL3_shadercross-vendored
			ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT devel
			RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT library
			LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" COMPONENT library
		)
		install(EXPORT SDL3_shadercross-vendored
			FILE SDL3_shadercross-vendored-targets.cmake
			NAMESPACE SDL3_shadercross::vendored::
			DESTINATION "${SDLSHADERCROSS_INSTALL_CMAKEDIR}"
			COMPONENT devel
		)
	endif()
	install(
		FILES "${CMAKE_CURRENT_SOURCE_DIR}/include/SDL3_shadercross/SDL_shadercross.h"
		DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/SDL3_shadercross" COMPONENT DEVEL
	)
	if(SDLSHADERCROSS_CLI)
		install(TARGETS shadercross RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
	endif()

	include(CMakePackageConfigHelpers)
	configure_package_config_file(cmake/SDL3_shadercrossConfig.cmake.in SDL3_shadercrossConfig.cmake
		NO_SET_AND_CHECK_MACRO
		INSTALL_DESTINATION "${SDLSHADERCROSS_INSTALL_CMAKEDIR}"
	)
	write_basic_package_version_file("${PROJECT_BINARY_DIR}/SDL3_shadercrossConfigVersion.cmake"
		COMPATIBILITY AnyNewerVersion
	)
	install(
		FILES
			"${CMAKE_CURRENT_BINARY_DIR}/SDL3_shadercrossConfig.cmake"
			"${CMAKE_CURRENT_BINARY_DIR}/SDL3_shadercrossConfigVersion.cmake"
		DESTINATION "${SDLSHADERCROSS_INSTALL_CMAKEDIR}"
		COMPONENT devel
	)
	if(NOT SDLSHADERCROSS_VENDORED)
		install(
			FILES
				"cmake/sdlcpu.cmake"
				"cmake/FindDirectXShaderCompiler.cmake"
			DESTINATION "${SDLSHADERCROSS_INSTALL_CMAKEDIR}"
			COMPONENT devel
		)
	endif()

    if(IS_ABSOLUTE "${CMAKE_INSTALL_INCLUDEDIR}")
        set(INCLUDEDIR_FOR_PKG_CONFIG "${CMAKE_INSTALL_INCLUDEDIR}")
    else()
        set(INCLUDEDIR_FOR_PKG_CONFIG "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")
    endif()
    if(IS_ABSOLUTE "${CMAKE_INSTALL_LIBDIR}")
        set(LIBDIR_FOR_PKG_CONFIG "${CMAKE_INSTALL_LIBDIR}")
    else()
        set(LIBDIR_FOR_PKG_CONFIG "\${prefix}/${CMAKE_INSTALL_LIBDIR}")
    endif()

	file(RELATIVE_PATH SDL_PATH_PREFIX_RELATIVE_TO_PKGCONFIG "${CMAKE_INSTALL_PREFIX}/${SDLSHADERCROSS_PKGCONFIG_INSTALLDIR}" "${CMAKE_INSTALL_PREFIX}")
	string(REGEX REPLACE "[/]+$" "" SDL_PATH_PREFIX_RELATIVE_TO_PKGCONFIG "${SDL_PATH_PREFIX_RELATIVE_TO_PKGCONFIG}")
	set(SDL_PKGCONFIG_PREFIX "\${pcfiledir}/${SDL_PATH_PREFIX_RELATIVE_TO_PKGCONFIG}")
	if(NOT SDLSHADERCROSS_VENDORED)
		if(SDLSHADERCROSS_SPIRVCROSS_SHARED)
			set(PC_REQUIRES "spirv-cross-c-shared")
		else()
			set(PC_REQUIRES "spirv-cross-c")
		endif()
	endif()
	set(PC_LIBS "-ldxcompiler")
	configure_file(cmake/sdl3-shadercross.pc.in sdl3-shadercross.pc @ONLY)

	# Always install sdl3-shadercross.pc file: libraries might be different between config modes
	install(FILES "${CMAKE_CURRENT_BINARY_DIR}/sdl3-shadercross.pc"
			DESTINATION "${SDLSHADERCROSS_PKGCONFIG_INSTALLDIR}" COMPONENT devel)

	install(FILES "LICENSE.txt"
		DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/licenses/${PROJECT_NAME}"
		COMPONENT library
	)
	if(SDLSHADERCROSS_INSTALL_CPACK)
		if(MSVC)
			set(CPACK_GENERATOR "ZIP")
		else()
			set(CPACK_GENERATOR "TGZ")
		endif()
		configure_file(cmake/CPackProjectConfig.cmake.in CPackProjectConfig.cmake @ONLY)
		set(CPACK_PROJECT_CONFIG_FILE "${PROJECT_BINARY_DIR}/CPackProjectConfig.cmake")
		# CPACK_SOURCE_PACKAGE_FILE_NAME must end with "-src" (so we can block creating a source archive)
		set(CPACK_SOURCE_PACKAGE_FILE_NAME "SDL3_shadercross-${PROJECT_VERSION}-src")
		set(CPACK_PACKAGE_DIRECTORY "${CMAKE_BINARY_DIR}/dist")
		include(CPack)
	endif()
endif()

if(SDLSHADERCROSS_INSTALL_MAN)
	sdl_get_git_revision_hash(SDLSHADERCROSS_REVISION)
	SDL_generate_manpages(
		HEADERS_DIR "${PROJECT_SOURCE_DIR}/include/SDL3_shadercross"
		SYMBOL "SDL_ShaderCross_Init"
		WIKIHEADERS_PL_PATH "${CMAKE_CURRENT_SOURCE_DIR}/build-scripts/wikiheaders.pl"
		REVISION "${SDLSHADERCROSS_REVISION}"
	)
endif()

if(SDLSHADERCROSS_INSTALL_RUNTIME)
	set(chmod_0755 OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)
	if(APPLE)
		set(rpath_origin "@executable_path")
	else()
		set(rpath_origin "\$ORIGIN")
	endif()

	if(NOT SDLSHADERCROSS_VENDORED)
		if(SDLSHADERCROSS_DXC)
			install(IMPORTED_RUNTIME_ARTIFACTS DirectXShaderCompiler::dxcompiler DirectXShaderCompiler::dxil RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}")
		endif()
		if(SDLSHADERCROSS_SPIRVCROSS_SHARED)
			install(IMPORTED_RUNTIME_ARTIFACTS spirv-cross-c-shared RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}")
			if(WIN32)
				install(CODE "
					set(CMAKE_GET_RUNTIME_DEPENDENCIES_PLATFORM \"windows+pe\")
					set(CMAKE_GET_RUNTIME_DEPENDENCIES_TOOL \"$<IF:$<BOOL:${MSVC}>,dumpbin,objdump>\")
					set(CMAKE_GET_RUNTIME_DEPENDENCIES_COMMAND \"${CMAKE_OBJDUMP}\")
					file(GET_RUNTIME_DEPENDENCIES
						RESOLVED_DEPENDENCIES_VAR resolved
						UNRESOLVED_DEPENDENCIES_VAR unresolved
						DIRECTORIES  \"$<TARGET_FILE_DIR:spirv-cross-c-shared>\" \"${CMAKE_INSTALL_PREFIX}${CMAKE_INSTALL_BINDIR}\"
						LIBRARIES \"$<TARGET_FILE:spirv-cross-c-shared>\"
						POST_EXCLUDE_REGEXES \".*[sS][yY][sS][tT][eE][mM]32.*\" \".*[aA][pP][iI]-[mM][sS].*\" \".*[aA][pP][iI]-[eE][xX][tT].*\"
					)
					file(INSTALL DESTINATION \"\${CMAKE_INSTALL_PREFIX}/bin\" TYPE SHARED_LIBRARY FILES \${resolved})
				")
			endif()
		endif()
	endif()

	if(TARGET shadercross)
		file(RELATIVE_PATH bin_to_lib "${CMAKE_INSTALL_FULL_BINDIR}" "${CMAKE_INSTALL_FULL_LIBDIR}")
		set_property(TARGET shadercross PROPERTY INSTALL_RPATH "${rpath_origin}/${bin_to_lib}")
	endif()

	# Install SDL3
	if(SDLSHADERCROSS_CLI AND NOT SDLSHADERCROSS_CLI_STATIC)
		install(IMPORTED_RUNTIME_ARTIFACTS SDL3::SDL3-shared RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}")
	endif()
	include(ExternalProject)

	if(NOT WIN32)
		set(configure_CFLAGS "${CMAKE_C_FLAGS}")
		if(APPLE)
			foreach(osx_arch IN LISTS CMAKE_OSX_ARCHITECTURES)
				string(APPEND configure_CFLAGS " -arch ${osx_arch}")
			endforeach()
		endif()
		string(REPLACE ";" "$<SEMICOLON>" genex_CMAKE_OSX_ARCHITECTURES "${CMAKE_OSX_ARCHITECTURES}")

		ExternalProject_Add(spirv_headers
			GIT_REPOSITORY 		"https://github.com/KhronosGroup/SPIRV-Headers.git"
			GIT_TAG 			"main"
			CMAKE_ARGS 			"-DCMAKE_BUILD_TYPE=Release" "-DCMAKE_INSTALL_LIBDIR=lib" "-DCMAKE_INSTALL_BINDIR=bin" "-DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>" "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}" "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}" "-DCMAKE_OSX_ARCHITECTURES=${genex_CMAKE_OSX_ARCHITECTURES}" "-DCMAKE_OSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET}"
			BUILD_COMMAND 		"${CMAKE_COMMAND}" "--build" "<BINARY_DIR>" --config "Release"
			INSTALL_COMMAND 	"${CMAKE_COMMAND}" "--install" "<BINARY_DIR>" --config "Release"
		)
		ExternalProject_Get_property(spirv_headers INSTALL_DIR)
		set(spirv_headers_install_dir "${INSTALL_DIR}")

		ExternalProject_Add(vulkan_headers
			GIT_REPOSITORY 		"https://github.com/KhronosGroup/Vulkan-Headers"
			GIT_TAG 			"main"
			CMAKE_ARGS 			"-DCMAKE_BUILD_TYPE=Release" "-DCMAKE_INSTALL_LIBDIR=lib" "-DCMAKE_INSTALL_BINDIR=bin" "-DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>" "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}" "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}" "-DCMAKE_OSX_ARCHITECTURES=${genex_CMAKE_OSX_ARCHITECTURES}" "-DCMAKE_OSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET}"
			BUILD_COMMAND 		"${CMAKE_COMMAND}" "--build" "<BINARY_DIR>" --config "Release"
			INSTALL_COMMAND 	"${CMAKE_COMMAND}" "--install" "<BINARY_DIR>" --config "Release"
		)
		ExternalProject_Get_property(vulkan_headers INSTALL_DIR)
		set(vulkan_headers_install_dir "${INSTALL_DIR}")

		ExternalProject_Add(vulkan_loader
			DEPENDS 			spirv_headers vulkan_headers
			GIT_REPOSITORY 		"https://github.com/KhronosGroup/Vulkan-Loader.git"
			GIT_TAG 			"main"
			CMAKE_ARGS 			"-DCMAKE_BUILD_TYPE=Release" "-DCMAKE_INSTALL_LIBDIR=lib" "-DCMAKE_INSTALL_BINDIR=bin" "-DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>" "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}" "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}" "-DCMAKE_OSX_ARCHITECTURES=${genex_CMAKE_OSX_ARCHITECTURES}" "-DCMAKE_OSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET}" "-DCMAKE_PREFIX_PATH=${vulkan_headers_install_dir}"
			BUILD_COMMAND 		"${CMAKE_COMMAND}" "--build" "<BINARY_DIR>" --config "Release"
			INSTALL_COMMAND 	"${CMAKE_COMMAND}" "--install" "<BINARY_DIR>" --config "Release"
		)
		ExternalProject_Get_property(vulkan_loader INSTALL_DIR)
		set(vulkan_loader_install_dir "${INSTALL_DIR}")

		find_package(BISON REQUIRED)
		ExternalProject_Add(vkd3d
			DEPENDS 			spirv_headers vulkan_headers vulkan_loader
			URL					"https://dl.winehq.org/vkd3d/source/vkd3d-1.16.tar.xz"
			URL_HASH			"SHA256=b4aef432ba1b533465b959d0e1bd22fac4610b7e178c95b3ff153854ef0f4136"
			DOWNLOAD_EXTRACT_TIMESTAMP "1"
			CONFIGURE_COMMAND 	"sh" "<SOURCE_DIR>/configure" "--prefix=<INSTALL_DIR>" "--enable-tests=no" "--enable-demos=no" "--disable-doxygen-doc" "CFLAGS=-I${spirv_headers_install_dir}/include -I${vulkan_headers_install_dir}/include -I${vulkan_loader_install_dir}/include ${configure_CFLAGS}" "LDFLAGS=-L${vulkan_loader_install_dir}/lib" "BISON=${BISON_EXECUTABLE}"
			BUILD_COMMAND	 	"make"
			INSTALL_COMMAND	 	"make" "install"
		)
		ExternalProject_Get_property(vkd3d SOURCE_DIR)
		ExternalProject_Get_property(vkd3d INSTALL_DIR)
		install(DIRECTORY "${INSTALL_DIR}/lib/" DESTINATION "${CMAKE_INSTALL_LIBDIR}" FILES_MATCHING PATTERN "*.so*" PERMISSIONS ${chmod_0755})
		install(DIRECTORY "${INSTALL_DIR}/lib/" DESTINATION "${CMAKE_INSTALL_LIBDIR}" FILES_MATCHING PATTERN "*.dylib*" PERMISSIONS ${chmod_0755})
		install(FILES "${SOURCE_DIR}/COPYING" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/licenses/vkd3d")
	endif()
	if(LINUX)
		find_program(PATCHELF_BIN NAMES "patchelf" REQUIRED)
		install(CODE "file(GLOB so_paths \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}/*so*\")\n  foreach(so_path \${so_paths})\n    if(NOT IS_SYMLINK \${so_path})\n      message(STATUS \"Adding \\\"\$ORIGIN\\\" to RPATH of \${so_path}\")\n      execute_process(COMMAND ${PATCHELF_BIN} \"\${so_path}\" --add-rpath \"\$ORIGIN\")\n    endif()\n  endforeach()")
	elseif(APPLE)
		find_program(INT_BIN NAMES "install_name_tool" REQUIRED)
		install(CODE "file(GLOB so_paths \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}/*dylib*\")\n  foreach(so_path \${so_paths})\n    if(NOT IS_SYMLINK \${so_path})\n      message(STATUS \"Adding \\\"\@loader_path\\\" to RPATH of \${so_path}\")\n      execute_process(COMMAND ${INT_BIN} \"\${so_path}\" -add_rpath \"\@loader_path\")\n    endif()\n  endforeach()")
	endif()
endif()
