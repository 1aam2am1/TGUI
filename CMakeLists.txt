if(CMAKE_SYSTEM_NAME MATCHES "Android")
    cmake_minimum_required(VERSION 3.7)  # CMake's built-in Android support requires 3.7
else()
    cmake_minimum_required(VERSION 3.5)  # Can be at most 3.5.1 as long as Ubuntu 16.04 is supported
endif()

# Use new RPATH behavior on macOS (CMake 3.9 and newer)
if(POLICY CMP0068)
    cmake_policy(SET CMP0068 NEW)
endif()

# Don't warn about existence of SFML_ROOT variable (CMake 3.12 and newer)
if(POLICY CMP0074)
    cmake_policy(SET CMP0074 NEW)
endif()

include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/Macros.cmake)

# Set a default build type
tgui_set_option(CMAKE_BUILD_TYPE Release STRING "Choose the type of build (Debug or Release)")

# Set CMAKE_MODULE_PATH to find SFML < 2.5 without manually having to specify a module path
if(NOT SFML_DIR AND NOT CMAKE_MODULE_PATH)
    set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules")
endif()

# Project name and version
project(TGUI VERSION 0.8.8)

# TGUI uses the paths from the cmake GNUInstallDirs module as defaults (https://cmake.org/cmake/help/v3.0/module/GNUInstallDirs.html)
include(GNUInstallDirs)

# Include the configuration file
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/Config.cmake)

# Add an option for choosing the build type (shared or static)
if(NOT TGUI_OS_IOS AND NOT TGUI_OS_ANDROID)
    tgui_set_option(TGUI_SHARED_LIBS TRUE BOOL "TRUE to build TGUI as a shared library, FALSE to build it as a static library")
else()
    if(TGUI_OS_IOS)
        set(TGUI_SHARED_LIBS FALSE)
    elseif(TGUI_OS_ANDROID)
        set(TGUI_SHARED_LIBS TRUE)
    endif()
endif()

# Add option to build the examples
if(NOT TGUI_OS_ANDROID)
    tgui_set_option(TGUI_BUILD_EXAMPLES FALSE BOOL "TRUE to build the TGUI examples, FALSE to ignore them")
else()
    set(TGUI_BUILD_EXAMPLES FALSE)
endif()

tgui_set_option(TGUI_BUILD_TESTS FALSE BOOL "TRUE to build the TGUI tests")
tgui_set_option(TGUI_BUILD_GUI_BUILDER TRUE BOOL "TRUE to compile the GUI Builder")
tgui_set_option(TGUI_BUILD_DOC FALSE BOOL "TRUE to generate the API documentation, FALSE to ignore it")
tgui_set_option(TGUI_CXX_STANDARD "14" STRING "C++ standard version to build TGUI with. Possible values: 14, 17 or 20. Projects using TGUI must use a version equal or higher to this")

# At least c++14 has to be used
if (TGUI_CXX_STANDARD LESS "14")
    message(FATAL_ERROR "TGUI_CXX_STANDARD has to be at least 14")
else()
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/include/TGUI/Config.hpp.in ${CMAKE_CURRENT_SOURCE_DIR}/include/TGUI/Config.hpp @ONLY)
endif()

# Define the install directory for miscellaneous files
if(TGUI_OS_WINDOWS OR TGUI_OS_IOS)
    set(DEFAULT_INSTALL_MISC_DIR .)
elseif(TGUI_OS_ANDROID)
    set(DEFAULT_INSTALL_MISC_DIR ${CMAKE_ANDROID_NDK}/sources/third_party/tgui)
else()
    set(DEFAULT_INSTALL_MISC_DIR ${CMAKE_INSTALL_DATAROOTDIR}/tgui-${TGUI_VERSION_MAJOR}.${TGUI_VERSION_MINOR})
endif()
tgui_set_option(TGUI_MISC_INSTALL_PREFIX "${DEFAULT_INSTALL_MISC_DIR}" PATH "Prefix installation path for miscellaneous files")
mark_as_advanced(TGUI_MISC_INSTALL_PREFIX)

# Install pkg-config files by default on Linux and BSD
if (TGUI_OS_LINUX OR TGUI_OS_BSD)
    set(TGUI_INSTALL_PKGCONFIG_DEFAULT TRUE)
else()
    set(TGUI_INSTALL_PKGCONFIG_DEFAULT FALSE)
endif()

tgui_set_option(TGUI_INSTALL_PKGCONFIG_FILES ${TGUI_INSTALL_PKGCONFIG_DEFAULT} BOOL "TRUE to automatically install pkg-config files so other projects can find TGUI")

