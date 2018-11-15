################################################################################
# Project:  OpenBLAS
# Purpose:  CMake build scripts
# Author:   dmitry.baryshnikov@nextgis.com
################################################################################
# Copyright (C) 2018, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################

cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(openblas CXX C ASM Fortran)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

set(CMAKE_Fortran_USE_RESPONSE_FILE_FOR_OBJECTS ON)
set(CMAKE_CXX_USE_RESPONSE_FILE_FOR_OBJECTS ON)

if(OSX_FRAMEWORK)
    set(INSTALL_BIN_DIR "bin" CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR "Library/Frameworks" CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_CMAKECONF_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Resources/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
    set(INSTALL_INC_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Headers CACHE INTERNAL "Installation directory for headers" FORCE)
    set(SKIP_INSTALL_HEADERS OFF)
    set(SKIP_INSTALL_EXECUTABLES ON)
    set(SKIP_INSTALL_FILES ON)
    set(SKIP_INSTALL_EXPORT ON)
    set(CMAKE_MACOSX_RPATH ON)
    set(CMAKE_Fortran_CREATE_MACOSX_FRAMEWORK ON)
else()
    include(GNUInstallDirs)

    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_CMAKECONF_DIR ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
endif()

include(util)
check_version(OpenBLAS_MAJOR_VERSION OpenBLAS_MINOR_VERSION OpenBLAS_PATCH_VERSION)
set(OpenBLAS_VERSION "${OpenBLAS_MAJOR_VERSION}.${OpenBLAS_MINOR_VERSION}.${OpenBLAS_PATCH_VERSION}")
set(VERSION ${OpenBLAS_MAJOR_VERSION}.${OpenBLAS_MINOR_VERSION}.${OpenBLAS_PATCH_VERSION})

if(OSX_FRAMEWORK)
    set(FRAMEWORK_VERSION "${OpenBLAS_MAJOR_VERSION}")
endif()

report_version(${PROJECT_NAME} ${VERSION})

set(OpenBLAS_LIBNAME ${PROJECT_NAME})
set(LIB_NAME ${OpenBLAS_LIBNAME})

#######
if(MSVC)
    option(BUILD_WITHOUT_LAPACK "Without LAPACK and LAPACKE (Only BLAS or CBLAS)" ON)
endif()
option(BUILD_WITHOUT_CBLAS "Without CBLAS" OFF)
option(BUILD_DEBUG "Build Debug Version" OFF)
#######
if(BUILD_WITHOUT_LAPACK)
    set(NO_LAPACK 1)
    set(NO_LAPACKE 1)
endif()

if(CMAKE_CONFIGURATION_TYPES) # multiconfig generator?
    set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "" FORCE)
    set(CMAKE_BUILD_TYPE
        Debug Debug
	Release Release
	)
else()
	if( NOT CMAKE_BUILD_TYPE )
		if(BUILD_DEBUG)
			set(CMAKE_BUILD_TYPE Debug)
		else()
			set(CMAKE_BUILD_TYPE Release)
		endif()
	endif()
endif()

if(BUILD_WITHOUT_CBLAS)
    set(NO_CBLAS 1)
endif()

#######

include(system)

set(BLASDIRS interface driver/level2 driver/level3 driver/others)

if (NOT DYNAMIC_ARCH)
    list(APPEND BLASDIRS kernel)
endif ()

if (DEFINED SANITY_CHECK)
    list(APPEND BLASDIRS reference)
endif ()

set(SUBDIRS	${BLASDIRS})
if (NOT NO_LAPACK)
    list(APPEND SUBDIRS lapack)
endif ()

# set which float types we want to build for
if (NOT DEFINED BUILD_SINGLE AND NOT DEFINED BUILD_DOUBLE AND NOT DEFINED BUILD_COMPLEX AND NOT DEFINED BUILD_COMPLEX16)
    # if none are defined, build for all
    set(BUILD_SINGLE true)
    set(BUILD_DOUBLE true)
    set(BUILD_COMPLEX true)
    set(BUILD_COMPLEX16 true)
endif ()

set(FLOAT_TYPES "")
if(BUILD_SINGLE)
    message(STATUS "Building Single Precision")
    list(APPEND FLOAT_TYPES "SINGLE") # defines nothing
endif()

if (BUILD_DOUBLE)
    message(STATUS "Building Double Precision")
    list(APPEND FLOAT_TYPES "DOUBLE") # defines DOUBLE
endif ()

