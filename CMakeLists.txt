# --- Detect if the plugin is build out of tree or not ---
if(CMAKE_PROJECT_NAME STREQUAL "obs-studio")
  set(BUILD_OUT_OF_TREE OFF)
else()
  set(BUILD_OUT_OF_TREE ON)
  cmake_minimum_required(VERSION 3.18)
endif()

project(draw-dock VERSION 0.2.4)
set(PROJECT_FULL_NAME "Draw Dock")

# Set new UUIDs when you start to create a new plugin.
set(MACOS_PACKAGE_UUID "8DFC9914-046A-4C0E-8BE6-D9718C5F8755")
set(MACOS_INSTALLER_UUID "FBF75ECD-EA89-4D99-93DA-6E3EC32605A3")

add_library(${PROJECT_NAME} MODULE)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/version.h.in ${CMAKE_CURRENT_SOURCE_DIR}/version.h)

set(CMAKE_AUTORCC ON)

target_sources(${PROJECT_NAME} PRIVATE
	draw-dock.cpp
	draw-dock.hpp
    draw-source.c
    draw-source.h
    qt-display.cpp
    qt-display.hpp
    name-dialog.cpp
    name-dialog.hpp
    display-helpers.hpp
	version.h)

if(BUILD_OUT_OF_TREE)
	find_package(libobs REQUIRED)
	find_package(obs-frontend-api REQUIRED)
	include(cmake/ObsPluginHelpers.cmake)
	find_qt(COMPONENTS Widgets COMPONENTS_LINUX Gui)
	set(OBS_FRONTEND_API_NAME "obs-frontend-api")
else()
	if(OBS_VERSION VERSION_GREATER_EQUAL 30.1.0)
		find_package(Qt6 COMPONENTS Core Widgets)
	else()
		find_qt(COMPONENTS Widgets COMPONENTS_LINUX Gui)
	endif()
	set(OBS_FRONTEND_API_NAME "frontend-api")
endif()

if(OS_WINDOWS)
	get_filename_component(ISS_FILES_DIR "${CMAKE_BINARY_DIR}\\..\\package" ABSOLUTE)
	file(TO_NATIVE_PATH "${ISS_FILES_DIR}" ISS_FILES_DIR)
	get_filename_component(ISS_PACKAGE_DIR "${CMAKE_PACKAGE_PREFIX}\\.." ABSOLUTE)
	file(TO_NATIVE_PATH "${ISS_PACKAGE_DIR}" ISS_PACKAGE_DIR)
	get_filename_component(ISS_SOURCE_DIR "${PROJECT_SOURCE_DIR}" ABSOLUTE)
	file(TO_NATIVE_PATH "${ISS_SOURCE_DIR}" ISS_SOURCE_DIR)
	configure_file("installer.iss.in"
			"${PROJECT_BINARY_DIR}/installer.iss"
		)
	configure_file(${CMAKE_CURRENT_SOURCE_DIR}/resource.rc.in ${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}.rc)
	target_sources(${PROJECT_NAME} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}.rc)
	
elseif(OS_MACOS)
	set(MACOS_BUNDLEID "com.exeldro.${PROJECT_NAME}")
	set(MACOSX_PLUGIN_GUI_IDENTIFIER "${MACOS_BUNDLEID}")
	set(MACOSX_PLUGIN_BUNDLE_VERSION "${PROJECT_VERSION}")
	set(MACOSX_PLUGIN_SHORT_VERSION_STRING "1")
	configure_file(cmake/bundle/macos/installer-macos.pkgproj.in ${CMAKE_BINARY_DIR}/installer-macos.generated.pkgproj)
	target_compile_options(${PROJECT_NAME} PRIVATE -Wall)
elseif(OS_POSIX)
if(Qt_VERSION VERSION_LESS "6.9.0")
	target_link_libraries(${PROJECT_NAME} Qt::GuiPrivate)
endif()
endif()

set_target_properties(${PROJECT_NAME} PROPERTIES AUTOMOC ON AUTOUIC ON AUTORCC ON)

target_link_libraries(${PROJECT_NAME}
		OBS::${OBS_FRONTEND_API_NAME}
		Qt::Widgets
		OBS::libobs)

if(BUILD_OUT_OF_TREE)
	if(NOT LIB_OUT_DIR)
		set(LIB_OUT_DIR "/lib/obs-plugins")
	endif()
	if(NOT DATA_OUT_DIR)
		set(DATA_OUT_DIR "/share/obs/obs-plugins/${PROJECT_NAME}")
	endif()
	set_target_properties(${PROJECT_NAME} PROPERTIES PREFIX "")
	install(TARGETS ${PROJECT_NAME}
		LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/${LIB_OUT_DIR})
	install(DIRECTORY data/locale data/effects
		DESTINATION ${CMAKE_INSTALL_PREFIX}/${DATA_OUT_DIR})
	setup_plugin_target(${PROJECT_NAME})
else()
	target_include_directories(${PROJECT_NAME} PRIVATE
		"${CMAKE_SOURCE_DIR}/UI/obs-frontend-api")
	if(OBS_CMAKE_VERSION VERSION_GREATER_EQUAL 3.0.0)
		set_target_properties_obs(${PROJECT_NAME} PROPERTIES FOLDER "plugins/exeldro" PREFIX "")
	else()
		set_target_properties(${PROJECT_NAME} PROPERTIES FOLDER "plugins/exeldro")
		setup_plugin_target(${PROJECT_NAME})
	endif()
endif()