if (TGUI_INSTALL_PKGCONFIG_FILES)
    tgui_set_option(TGUI_PKGCONFIG_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}/${TGUI_PKGCONFIG_DIR}" PATH "Install directory for TGUI's pkg-config .pc files")

    configure_file("pkgconfig/tgui.pc.in" "pkgconfig/tgui.pc" @ONLY)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/pkgconfig/tgui.pc" DESTINATION "${TGUI_PKGCONFIG_INSTALL_PREFIX}")
endif()

# Define an option for choosing between static and dynamic C runtime
if(TGUI_OS_WINDOWS)
    tgui_set_option(TGUI_USE_STATIC_STD_LIBS FALSE BOOL "TRUE to statically link to the standard libraries, FALSE to use them as DLLs. This option has to match with the one from sfml.")

    # The following combination of flags is not valid
    if(TGUI_SHARED_LIBS AND TGUI_USE_STATIC_STD_LIBS)
        message(FATAL_ERROR "TGUI_SHARED_LIBS and TGUI_USE_STATIC_STD_LIBS cannot be used together")
    endif()
endif()

# Mac OS X specific options
if(TGUI_OS_MACOSX)

    # Add an option to build framework instead of dylib (release only)
    tgui_set_option(TGUI_BUILD_FRAMEWORK FALSE BOOL "TRUE to build TGUI as a framework library (release only), FALSE to build according to TGUI_SHARED_LIBS")

    # Only the default architecture (i.e. 64-bit) is supported
    if(CMAKE_OSX_ARCHITECTURES AND NOT "${CMAKE_OSX_ARCHITECTURES}" STREQUAL "x86_64")
        message(FATAL_ERROR "Only 64-bit architecture is supported")
        return()
    endif()

    # Enable to use of rpath according to CMake Policy CMP0042
    set(CMAKE_MACOSX_RPATH 1)

    if(TGUI_BUILD_FRAMEWORK)
        # Frameworks are only available for release (because cmake currently doesn't allow specifying a custom framework name so TGUI-d is not possible)
        if(NOT CMAKE_BUILD_TYPE STREQUAL "Release")
            message(FATAL_ERROR "CMAKE_BUILD_TYPE should be \"Release\" when TGUI_BUILD_FRAMEWORK is TRUE")
            return()
        endif()

        # Frameworks only work with TGUI_SHARED_LIBS enabled
        if(NOT TGUI_SHARED_LIBS)
            message(FATAL_ERROR "TGUI_SHARED_LIBS should be TRUE when TGUI_BUILD_FRAMEWORK is TRUE")
            return()
        endif()
    endif()
endif()

# Android options
if(TGUI_OS_ANDROID)

    # Make sure there's the android library available
    if(CMAKE_ANDROID_API LESS 19)
        message(FATAL_ERROR "Android API level (${CMAKE_ANDROID_API}) must be equal or greater than 19.")
    endif()

    # CMake doesn't support defining the STL to be used with Nsight Tegra, so warn the user
    if(CMAKE_VS_PLATFORM_NAME STREQUAL "Tegra-Android")
        message(WARNING "CMake might not properly support setting the STL. Make sure to adjust all generated library projects!")
    endif()

    # Install everything in $NDK/sources/third_party by default
    if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
        set(CMAKE_INSTALL_PREFIX ${CMAKE_ANDROID_NDK}/sources/third_party/tgui CACHE PATH "Installation path (should be inside your NDK's 'sources' directory)" FORCE)
    endif()

    # We install libs in a subdirectory named after the ABI (e.g. lib/armeabi/libtgui.so)
    set(CMAKE_INSTALL_LIBDIR "${CMAKE_INSTALL_LIBDIR}/${CMAKE_ANDROID_ARCH_ABI}")
endif()

# Add an option to choose whether PDB debug symbols should be generated for Visual Studio
if(TGUI_COMPILER_MSVC)
    tgui_set_option(TGUI_GENERATE_PDB TRUE BOOL "True to generate PDB debug symbols, FALSE otherwise.")
endif()