if (BUILD_COMPLEX)
    message(STATUS "Building Complex Precision")
    list(APPEND FLOAT_TYPES "COMPLEX") # defines COMPLEX
endif ()

if (BUILD_COMPLEX16)
    message(STATUS "Building Double Complex Precision")
    list(APPEND FLOAT_TYPES "ZCOMPLEX") # defines COMPLEX and DOUBLE
endif ()

# libs :
if (NOT DEFINED CORE OR "${CORE}" STREQUAL "UNKNOWN")
    message(FATAL_ERROR "Detecting CPU failed. Please set TARGET explicitly, e.g. make TARGET=your_cpu_target. Please read README for details.")
endif ()

if (${NO_STATIC} AND ${NO_SHARED})
    message(FATAL_ERROR "Neither static nor shared are enabled.")
endif ()

#Set default output directory
set( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
if(MSVC)
    set( CMAKE_LIBRARY_OUTPUT_DIRECTORY_DEBUG ${PROJECT_BINARY_DIR}/lib/Debug)
    set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${PROJECT_BINARY_DIR}/lib/Release)
endif ()
# get obj vars into format that add_library likes: $<TARGET_OBJS:objlib> (see http://www.cmake.org/cmake/help/v3.0/command/add_library.html)
set(TARGET_OBJS "")
foreach (SUBDIR ${SUBDIRS})
    add_subdirectory(${SUBDIR})
    string(REPLACE "/" "_" subdir_obj ${SUBDIR})
    list(APPEND TARGET_OBJS "$<TARGET_OBJECTS:${subdir_obj}>")
endforeach ()

# netlib:

# Can't just use lapack-netlib's CMake files, since they are set up to search for BLAS, build and install a binary. We just want to build a couple of lib files out of lapack and lapacke.
# Not using add_subdirectory here because lapack-netlib already has its own CMakeLists.txt. Instead include a cmake script with the sources we want.
if (NOT NOFORTRAN AND NOT NO_LAPACK)
    include("${PROJECT_SOURCE_DIR}/cmake/lapack.cmake")
    if (NOT NO_LAPACKE)
        include("${PROJECT_SOURCE_DIR}/cmake/lapacke.cmake")
    endif ()
endif ()

# Only generate .def for dll on MSVC and always produce pdb files for debug and release
if(MSVC)
    set(OpenBLAS_DEF_FILE "${PROJECT_BINARY_DIR}/openblas.def")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /Zi")
    set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} /DEBUG /OPT:REF /OPT:ICF")
endif()

include("${PROJECT_SOURCE_DIR}/cmake/export.cmake")

enable_testing()
add_subdirectory(utest)

# Install include files
set (GENCONFIG_BIN ${CMAKE_BINARY_DIR}/gen_config_h${CMAKE_EXECUTABLE_SUFFIX})
add_custom_command(
    OUTPUT ${CMAKE_BINARY_DIR}/openblas_config.h
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/config.h
    COMMAND ${GENCONFIG_BIN} ${CMAKE_CURRENT_SOURCE_DIR}/config.h ${CMAKE_CURRENT_SOURCE_DIR}/openblas_config_template.h > ${CMAKE_BINARY_DIR}/openblas_config.h
)

add_custom_target(genconfig ALL DEPENDS openblas_config.h)
add_dependencies(genconfig ${OpenBLAS_LIBNAME})

if(BUILD_SHARED_LIBS)
    add_library(${LIB_NAME} SHARED ${LA_SOURCES} ${LAPACKE_SOURCES} ${TARGET_OBJS} ${OpenBLAS_DEF_FILE})
    if(MSVC)
        set_target_properties(${LIB_NAME} PROPERTIES IMPORT_SUFFIX "_i.lib")
    endif()
elseif(OSX_FRAMEWORK)
    add_library(${LIB_NAME} SHARED ${LA_SOURCES} ${LAPACKE_SOURCES} ${TARGET_OBJS} ${OpenBLAS_DEF_FILE} ${CMAKE_BINARY_DIR}/openblas_config.h)
    set_target_properties(${LIB_NAME} PROPERTIES
      FRAMEWORK TRUE
      FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
      MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
      MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
      MACOSX_FRAMEWORK_IDENTIFIER org.openblas.openblas
      XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
      PUBLIC_HEADER "${CMAKE_BINARY_DIR}/openblas_config.h")
