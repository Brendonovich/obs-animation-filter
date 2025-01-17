cmake_minimum_required(VERSION 3.16...3.20)

# Version variables
project(obs-animation VERSION 0.0.1)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Prohibit in-source builds
file(TO_CMAKE_PATH "${PROJECT_BINARY_DIR}/CMakeLists.txt" _LOC_PATH)
if(EXISTS "${LOC_PATH}")
	message(FATAL_ERROR "obs-animation: You cannot build in a source directory (or any directory with "
						"CMakeLists.txt file). Please make a build subdirectory. Feel free to "
						"remove CMakeCache.txt and CMakeFiles.")
endif()
unset(_LOC_PATH)


# Allow selection of common build types via UI
if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING
		"OBS build type [Release, RelWithDebInfo, Debug, MinSizeRel]" FORCE)
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS Release RelWithDebInfo Debug MinSizeRel)
endif()


# Plugin tests flag
option(PLUGIN_TESTS "Enable plugin runtime tests" OFF)


# Qt build stuff
set(CMAKE_PREFIX_PATH "${QTDIR}")
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)

set(CMAKE_AUTORCC ON) # For resources.qrc


# Arm build fixes
if (${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm")
	set(CMAKE_CXX_FLAGS "-mfpu=neon")
endif()


# Find libobs
if (WIN32 OR APPLE)
	include(cmake/FindLibObs.cmake)
endif()
find_package(LibObs REQUIRED)


# Find Qt5
find_package(Qt5 REQUIRED COMPONENTS Core Widgets)


# Find nlohmann
set(JSON_BuildTests OFF CACHE INTERNAL "")


# Configure files
configure_file(
	src/plugin-macros.h.in
	../src/plugin-macros.generated.h
)
configure_file(
	installer/installer-windows.iss.in
	../installer/installer-windows.generated.iss
)


# Inlude sources
set(obs-animation_SOURCES
	src/obs-animation.cpp
	src/State.cpp
	src/Animation.cpp
	src/AnimationTarget.cpp
	src/OBSSceneItem.cpp
	src/Keyframe.cpp)

set(obs-animation_HEADERS
	vendor/obs-websocket-api.h
	src/State.h
	src/Animation.h
	src/AnimationTarget.h
	src/OBSSceneItem.h
	src/OBSSource.h
	src/Keyframe.h
	src/types.h)


# Platform-independent build settings
add_library(obs-animation MODULE
	${obs-animation_SOURCES}
	${obs-animation_HEADERS})

include_directories(
	"${LIBOBS_INCLUDE_DIR}/../UI/obs-frontend-api"
	${Qt5Core_INCLUDES}
	${Qt5Widgets_INCLUDES})

target_link_libraries(obs-animation
	libobs
	Qt5::Core
	Qt5::Widgets)

if(PLUGIN_TESTS)
  target_compile_definitions(obs-animation PRIVATE PLUGIN_TESTS)
endif()


# Windows-specific build settings and tasks
if(WIN32)
	if(NOT DEFINED OBS_FRONTEND_LIB)
		set(OBS_FRONTEND_LIB "OBS_FRONTEND_LIB-NOTFOUND" CACHE FILEPATH "OBS frontend library")
		message(FATAL_ERROR "Could not find OBS Frontend API's library!")
	endif()

	if(MSVC)
		# Enable Multicore Builds and disable FH4 (to not depend on VCRUNTIME140_1.DLL)
		add_definitions(/MP /d2FH4-)
	endif()

	# add_definitions(-D_WEBSOCKETPP_CPP11_STL_)

	if(CMAKE_SIZEOF_VOID_P EQUAL 8)
		set(ARCH_NAME "64bit")
		set(OBS_BUILDDIR_ARCH "build64")
	else()
		set(ARCH_NAME "32bit")
		set(OBS_BUILDDIR_ARCH "build32")
	endif()

	include_directories(
		"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/UI"
	)

	target_link_libraries(obs-animation
		"${OBS_FRONTEND_LIB}")

	# Release package helper
	# The "release" folder has a structure similar OBS' one on Windows
	set(RELEASE_DIR "${PROJECT_SOURCE_DIR}/release")

	add_custom_command(TARGET obs-animation POST_BUILD
		# If config is Release or RelWithDebInfo, package release files
		COMMAND if $<OR:$<CONFIG:Release>,$<CONFIG:RelWithDebInfo>>==1 (
			"${CMAKE_COMMAND}" -E make_directory
			"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}"
		)

		COMMAND if $<OR:$<CONFIG:Release>,$<CONFIG:RelWithDebInfo>>==1 (
			"${CMAKE_COMMAND}" -E copy_directory
			"${PROJECT_SOURCE_DIR}/data"
			"${RELEASE_DIR}/data/obs-plugins/obs-animation"
		)

		COMMAND if $<OR:$<CONFIG:Release>,$<CONFIG:RelWithDebInfo>>==1 (
			"${CMAKE_COMMAND}" -E copy
			"$<TARGET_FILE:obs-animation>"
			"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}"
		)

		# In Release or RelWithDebInfo mode, copy Qt image format plugins
		COMMAND if $<OR:$<CONFIG:Release>,$<CONFIG:RelWithDebInfo>>==1 (
			"${CMAKE_COMMAND}" -E make_directory
			"${RELEASE_DIR}/bin/${ARCH_NAME}/imageformats"
		)
		# COMMAND if $<OR:$<CONFIG:Release>,$<CONFIG:RelWithDebInfo>>==1 (
		# 	"${CMAKE_COMMAND}" -E copy
		# 	"${QTDIR}/plugins/imageformats/qicns.dll"
		# 	"${QTDIR}/plugins/imageformats/qico.dll"
		# 	"${QTDIR}/plugins/imageformats/qjpeg.dll"
		# 	"${QTDIR}/plugins/imageformats/qtiff.dll"
		# 	"${QTDIR}/plugins/imageformats/qwbmp.dll"
		# 	"${QTDIR}/plugins/imageformats/qwebp.dll"
		# 	"${RELEASE_DIR}/bin/${ARCH_NAME}/imageformats"
		# )

		# If config is RelWithDebInfo, package PDB file for target
		COMMAND if $<CONFIG:RelWithDebInfo>==1 (
			"${CMAKE_COMMAND}" -E copy
			"$<TARGET_PDB_FILE:obs-animation>"
			"${RELEASE_DIR}/obs-plugins/${ARCH_NAME}"
		)

		# In the Debug configuration, copy to obs-studio dev environment for immediate testing
		COMMAND if $<CONFIG:Debug>==1 (
			"${CMAKE_COMMAND}" -E copy
				"$<TARGET_FILE:obs-animation>"
				"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/obs-plugins/${ARCH_NAME}"
		)

		COMMAND if $<CONFIG:Debug>==1 (
			"${CMAKE_COMMAND}" -E copy
			"$<TARGET_PDB_FILE:obs-animation>"
			"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/obs-plugins/${ARCH_NAME}"
		)

		COMMAND if $<CONFIG:Debug>==1 (
			"${CMAKE_COMMAND}" -E make_directory
			"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/data/obs-plugins/obs-animation"
		)

		COMMAND if $<CONFIG:Debug>==1 (
			"${CMAKE_COMMAND}" -E copy_directory
			"${PROJECT_SOURCE_DIR}/data"
			"${LIBOBS_INCLUDE_DIR}/../${OBS_BUILDDIR_ARCH}/rundir/$<CONFIG>/data/obs-plugins/obs-animation"
		)
	)
endif()


# Linux-specific build settings and tasks
if(UNIX AND NOT APPLE)
	include(GNUInstallDirs)

	target_compile_options(obs-animation PRIVATE -Wall -Wextra -Wpedantic -Werror -Wno-missing-field-initializers)

	set_target_properties(obs-animation PROPERTIES PREFIX "")
	target_link_libraries(obs-animation obs-frontend-api)

	# Manually set permissions for locales
	file(GLOB locale_files data/locale/*.ini)
		set(CMAKE_INSTALL_DEFAULT_DIRECTORY_PERMISSIONS
		OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_WRITE GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

	# Manually set file permissions for binary
	install(TARGETS obs-animation LIBRARY
		DESTINATION "${CMAKE_INSTALL_FULL_LIBDIR}/obs-plugins"
		PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ)

	# OBS on Ubuntu installs into a different directory than most linux distros
	if(${USE_UBUNTU_FIX})
		install(TARGETS obs-animation LIBRARY
			DESTINATION "/usr/lib/obs-plugins"
			PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ)
	endif()

	install(FILES ${locale_files}
		DESTINATION "${CMAKE_INSTALL_FULL_DATAROOTDIR}/obs/obs-plugins/obs-animation/locale")
endif()

# MacOS-specific build settings and tasks
if(APPLE)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++ -fvisibility=default")

	set(CMAKE_SKIP_RPATH TRUE)
	set_target_properties(obs-animation PROPERTIES PREFIX "")
	target_link_libraries(obs-animation "${OBS_FRONTEND_LIB}")
endif()