# Apply the TGUI_USE_STATIC_STD_LIBS option on windows when using VC++
# This can't be done in the tgui_set_stdlib macro because it is too late to change these variables by then
if(TGUI_OS_WINDOWS AND TGUI_COMPILER_MSVC AND TGUI_USE_STATIC_STD_LIBS)
    foreach(flag
            CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
            CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
        if(${flag} MATCHES "/MD")
            string(REGEX REPLACE "/MD" "/MT" ${flag} "${${flag}}")
        endif()
    endforeach()
endif()

# Link SFML in the same way as TGUI, unless SFML_STATIC_LIBRARIES is manually specified
if(NOT DEFINED SFML_STATIC_LIBRARIES)
  if(TGUI_SHARED_LIBS)
    set(SFML_STATIC_LIBRARIES FALSE)
  else()
    set(SFML_STATIC_LIBRARIES TRUE)
  endif()
endif()

# Attempt to find the SFML dependencies when linking statically
if(NOT TGUI_SHARED_LIBS)
    if(SFML_ROOT)
        if(TGUI_OS_WINDOWS)
            set(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH} "${SFML_ROOT}/extlibs/headers")
            if(ARCH_32BITS)
                if(TGUI_COMPILER_MSVC AND MSVC_VERSION LESS 1900) # older than VC++14
                    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${SFML_ROOT}/extlibs/libs-msvc/x86")
                elseif(TGUI_COMPILER_MSVC) # VC++14 or newer
                    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${SFML_ROOT}/extlibs/libs-msvc-universal/x86")
                else() # gcc
                    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${SFML_ROOT}/extlibs/libs-mingw/x86")
                endif()
            elseif(ARCH_64BITS)
                if(TGUI_COMPILER_MSVC AND MSVC_VERSION LESS 1900) # older than VC++14
                    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${SFML_ROOT}/extlibs/libs-msvc/x64")
                elseif(TGUI_COMPILER_MSVC) # VC++14 or newer
                    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${SFML_ROOT}/extlibs/libs-msvc-universal/x64")
                else() # gcc
                    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${SFML_ROOT}/extlibs/libs-mingw/x64")
                endif()
            endif()
        elseif(TGUI_OS_MACOSX)
            set(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH} "${SFML_ROOT}/extlibs/headers")
            set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${SFML_ROOT}/extlibs/libs-osx/lib/")
        elseif(TGUI_OS_ANDROID)
            set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${CMAKE_ANDROID_NDK}/sources/third_party/sfml/extlibs/lib/${CMAKE_ANDROID_ARCH_ABI}/")
        elseif(TGUI_OS_IOS)
            set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${SFML_ROOT}/extlibs/libs-ios/")
        endif()
    endif()
endif()

# Find sfml if not build in same project
if(NOT TARGET sfml-graphics)
    if(TGUI_OS_WINDOWS AND TGUI_COMPILER_MSVC) # Also look for the main component when using Visual Studio
        find_package(SFML 2 COMPONENTS main graphics window system)
    elseif(TGUI_OS_ANDROID)  # Search for SFML in the android NDK (if no other directory is specified)
        find_package(SFML 2 COMPONENTS graphics window system PATHS "${CMAKE_ANDROID_NDK}/sources/third_party/sfml/lib/${CMAKE_ANDROID_ARCH_ABI}/cmake/SFML")
    elseif(TGUI_OS_IOS)  # Use the find_host_package macro from the toolchain on iOS
        find_host_package(SFML 2 COMPONENTS main graphics window system)
    else()
        find_package(SFML 2 COMPONENTS graphics window system)
    endif()

    # find_package couldn't find SFML
    if(NOT SFML_FOUND)
        set(SFML_DIR "" CACHE PATH "Path to SFMLConfig.cmake")
        set(SFML_ROOT "" CACHE PATH "SFML root directory")
        message(FATAL_ERROR "CMake couldn't find SFML.\nEither set SFML_DIR to the directory containing SFMLConfig.cmake or set the SFML_ROOT entry to SFML's root directory (containing \"include\" and \"lib\" directories).")
    endif()
endif()

# Set the path for the libraries
set(LIBRARY_OUTPUT_PATH "${PROJECT_BINARY_DIR}/lib")

# Jump to the CMakeLists.txt file in the source folder
add_subdirectory(src/TGUI)

# Build the documentation when requested
if(TGUI_BUILD_DOC)
    add_subdirectory(doc)
endif()

# Build the examples if requested
if(TGUI_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

# Build the tests if requested
if(TGUI_BUILD_TESTS)
    if(NOT ${CMAKE_BUILD_TYPE} STREQUAL "Debug")
        message(FATAL_ERROR "TGUI_BUILD_TESTS should only be enabled when CMAKE_BUILD_TYPE is Debug")
    endif()

    add_subdirectory(tests)
endif()

# Build the GUI Builder if requested
if(TGUI_BUILD_GUI_BUILDER)
    add_subdirectory("${PROJECT_SOURCE_DIR}/gui-builder")
endif()

# Install include files
if(NOT TGUI_BUILD_FRAMEWORK)
    install(DIRECTORY include
            DESTINATION .
            COMPONENT devel
            FILES_MATCHING PATTERN "*.hpp" PATTERN "*.inl")

    if(TGUI_GENERATE_PDB)
        install(DIRECTORY "${PROJECT_BINARY_DIR}/lib"
                DESTINATION .
                COMPONENT devel
                FILES_MATCHING PATTERN "*.pdb")
    endif()
endif()

# Install miscellaneous files
install(FILES license.txt DESTINATION ${CMAKE_INSTALL_DOCDIR})
install(FILES README.md DESTINATION ${CMAKE_INSTALL_DOCDIR})
install(DIRECTORY themes DESTINATION "${TGUI_MISC_INSTALL_PREFIX}")