else()
    add_library(${LIB_NAME} STATIC ${LA_SOURCES} ${LAPACKE_SOURCES} ${TARGET_OBJS} ${OpenBLAS_DEF_FILE})
    if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
        set( CMAKE_CXX_FLAGS "-fpic ${CMAKE_CXX_FLAGS}" )
        set( CMAKE_C_FLAGS   "-fpic ${CMAKE_C_FLAGS}" )
    endif()
endif()

set_target_properties(${LIB_NAME} PROPERTIES
    VERSION ${OpenBLAS_MAJOR_VERSION}.${OpenBLAS_MINOR_VERSION}
    SOVERSION ${OpenBLAS_MAJOR_VERSION}
    RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib
    LIBRARY_OUTPUT_NAME_DEBUG "${OpenBLAS_LIBNAME}_d"
)

foreach (OUTPUTCONFIG ${CMAKE_CONFIGURATION_TYPES})
  string( TOUPPER ${OUTPUTCONFIG} OUTPUTCONFIG )

  set_target_properties( ${OpenBLAS_LIBNAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${PROJECT_BINARY_DIR}/lib/${OUTPUTCONFIG} )
  set_target_properties( ${OpenBLAS_LIBNAME} PROPERTIES LIBRARY_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${PROJECT_BINARY_DIR}/lib/${OUTPUTCONFIG} )
  set_target_properties( ${OpenBLAS_LIBNAME} PROPERTIES ARCHIVE_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${PROJECT_BINARY_DIR}/lib/${OUTPUTCONFIG} )
endforeach()

include("${PROJECT_SOURCE_DIR}/cmake/export.cmake")

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

# Install
if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(TARGETS ${LIB_NAME}
        EXPORT ${PACKAGE_UPPER_NAME}Targets
        RUNTIME DESTINATION ${INSTALL_BIN_DIR} # at least for dlls
        ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
        LIBRARY DESTINATION ${INSTALL_LIB_DIR}
        INCLUDES DESTINATION ${INSTALL_INC_DIR}
        FRAMEWORK DESTINATION ${INSTALL_LIB_DIR})
endif()

if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL)
    install(FILES ${CMAKE_BINARY_DIR}/openblas_config.h DESTINATION ${INSTALL_INC_DIR})

    file(READ ${CMAKE_CURRENT_SOURCE_DIR}/common_interface.h _COMMON_INTERFACE_H_CONTENTS)
    file(WRITE ${CMAKE_BINARY_DIR}/f77blas.h "#ifndef OPENBLAS_F77BLAS_H
#define OPENBLAS_F77BLAS_H
#include \"openblas_config.h\"
${_COMMON_INTERFACE_H_CONTENTS}
#endif
")
	install (FILES ${CMAKE_BINARY_DIR}/f77blas.h DESTINATION ${INSTALL_INC_DIR})

    if(NOT NO_CBLAS)
        file(READ ${CMAKE_CURRENT_SOURCE_DIR}/cblas.h _CBLAS_H_CONTENTS)
        string(REPLACE "common.h" "openblas_config.h" _CBLAS_H_CONTENTS ${_CBLAS_H_CONTENTS})
        file(WRITE ${CMAKE_BINARY_DIR}/cblas.h "${_CBLAS_H_CONTENTS}")
    	install(FILES ${CMAKE_BINARY_DIR}/cblas.h DESTINATION ${INSTALL_INC_DIR})
    endif()

    if(NOT NO_LAPACKE)
    	file(GLOB_RECURSE INCLUDE_FILES "${CMAKE_CURRENT_SOURCE_DIR}/lapack-netlib/LAPACKE/*.h")
    	install(FILES ${INCLUDE_FILES} DESTINATION ${INSTALL_INC_DIR})

    	configure_file(
            ${CMAKE_CURRENT_SOURCE_DIR}/lapack-netlib/LAPACKE/include/lapacke_mangling_with_flags.h.in
            "${CMAKE_BINARY_DIR}/lapacke_mangling.h"
        	COPYONLY)
    	install(FILES ${CMAKE_BINARY_DIR}/lapacke_mangling.h DESTINATION ${INSTALL_INC_DIR})
    endif()
endif()

# Uninstall
add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)


# Export package ===============================================================

# Add path to includes to build-tree export
target_include_directories(${LIB_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
)

# Add all targets to the build-tree export set
export(TARGETS ${LIB_NAME}
    FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

if(REGISTER_PACKAGE)
    # Export the package for use from the build-tree
    # (this registers the build-tree with a global CMake-registry)
    export(PACKAGE ${PACKAGE_UPPER_NAME})
endif()

# Create the ZLIBConfig.cmake file
configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    # Install the <Package>Config.cmake
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
