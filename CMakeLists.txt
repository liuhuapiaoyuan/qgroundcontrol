cmake_minimum_required(VERSION 3.22.1 FATAL_ERROR)

#######################################################
#                   Project Info
#######################################################

project(QGroundControl LANGUAGES C CXX)

#######################################################
#            CMake Configuration Options
#######################################################

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/staging)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)

set(CMAKE_INCLUDE_CURRENT_DIR OFF)

include(FeatureSummary)
include(CMakePrintHelpers)
include(FetchContent)
include(ExternalProject)
include(GNUInstallDirs)

set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug;Release")
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()
cmake_print_variables(CMAKE_BUILD_TYPE)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

option(QGC_STABLE_BUILD "Stable build option" OFF)
cmake_print_variables(QGC_STABLE_BUILD)
if(NOT QGC_STABLE_BUILD)
    add_compile_definitions(DAILY_BUILD)
endif()

include(CMakeDependentOption)
cmake_dependent_option(QGC_BUILD_TESTING "Enable testing" ON "CMAKE_BUILD_TYPE STREQUAL Debug" OFF)

include(CompileOptions)

find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
endif()

#######################################################
#               Qt6 Configuration
#######################################################

include(Qt6QGCConfiguration)
cmake_print_variables(QT_VERSION QT_MKSPEC)

if(LINUX)
    set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/AppDir/usr)
elseif(ANDROID)
    cmake_print_variables(QT_ANDROID_APPLICATION_ARGUMENTS QT_HOST_PATH)

    set(QT_ENABLE_VERBOSE_DEPLOYMENT ON CACHE BOOL "Verbose Deployment")
    # set(QT_ANDROID_DEPLOYMENT_TYPE ON CACHE BOOL "Deployment Type")
    cmake_dependent_option(QT_ANDROID_SIGN_APK "Enable Signing APK" ON "CMAKE_BUILD_TYPE STREQUAL Release" OFF)
    if(QT_ANDROID_SIGN_APK)
        message(STATUS "Signing APK")
        message(STATUS "QT_ANDROID_KEYSTORE_PATH $ENV{QT_ANDROID_KEYSTORE_PATH}")
        message(STATUS "QT_ANDROID_KEYSTORE_ALIAS $ENV{QT_ANDROID_KEYSTORE_ALIAS}")
        # QT_ANDROID_KEYSTORE_STORE_PASS, QT_ANDROID_KEYSTORE_KEY_PASS
    endif()

    if(NOT Qt6LinguistTools_DIR)
        set(Qt6LinguistTools_DIR ${QT_HOST_PATH}/lib/cmake/Qt6LinguistTools)
    endif()
endif()

find_package(Qt6
    REQUIRED
    COMPONENTS
        Bluetooth
        Charts
        Concurrent
        Core
        Core5Compat
        Location
        Multimedia
        Network
        Positioning
        Quick
        QuickControls2
        QuickWidgets
        OpenGL
        Sql
        Svg
        Test
        TextToSpeech
        Widgets
        Xml
    OPTIONAL_COMPONENTS
        LinguistTools
        SerialPort
    HINTS
        ${QT_LIBRARY_HINTS}
)

qt_standard_project_setup(REQUIRES 6.6.0)

qt_policy(
    SET QTP0001 NEW
    SET QTP0002 NEW
)

