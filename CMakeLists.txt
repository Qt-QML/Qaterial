#
#   Qaterial CMake script by Olivier Le Doeuff
#

cmake_minimum_required(VERSION 3.14.0 FATAL_ERROR)

# ───── PROJECT OPTIONS ─────

set(QATERIAL_VERSION_MAJOR 1)
set(QATERIAL_VERSION_MINOR 2)
set(QATERIAL_VERSION_PATCH 1)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git")
  execute_process(
    COMMAND git rev-parse --short HEAD
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    OUTPUT_VARIABLE QATERIAL_VERSION_TAG
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
endif()
if(NOT QATERIAL_VERSION_TAG)
  set(QATERIAL_VERSION_TAG 00000000)
endif()
set(QATERIAL_VERSION_TAG_HEX 0x${QATERIAL_VERSION_TAG})
set(QATERIAL_VERSION ${QATERIAL_VERSION_MAJOR}.${QATERIAL_VERSION_MINOR}.${QATERIAL_VERSION_PATCH} CACHE STRING "Version of Qaterial")

# Options
set(QATERIAL_PROJECT "Qaterial" CACHE STRING "Project Name")
set(QATERIAL_TARGET ${QATERIAL_PROJECT} CACHE STRING "Target Name")
set(QATERIAL_FOLDER_PREFIX ${QATERIAL_PROJECT} CACHE STRING "Prefix folder for all Qaterial generated targets in generated project (only decorative)")
set(QATERIAL_BUILD_SHARED OFF CACHE BOOL "Build as a shared library")
set(QATERIAL_ENABLE_HOTRELOAD OFF CACHE BOOL "Build Qaterial HotReload executable")
set(QATERIAL_ENABLE_HOTRELOAD_LIB ${QATERIAL_ENABLE_HOTRELOAD} CACHE BOOL "Build Qaterial HotReload executable")
set(QATERIAL_ENABLE_HELLOWORLD OFF CACHE BOOL "Build Qaterial HelloWorld executable")
set(QATERIAL_ENABLE_TESTS OFF CACHE BOOL "Build Qaterial test executable")
set(QATERIAL_ENABLE_PCH ON CACHE BOOL
 "Enable precompile headers support for 'Qaterial'. \"
 Only work if CMake support 'target_precompile_headers'. \"
 This can speed up compilation time.")

set(QATERIAL_ICONS "*.svg" CACHE STRING "Filter embedded icons")

project(${QATERIAL_PROJECT} VERSION ${QATERIAL_VERSION} LANGUAGES CXX)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

message(STATUS "------ ${QATERIAL_TARGET} Configuration ------")
message(STATUS "QATERIAL_PROJECT                : ${QATERIAL_PROJECT}")
message(STATUS "QATERIAL_TARGET                 : ${QATERIAL_TARGET}")
message(STATUS "QATERIAL_VERSION                : ${QATERIAL_VERSION}")
message(STATUS "QATERIAL_VERSION_TAG_HEX        : ${QATERIAL_VERSION_TAG_HEX}")
message(STATUS "QATERIAL_BUILD_SHARED           : ${QATERIAL_BUILD_SHARED}")
message(STATUS "QATERIAL_FOLDER_PREFIX          : ${QATERIAL_FOLDER_PREFIX}")
message(STATUS "QATERIAL_ENABLE_HOTRELOAD       : ${QATERIAL_ENABLE_HOTRELOAD}")
message(STATUS "QATERIAL_ENABLE_HOTRELOAD_LIB   : ${QATERIAL_ENABLE_HOTRELOAD_LIB}")
message(STATUS "QATERIAL_ENABLE_HELLOWORLD      : ${QATERIAL_ENABLE_HELLOWORLD}")
message(STATUS "QATERIAL_ENABLE_TESTS           : ${QATERIAL_ENABLE_TESTS}")
message(STATUS "------ ${QATERIAL_TARGET} End Configuration ------")

# ───── QT DEPENDENCIES ─────

find_package(Qt5 REQUIRED COMPONENTS
  Core
  Gui
  Qml
  Quick
  QuickControls2
  Svg
  Xml
  QuickCompiler
)
find_package(Qt5 QUIET COMPONENTS QmlWorkerScript)

# ───── FETCH DEPEDENCIES ─────

include(cmake/FetchSpdlog.cmake)
include(cmake/FetchQOlm.cmake)

# ───── QML SOURCES ─────

file(GLOB_RECURSE QATERIAL_QML_SRCS qml/*.qml)

foreach(SOURCE IN ITEMS ${QATERIAL_QML_SRCS})
  get_filename_component(SOURCE_PATH "${SOURCE}" PATH)
  file(RELATIVE_PATH SOURCE_PATH_REL ${CMAKE_CURRENT_SOURCE_DIR}/qml "${SOURCE_PATH}")
  string(REPLACE "/" "\\" GROUP_PATH "${SOURCE_PATH_REL}")
  source_group("Qml\\${GROUP_PATH}" FILES "${SOURCE}")
endforeach()

include(cmake/FetchQtGeneratorCMake.cmake)
include(cmake/FetchMaterialDesignIcons.cmake)

qt_generate_qrc(QATERIAL_QML_ICONS_QRC
  SOURCE_DIR ${MATERIALDESIGNICONS_ICONS_DIR}
  NAME QaterialIcons.qrc
  PREFIX "Qaterial/Icons"
  GLOB_EXPRESSION ${QATERIAL_ICONS})

qt_generate_qrc_alias_qt_object(QATERIAL_ICONS_QML
  SOURCE_DIR ${MATERIALDESIGNICONS_ICONS_DIR}
  NAME "${CMAKE_CURRENT_BINARY_DIR}/QaterialImplIconsGenerated/Icons.qml"
  PREFIX "Qaterial/Icons"
  GLOB_EXPRESSION ${QATERIAL_ICONS})

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/QaterialImplIconsGenerated/qmldir
  "module Qaterial.Impl.Icons\n"
  "Icons 1.0 Icons.qml\n"
)

qt_generate_qrc(QATERIAL_QML_ICONS_ALIAS_QRC
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/QaterialImplIconsGenerated"
  NAME QaterialIconsAliases.qrc
  PREFIX "Qaterial/Impl/Icons"
  GLOB_EXPRESSION "*.qml;qmldir;")

qt_generate_qmldir(QATERIAL_QMLDIR_FILE
 SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/qml/Qaterial
 MODULE "Qaterial")

qt_generate_qrc(QATERIAL_QML_QRC
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/qml/Qaterial
  DEST_DIR ${CMAKE_CURRENT_BINARY_DIR}/QaterialGenerated
  NAME Qaterial.qrc
  PREFIX "Qaterial"
  GLOB_EXPRESSION "*.qml;qmldir"
  RECURSE)

qt_generate_qrc(QATERIAL_FONTS_QML_QRC
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/qml/Qaterial
  DEST_DIR ${CMAKE_CURRENT_BINARY_DIR}/QaterialGenerated
  NAME QaterialFonts.qrc
  PREFIX "Qaterial"
  GLOB_EXPRESSION "*.ttf"
  RECURSE)

# Add qml to the qtquick compiler (should only be call once per target)
qtquick_compiler_add_resources(QATERIAL_QML_RES ${QATERIAL_QML_QRC} ${QATERIAL_FONTS_QML_QRC} ${QATERIAL_QML_ICONS_QRC} ${QATERIAL_QML_ICONS_ALIAS_QRC})

# Dont Run Moc on cpp file generated from qml
foreach(QATERIAL_CURRENT_QML_FILE ${QATERIAL_QML_RES})
  set_property(SOURCE ${QATERIAL_CURRENT_QML_FILE} PROPERTY SKIP_AUTOMOC ON)
  source_group("Qml\\QtQuickCompiler Files" FILES ${QATERIAL_CURRENT_QML_FILE})
endforeach()

set(QATERIAL_IO_SRCS
  include/Qaterial/TextFile.hpp
  src/TextFile.cpp
  include/Qaterial/Clipboard.hpp
  src/Clipboard.cpp
  include/Qaterial/FolderTreeModel.hpp
  src/FolderTreeModel.cpp
)

set(QATERIAL_NAVIGATION_SRCS
  include/Qaterial/StepperElement.hpp
  src/StepperElement.cpp
  include/Qaterial/TreeElement.hpp
  src/TreeElement.cpp
)

set(QATERIAL_UTILS_SRCS
  include/Qaterial/Qaterial.hpp
  include/Qaterial/Export.hpp
  include/Qaterial/Property.hpp
  include/Qaterial/Version.hpp
  src/Version.cpp
  include/Qaterial/Utils.hpp
  src/Utils.cpp
  include/Qaterial/Logger.hpp
  src/Logger.cpp
)

set(QATERIAL_SRCS
  ${QATERIAL_IO_SRCS}
  ${QATERIAL_NAVIGATION_SRCS}
  ${QATERIAL_UTILS_SRCS}
  ${QATERIAL_QML_RES}
  ${QATERIAL_QML_SRCS}
)

source_group("IO" FILES ${QATERIAL_IO_SRCS})
source_group("Navigation" FILES ${QATERIAL_NAVIGATION_SRCS})
source_group("Utils" FILES ${QATERIAL_UTILS_SRCS})

# ───── QATERIAL TARGET ─────

if(${QATERIAL_BUILD_SHARED})
  add_library(${QATERIAL_TARGET} SHARED ${QATERIAL_SRCS})
  target_compile_definitions(${QATERIAL_TARGET} PUBLIC -DQATERIAL_SHARED)
else()
  add_library(${QATERIAL_TARGET} STATIC ${QATERIAL_SRCS})
  target_compile_definitions(${QATERIAL_TARGET} PUBLIC -DQATERIAL_STATIC)
endif()

target_compile_features(${QATERIAL_TARGET} PUBLIC cxx_std_11)
target_include_directories(${QATERIAL_TARGET} PUBLIC include)
if(QATERIAL_ENABLE_PCH AND COMMAND target_precompile_headers)
  target_precompile_headers(${QATERIAL_TARGET} PRIVATE include/Qaterial/Pch/Pch.hpp)
endif()
target_link_libraries(${QATERIAL_TARGET} PUBLIC
  spdlog
  QOlm
  Qt5::Core
  Qt5::Gui
  Qt5::Svg
  Qt5::Xml
  Qt5::Qml
  Qt5::Quick
  Qt5::QuickControls2
)
set_target_properties(${QATERIAL_TARGET} PROPERTIES AUTORCC TRUE)
set_target_properties(${QATERIAL_TARGET} PROPERTIES AUTOMOC TRUE)

# Version
target_compile_definitions(${QATERIAL_TARGET}
  PRIVATE -DQATERIAL_VERSION_MAJOR=${QATERIAL_VERSION_MAJOR}
  PRIVATE -DQATERIAL_VERSION_MINOR=${QATERIAL_VERSION_MINOR}
  PRIVATE -DQATERIAL_VERSION_PATCH=${QATERIAL_VERSION_PATCH}
  PRIVATE -DQATERIAL_VERSION_TAG=${QATERIAL_VERSION_TAG}
  PRIVATE -DQATERIAL_VERSION_TAG_HEX=${QATERIAL_VERSION_TAG_HEX}
)

if(QATERIAL_FOLDER_PREFIX)
  set_target_properties(${QATERIAL_TARGET} PROPERTIES FOLDER ${QATERIAL_FOLDER_PREFIX})
endif()

# Correctly link to static qt
get_target_property(QT_TARGET_TYPE Qt5::Core TYPE)
if(${QT_TARGET_TYPE} STREQUAL "STATIC_LIBRARY")
  include(cmake/FetchQtStaticCMake.cmake)
  qt_generate_qml_plugin_import(${QATERIAL_TARGET}
    QML_SRC qml
    EXTRA_PLUGIN
      QtQuickVirtualKeyboardPlugin
      QtQuickVirtualKeyboardSettingsPlugin
      QtQuickVirtualKeyboardStylesPlugin
      QmlFolderListModelPlugin
    VERBOSE
  )
  qt_generate_plugin_import(${QATERIAL_TARGET} VERBOSE)
endif()

if(TARGET Qt5::QmlWorkerScript)
  target_link_libraries(${QATERIAL_TARGET} PUBLIC Qt5::QmlWorkerScript)
endif()

# ───── QATERIAL TOOLS ─────

if(QATERIAL_ENABLE_HOTRELOAD OR QATERIAL_ENABLE_HOTRELOAD_LIB OR QATERIAL_ENABLE_HELLOWORLD)
  add_subdirectory(tools)
endif()

# ───── QATERIAL TESTS ─────

if(QATERIAL_ENABLE_TESTS)
  enable_testing()
  add_subdirectory(tests)
endif()
