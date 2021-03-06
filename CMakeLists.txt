cmake_minimum_required(VERSION 3.2)

option (BUILD_QT "Make a Qt App" ON)
option (BUILD_IMGUI "Make an imgui App" ON)

set (CMAKE_CXX_STANDARD 14)
set (CMAKE_EXPORT_COMPILE_COMMANDS ON)

PROJECT (StarterKit)

MESSAGE(STATUS " CMakeLists: StarterKit")

SET_PROPERTY(GLOBAL PROPERTY USE_FOLDERS ON)
SET_PROPERTY(GLOBAL PROPERTY AUTOGEN_TARGETS_FOLDER AutoMoc)

if (CMAKE_BUILD_TYPE MATCHES Coverage)
    set (CMAKE_BUILD_TYPE "Debug")
    set (PROJECT_COVERAGE ON)
    set (BUILD_IMGUI ON)
endif()

# global needed variables
SET (APPLICATION_NAME "StarterKit")
SET (APPLICATION_VERSION_MAJOR "0")
SET (APPLICATION_VERSION_MINOR "1")
SET (APPLICATION_VERSION_PATCH "0")
SET (APPLICATION_VERSION "${APPLICATION_VERSION_MAJOR}.${APPLICATION_VERSION_MINOR}.${APPLICATION_VERSION_PATCH}")

if (BUILD_QT)
SET(CMAKE_AUTORCC ON)
SET(CMAKE_PREFIX_PATH $ENV{QT_INSTALL_LOCATION})
SET(AUTOGEN_TARGETS_FOLDER MocTargets)

FIND_PACKAGE(Qt5 COMPONENTS Core Widgets Gui REQUIRED)

endif()

SET(APP_LIBRARY App)
SET(APP_NAME_QT Main_Qt)
SET(APP_NAME_IMGUI Main_ImGui)

# Third party sources
SET(M3RDPARTY_SOURCE "")
SET(M3RDPARTY_INCLUDE "")

# Common sources
SET(MCOMMON_SOURCE "")
SET(MCOMMON_INCLUDE "")

SET(RESOURCE_FOLDER "" )
SET(RESOURCE_FILES "")

SET(APPLIB_SOURCE "")
SET(APPLIB_INCLUDE "")

SET(APPLIBIMGUI_SOURCE "")
SET(APPLIBIMGUI_INCLUDE "")

SET(APPLIBQT_SOURCE "")
SET(APPLIBQT_INCLUDE "")

SET(APP_SOURCE "")
SET(APP_INCLUDE "")

SET(APPIMGUI_SOURCE "")
SET(APPIMGUI_INCLUDE "")

SET(APPQT_SOURCE "")
SET(APPQT_INCLUDE "")

# Set all compiler flags 
INCLUDE(cmake/all.cmake)

# Functions for file copying
INCLUDE(m3rdparty/cmake/copy_files.cmake)

# config_app.h checks
# This makes a config_shared.h file which can be included for system settings
#  Must be after setting up the platforms
CONFIGURE_FILE(${CMAKE_CURRENT_LIST_DIR}/cmake/config_app.h.cmake ${CMAKE_BINARY_DIR}/config_app.h)

INCLUDE(m3rdparty/list.cmake)
INCLUDE(mcommon/list.cmake)
INCLUDE(src/list.cmake)

INCLUDE_DIRECTORIES(
    m3rdparty
    ${M3RDPARTY_INCLUDE}
    ${MCOMMON_INCLUDE}
    ${APPLIB_INCLUDE}
    ${APP_INCLUDE}
    ${CMAKE_BINARY_DIR}
    .
    ) 

# Create the library - No depenencies on anything else in this file
ADD_LIBRARY(${APP_LIBRARY} STATIC ${APPLIB_SOURCE} ${APPLIBIMGUI_SOURCE} ${APPLIBQT_SOURCE} ${MCOMMON_SOURCE})

# Create the ImGUI version of the app
IF (BUILD_IMGUI)
SET(CMAKE_DEFAULT_STARTUP_PROJECT ${APP_NAME_IMGUI})
INCLUDE_DIRECTORIES(${APPIMGUI_INCLUDE} ${APPLIBIMGUI_INCLUDE})
ADD_EXECUTABLE (${APP_NAME_IMGUI} WIN32 
    ${APPIMGUI_SOURCE}
    ${APP_SOURCE}
    ) # Win32 ignored on non-windows

TARGET_LINK_LIBRARIES (${APP_NAME_IMGUI} ${APP_LIBRARY} ${PLATFORM_LINKLIBS} ${SDL_LINKLIBS})
ADD_DEPENDENCIES(${APP_NAME_IMGUI} sdl2)
ENDIF() # IMGUI

# Create the Qt version of the app
IF (BUILD_QT)
SET(CMAKE_DEFAULT_STARTUP_PROJECT ${APP_NAME_QT})
INCLUDE_DIRECTORIES(${APPQT_INCLUDE} ${APPLIBQT_INCLUDE})
ADD_EXECUTABLE (${APP_NAME_QT} WIN32
    ${APPQT_SOURCE}
    ${APP_SOURCE}
    ${RESOURCE_FILES}) # Win32 ignored on non-windows
TARGET_LINK_LIBRARIES(${APP_NAME_QT} Qt5::Core Qt5::Gui Qt5::Widgets ${APP_LIBRARY} ${PLATFORM_LINKLIBS})
SET_TARGET_PROPERTIES(${APP_NAME_QT} PROPERTIES AUTOMOC TRUE)

if(WIN32)
message(INFO "Copying required Qt libraries and binaries to output directory....")
# Run winddeployqt if it can be found, to ensure installed dependencies
find_program(WINDEPLOYQT_EXECUTABLE NAMES windeployqt HINTS ${QTDIR} ENV QTDIR PATH_SUFFIXES bin)
add_custom_command(TARGET ${APP_NAME_QT} POST_BUILD
COMMAND ${WINDEPLOYQT_EXECUTABLE} $<TARGET_FILE:${APP_NAME_QT}>)
ENDIF() # Win32
ENDIF() # Build QT

# Unit tests
# Require SDL/IMgui build to work
IF (BUILD_IMGUI)
enable_testing()
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DGTEST_HAS_TR1_TUPLE=0")
SET (TEST_SOURCES
    ${M3RDPARTY_DIR}/googletest/googletest/src/gtest-all.cc
    ${TEST_SOURCES}
)
ADD_EXECUTABLE (unittests ${TEST_SOURCES} )
ADD_DEPENDENCIES(unittests sdl2)
TARGET_LINK_LIBRARIES (unittests ${APP_LIBRARY} ${PLATFORM_LINKLIBS} ${CMAKE_THREAD_LIBS_INIT})
add_test(unittests unittests)
INCLUDE_DIRECTORIES(
    ${M3RDPARTY_DIR}/googletest/googletest/include
    ${M3RDPARTY_DIR}/googletest/googletest
    ${M3RDPARTY_DIR}/googletest/googlemock/include
    ${M3RDPARTY_DIR}/googletest/googlemock
)
ENDIF()

SOURCE_GROUP(src REGULAR_EXPRESSION "src/.*")

SOURCE_GROUP(AutoMoc FILES ${CMAKE_CURRENT_BINARY_DIR}/${APP_NAME_QT}_automoc.cpp ) 
SOURCE_GROUP(AutoMoc REGULAR_EXPRESSION "(qrc)+_.*" ) 