set(QT_QML_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/qml" CACHE PATH "Install path for QML" FORCE)
set(QML_IMPORT_PATH ${CMAKE_SOURCE_DIR}/qml ${CMAKE_BINARY_DIR}/imports CACHE STRING "Extra QML Import Paths" FORCE)

#######################################################
#                QML Configuration
#######################################################

option(QGC_COMPILE_QML "Pre-compile QML files using the Qt Quick compiler." FALSE)
add_feature_info(COMPILE_QML COMPILE_QML "Pre-compile QML files using the Qt Quick compiler.")
if(QGC_COMPILE_QML)
    find_package(Qt6QuickCompiler)
    set_package_properties(Qt6QuickCompiler
        PROPERTIES
            DESCRIPTION "Pre-compile QML files using the Qt Quick compiler."
            TYPE OPTIONAL
    )
endif()

option(QGC_DEBUG_QML "Build QGroundControl with QML debugging/profiling support." FALSE)
add_feature_info(DEBUG_QML DEBUG_QML "Build QGroundControl with QML debugging/profiling support.")
if(QGC_DEBUG_QML)
    message(STATUS "To enable the QML debugger/profiler, run with: '-qmljsdebugger=port:1234'")
    add_compile_definitions(
        QMLJSDEBUGGER
        QT_QML_DEBUG
    )
endif()

#######################################################
#               QGroundControl Info
#######################################################

set(COMPANY "Mavlink")
set(COPYRIGHT "Copyright (c) 2018 QGroundControl. All rights reserved.")
set(IDENTIFIER "io.mavlink.qgroundcontrol")

add_compile_definitions(
    QGC_APPLICATION_NAME="QGroundControl"
    QGC_ORG_NAME="QGroundControl.org"
    QGC_ORG_DOMAIN="org.qgroundcontrol"
)

#######################################################
#                QGroundControl Git Information
#######################################################

include(Git)
cmake_print_variables(APP_VERSION_STR)

#######################################################
#                QGroundControl Resources
#######################################################

set(QGC_RESOURCES
    ${CMAKE_SOURCE_DIR}/qgcimages.qrc
    ${CMAKE_SOURCE_DIR}/qgcresources.qrc
    ${CMAKE_SOURCE_DIR}/qgroundcontrol.qrc
    ${CMAKE_SOURCE_DIR}/resources/InstrumentValueIcons/InstrumentValueIcons.qrc
    ${CMAKE_SOURCE_DIR}/src/FirmwarePlugin/APM/APMResources.qrc
    ${CMAKE_SOURCE_DIR}/src/FirmwarePlugin/PX4/PX4Resources.qrc
)

if(CONFIG_UTM_ADAPTER)
    list(APPEND QGC_RESOURCES ${CMAKE_SOURCE_DIR}/src/UTMSP/utmsp.qrc)
else()
    list(APPEND QGC_RESOURCES ${CMAKE_SOURCE_DIR}/src/UTMSP/dummy/utmsp_dummy.qrc)
endif()

if(QGC_BUILD_TESTING)
    list(APPEND QGC_RESOURCES ${CMAKE_SOURCE_DIR}/test/UnitTest.qrc)
endif()

if(WIN32)
    list(APPEND QGC_RESOURCES ${CMAKE_SOURCE_DIR}/deploy/windows/QGroundControl.rc)
elseif(MACOS)
    set(MACOSX_BUNDLE_ICON_FILE macx.icns)
    set(app_icon_macos "${CMAKE_SOURCE_DIR}/resources/icons/macx.icns")
    set_source_files_properties(${app_icon_macos} PROPERTIES MACOSX_PACKAGE_LOCATION "Resources")
    list(APPEND QGC_RESOURCES ${app_icon_macos})
endif()

#######################################################
#               QGroundControl Target
#######################################################

qt_add_executable(${PROJECT_NAME} src/main.cc ${QGC_RESOURCES})

add_subdirectory(libs)
add_subdirectory(src)

set_target_properties(${PROJECT_NAME}
    PROPERTIES
        QT_RESOURCE_PREFIX "/qgc"
)

target_precompile_headers(${PROJECT_NAME} PRIVATE src/pch.h)

if(Qt6LinguistTools_FOUND)
    file(GLOB TS_SOURCES RELATIVE ${CMAKE_SOURCE_DIR} ${CMAKE_SOURCE_DIR}/translations/qgc_*.ts)
    # TODO: Update to new qt_add_translations form in Qt6.7
    qt_add_translations(${PROJECT_NAME}
        TS_FILES
            ${TS_SOURCES}
    )
endif()

if(MACOS)
    set_target_properties(${PROJECT_NAME}
        PROPERTIES
            MACOSX_BUNDLE TRUE
            MACOSX_BUNDLE_INFO_PLIST ${CMAKE_SOURCE_DIR}/deploy/mac/MacOSXBundleInfo.plist.in
    )
elseif(IOS)
    set_target_properties(${PROJECT_NAME}
       PROPERTIES
           QT_IOS_LAUNCH_SCREEN ${CMAKE_SOURCE_DIR}/deploy/ios/QGCLaunchScreen.xib
    )
elseif(WIN32)
    set_target_properties(${PROJECT_NAME} PROPERTIES WIN32_EXECUTABLE TRUE)
elseif(ANDROID)
    FetchContent_Declare(android_openssl
        DOWNLOAD_EXTRACT_TIMESTAMP true
        URL https://github.com/KDAB/android_openssl/archive/refs/heads/master.zip
    )
    FetchContent_MakeAvailable(android_openssl)
    include(${android_openssl_SOURCE_DIR}/android_openssl.cmake)
    add_android_openssl_libraries(${PROJECT_NAME})

    cmake_print_variables(
        QT_ANDROID_ABIS
        QT_ANDROID_BUILD_ALL_ABIS
        QT_PATH_ANDROID_ABI_armeabi-v7a
        QT_PATH_ANDROID_ABI_arm64-v8a
        QT_PATH_ANDROID_ABI_x86
        QT_PATH_ANDROID_ABI_x86_64
    )

    set_target_properties(${PROJECT_NAME}
        PROPERTIES
            # QT_ANDROID_ABIS ${ANDROID_ABI}
            QT_ANDROID_MIN_SDK_VERSION 26
            QT_ANDROID_TARGET_SDK_VERSION 34
            QT_ANDROID_PACKAGE_SOURCE_DIR ${CMAKE_SOURCE_DIR}/android
            QT_ANDROID_VERSION_NAME "${CMAKE_PROJECT_VERSION}"
            # QT_ANDROID_VERSION_CODE "${CMAKE_PROJECT_VERSION}"
            QT_QML_ROOT_PATH ${CMAKE_SOURCE_DIR}
    )

    # get_target_property(QGC_ANDROID_DEPLOY_FILE ${PROJECT_NAME} QT_ANDROID_DEPLOYMENT_SETTINGS_FILE)
    # cmake_print_variables(QGC_ANDROID_DEPLOY_FILE)
endif()

target_link_libraries(${PROJECT_NAME} PRIVATE qgc)

#######################################################
#               Testing Configuration
#######################################################

if(QGC_BUILD_TESTING)
    add_subdirectory(test)
    target_link_libraries(${PROJECT_NAME} PRIVATE qgctest)
endif()

#######################################################
#		Install Configuration
#######################################################

install(
    TARGETS ${PROJECT_NAME}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    BUNDLE  DESTINATION .
)

qt_generate_deploy_qml_app_script(
    TARGET ${PROJECT_NAME}
    OUTPUT_SCRIPT deploy_script
    # DEPLOY_TOOL_OPTIONS
    MACOS_BUNDLE_POST_BUILD
    NO_UNSUPPORTED_PLATFORM_ERROR
    DEPLOY_USER_QML_MODULES_ON_UNSUPPORTED_PLATFORM
)
install(SCRIPT ${deploy_script})

install(
    DIRECTORY ${CMAKE_SOURCE_DIR}/resources/
    DESTINATION ${CMAKE_INSTALL_DATADIR}/qgroundcontrol
)
if(LINUX)
    install(
        FILES ${CMAKE_SOURCE_DIR}/deploy/linux/org.mavlink.qgroundcontrol.desktop
        DESTINATION ${CMAKE_INSTALL_DATADIR}/applications
    )
    install(
        FILES ${CMAKE_SOURCE_DIR}/resources/icons/qgroundcontrol.png
        DESTINATION ${CMAKE_INSTALL_DATADIR}/icons/hicolor/128x128/apps/
        RENAME org.mavlink.qgroundcontrol.png
    )
    configure_file(
        ${CMAKE_SOURCE_DIR}/deploy/linux/org.mavlink.qgroundcontrol.metainfo.xml.in
        ${CMAKE_BINARY_DIR}/metainfo/org.mavlink.qgroundcontrol.metainfo.xml
        @ONLY
    )
    install(
        FILES ${CMAKE_BINARY_DIR}/metainfo/org.mavlink.qgroundcontrol.metainfo.xml
        DESTINATION ${CMAKE_INSTALL_DATADIR}/metainfo/
    )
    install(SCRIPT "${CMAKE_SOURCE_DIR}/cmake/CreateAppImage.cmake")
elseif(WIN32)
    install(SCRIPT "${CMAKE_SOURCE_DIR}/cmake/CreateWinInstaller.cmake")
endif()
