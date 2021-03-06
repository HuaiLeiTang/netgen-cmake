# -------------------------------------------------------------------------------------------------
# Netgen Mesher CMake based Build System
# Author: Philippose Rajan
# Date started: 18 Sept. 2011
# Modified: 12 October 2012
#
# REQUIREMENTS: this script needs CMake 2.7 or higher. Downloads are available at http://www.cmake.org
#
# The CMake Build System is configured to work with 32bit/64bit versions of Linux and Windows 
# operating systems, and simplifies the maintenance of cross-platform builds for the Netgen Mesher 
# Project
# -------------------------------------------------------------------------------------------------

#############################################
# Basic initialisation for the CMake System #
#############################################
# Set the minimum version of cmake required to 2.7
CMAKE_MINIMUM_REQUIRED(VERSION 2.8.11 FATAL_ERROR)

# Make sure that the current directory is also included 
# into the list of includes
set(CMAKE_INCLUDE_CURRENT_DIR ON)

# if CMAKE_BUILD_TYPE is specified use it; otherwise set the default
# build type to "Release prior to calling project()
if(DEFINED CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE} CACHE STRING "Choose build type")
else(DEFINED CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose build type")
endif(DEFINED CMAKE_BUILD_TYPE)


########################
# Set the Project Name #
########################
PROJECT(netgen)

##########################################
# Setup the Version Number for the build #
##########################################
SET(NETGEN_VERSION_MAJOR 5)
SET(NETGEN_VERSION_MINOR 2)
SET(NETGEN_VERSION_PATCH 0)
# Note: Empty for official releases, set to -dev, -rc1, etc for development releases
SET(NETGEN_VERSION_DEVEL -dev)

SET(NETGEN_VERSION ${NETGEN_VERSION_MAJOR}.${NETGEN_VERSION_MINOR}.${NETGEN_VERSION_PATCH}${NETGEN_VERSION_DEVEL})
MESSAGE(STATUS "Netgen Version: ${NETGEN_VERSION}")

# This is used for Linux .so file versioning
SET(NETGEN_LIB_SOVERSION ${NETGEN_VERSION_MAJOR})
SET(NETGEN_LIB_VERSION ${NETGEN_VERSION_MAJOR}.0.0)

############################
# Disallow in-source build #
############################
IF("${PROJECT_SOURCE_DIR}" STREQUAL "${PROJECT_BINARY_DIR}")
	MESSAGE(FATAL_ERROR "Netgen requires an out of source Build. Please create a separate build directory and run CMake there.")
ENDIF("${PROJECT_SOURCE_DIR}" STREQUAL "${PROJECT_BINARY_DIR}")


# Add the custom CMake Modules folder to the search path
SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/modules ${CMAKE_MODULE_PATH})
INCLUDE(AuxiliaryUtilities)
INCLUDE(CMakeDependentOption)

############################
# Check 32/64 bit platform #
############################
IF (${CMAKE_SIZEOF_VOID_P} MATCHES "8") # It is 64bit, otherwise 32 bit systems match 4
	SET(BIT_WIDTH 64)
ELSE (${CMAKE_SIZEOF_VOID_P} MATCHES "8")
	SET(BIT_WIDTH 32)
ENDIF(${CMAKE_SIZEOF_VOID_P} MATCHES "8")

MESSAGE(STATUS "Build type: ${BIT_WIDTH}bit")


#####################################################
# Specify the various options available to the user #
#####################################################
OPTION_WITH_DEFAULT(${PROJECT_NAME}_BUILD_GUI "Build the Graphical User Interface for Netgen" ON)
OPTION_WITH_DEFAULT(${PROJECT_NAME}_BUILD_NGLIB_API "Build independent Netgen Library - nglib" ON)
OPTION_WITH_DEFAULT(${PROJECT_NAME}_ENABLE_OPENCASCADE_SUPPORT "Build with OpenCascade support (for STEP, IGES support)" OFF)
OPTION_WITH_DEFAULT(${PROJECT_NAME}_ENABLE_STL_SUPPORT "Build with STL support (for STL, STLB support)" ON)
OPTION_WITH_DEFAULT(${PROJECT_NAME}_ENABLE_PARALLEL_SUPPORT "Build with MPI Parallelization support" OFF)
OPTION_WITH_DEFAULT(${PROJECT_NAME}_USE_JPEGLIB "Use external JPEG Library to create snapshots" OFF)
OPTION_WITH_DEFAULT(${PROJECT_NAME}_USE_FFMPEG "Enable video recording with FFMPEG (uses avcodec)" OFF)
OPTION_WITH_DEFAULT(${PROJECT_NAME}_ENABLE_EXTRA_WARNINGS "Enable Maximum Warnings Level" OFF)


# As of now, the MSVC build can handle only static builds, and the GCC build can handle only shared builds !!!
IF(MSVC OR MSVC_IDE)
	SET(${PROJECT_NAME}_ENABLE_STATIC_BUILD ON CACHE INTERNAL "Statically link all the Geometry Kernels into the ${PROJECT_NAME} Executable" FORCE)
ELSE(MSVC OR MSVC_IDE)
	OPTION_WITH_DEFAULT(${PROJECT_NAME}_ENABLE_STATIC_BUILD "Statically link all the Geometry Kernels into the ${PROJECT_NAME} Executable" OFF)
ENDIF(MSVC OR MSVC_IDE)	

CMAKE_DEPENDENT_OPTION(${PROJECT_NAME}_BUILD_STL_PLUGIN 
                       "Build DLL plugin for STL Geometry support" ON
					   "NOT ${PROJECT_NAME}_ENABLE_STATIC_BUILD;${PROJECT_NAME}_ENABLE_STL_SUPPORT"
					   OFF
					   )

CMAKE_DEPENDENT_OPTION(${PROJECT_NAME}_BUILD_OPENCASCADE_PLUGIN 
                       "Build DLL plugin for IGES/STEP/BREP Geometry support" ON
					   "NOT ${PROJECT_NAME}_ENABLE_STATIC_BUILD;${PROJECT_NAME}_ENABLE_OPENCASCADE_SUPPORT"
					   OFF
					   )

					   
#################################
# Windows MSVC specific options #
#################################
IF(MSVC_IDE)
    OPTION_WITH_DEFAULT(${PROJECT_NAME}_MSVC_EXPRESS_IDE "Use a Visual C++ Express Edition IDE for compiling ${PROJECT_NAME}" OFF)
ENDIF(MSVC_IDE)

# Thread specific options
IF(NOT DEFINED ${PROJECT_NAME}_THREAD_LIBRARY)
	IF(WIN32)
		IF(${PROJECT_NAME}_MSVC_EXPRESS_IDE)
			SET(${PROJECT_NAME}_THREAD_LIBRARY "pthreads-w32" CACHE STRING "(values are: pthreads-w32)" FORCE)
		ELSE(${PROJECT_NAME}_MSVC_EXPRESS_IDE)
			SET(${PROJECT_NAME}_THREAD_LIBRARY "MFC-Threads" CACHE STRING "(values are: pthreads-w32 MFC-Threads)" FORCE)
		ENDIF(${PROJECT_NAME}_MSVC_EXPRESS_IDE)
	ELSE(WIN32)
		SET(${PROJECT_NAME}_THREAD_LIBRARY "pthreads" CACHE STRING "(values are: pthreads)" FORCE)	
	ENDIF(WIN32)
ENDIF(NOT DEFINED ${PROJECT_NAME}_THREAD_LIBRARY)

IF(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} GREATER 2.7)
	IF(WIN32)
		SET_PROPERTY(CACHE ${PROJECT_NAME}_THREAD_LIBRARY PROPERTY STRINGS pthreads-w32 MFC-Threads)
	ELSE(WIN32)
		SET_PROPERTY(CACHE ${PROJECT_NAME}_THREAD_LIBRARY PROPERTY STRINGS pthreads)
	ENDIF(WIN32)
ENDIF(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} GREATER 2.7)

IF(${PROJECT_NAME}_MSVC_EXPRESS_IDE)
	SET(${PROJECT_NAME}_THREAD_LIBRARY "pthreads-w32" CACHE STRING "only pthreads-w32 available for Express Editions" FORCE)
ENDIF(${PROJECT_NAME}_MSVC_EXPRESS_IDE)
	
OPTION(${PROJECT_NAME}_THREAD_LIBRARY "Multithread library" ${${PROJECT_NAME}_THREAD_LIBRARY})

# Options for the Install Location 
IF(NOT DEFINED ${PROJECT_NAME}_INSTALL_PREFIX )
	IF(WIN32)
		SET(${PROJECT_NAME}_INSTALL_PREFIX "c:/${PROJECT_NAME}-${NETGEN_VERSION}")
	ELSE(WIN32)
		SET(${PROJECT_NAME}_INSTALL_PREFIX "/opt/${PROJECT_NAME}")
	ENDIF(WIN32)
ENDIF(NOT DEFINED ${PROJECT_NAME}_INSTALL_PREFIX)

SET(${PROJECT_NAME}_INSTALL_PREFIX ${${PROJECT_NAME}_INSTALL_PREFIX} CACHE
	PATH "Install prefix prepended to target to create install location" )
SET(CMAKE_INSTALL_PREFIX "${${PROJECT_NAME}_INSTALL_PREFIX}" CACHE INTERNAL "Prefix
	prepended to install directories if target destination is not absolute,
	immutable" FORCE)


MARK_AS_ADVANCED( 
	${PROJECT_NAME}_ENABLE_PARALLEL_SUPPORT 
	${PROJECT_NAME}_USE_JPEGLIB
	${PROJECT_NAME}_USE_FFMPEG
	${PROJECT_NAME}_ENABLE_EXTRA_WARNINGS
	${PROJECT_NAME}_THREAD_LIBRARY
	${PROJECT_NAME}_ENABLE_STATIC_BUILD
)


#####################################################
######### Handle all the options one by one #########
#####################################################


###################################################
# Tell the system that MSVC Express is being used #
###################################################
IF(${PROJECT_NAME}_MSVC_EXPRESS_IDE)
	IF(WIN32)
		ADD_DEFINITIONS("-DMSVC_EXPRESS")
	ENDIF(WIN32)
ENDIF(${PROJECT_NAME}_MSVC_EXPRESS_IDE)


##################################
# Set the compiler warning level #
##################################
IF(${PROJECT_NAME}_ENABLE_EXTRA_WARNINGS)
	SET_WARNING_LEVEL(4)
ELSE(${PROJECT_NAME}_ENABLE_EXTRA_WARNINGS)
	SET_WARNING_LEVEL(0)
ENDIF(${PROJECT_NAME}_ENABLE_EXTRA_WARNINGS)	


###################################################
# Setup whether to use Static or Shared libraries #
# (NOTE: Only for the Geometry components)        #
###################################################
IF(${PROJECT_NAME}_ENABLE_STATIC_BUILD)
	SET(${PROJECT_NAME}_LIBRARY_TYPE STATIC)
ELSE(${PROJECT_NAME}_ENABLE_STATIC_BUILD)
	SET(${PROJECT_NAME}_LIBRARY_TYPE SHARED)
ENDIF(${PROJECT_NAME}_ENABLE_STATIC_BUILD)


IF(${PROJECT_NAME}_BUILD_GUI)
	##################################
	# Try to find the TCL/Tk Package #
	##################################
	FIND_PACKAGE(TCL REQUIRED)
	IF(TCLTK_FOUND)
		LIST(APPEND EXT_LIB_INCLUDES ${TCL_INCLUDE_PATH} ${TK_INCLUDE_PATH})
		LIST(APPEND EXT_LIB_LIBRARIES ${TCL_LIBRARY} ${TK_LIBRARY})
	ENDIF(TCLTK_FOUND)
	

	###############################
	# Try to find the Tix Package #
	###############################
	FIND_PACKAGE(Tix REQUIRED)
	IF(TIX_FOUND)
		LIST(APPEND EXT_LIB_INCLUDES ${TIX_INCLUDE_PATH})
		LIST(APPEND EXT_LIB_LIBRARIES ${TIX_LIBRARY})
	ENDIF(TIX_FOUND)


	##################################
	# Try to find the OpenGL Package #
	##################################
	FIND_PACKAGE(OpenGL REQUIRED)
	IF(OPENGL_FOUND)
		LIST(APPEND EXT_LIB_INCLUDES ${OPENGL_INCLUDE_DIR})
		LIST(APPEND EXT_LIB_LIBRARIES ${OPENGL_LIBRARIES})
		
		ADD_DEFINITIONS("-DOPENGL")
	ENDIF(OPENGL_FOUND)


	####################################
	# Try to find the Togl-1.7 Package #
	####################################
	FIND_PACKAGE(Togl1.7 REQUIRED)
	IF(TOGL1.7_FOUND)
		LIST(APPEND EXT_LIB_INCLUDES ${TOGL1.7_INCLUDE_PATH})
		LIST(APPEND EXT_LIB_LIBRARIES ${TOGL1.7_LIBRARY})
		
		# Setup the Togl Windowing System
		IF(WIN32)
			SET(TOGL_WINDOWING_SYSTEM "TOGL_WGL")
		ELSEIF(APPLE)
			SET(TOGL_WINDOWING_SYSTEM "TOGL_AGL")
		ELSE(WIN32)
			SET(TOGL_WINDOWING_SYSTEM "TOGL_X11")
		ENDIF(WIN32)	
		
		ADD_DEFINITIONS("-D${TOGL_WINDOWING_SYSTEM}")
	ENDIF(TOGL1.7_FOUND)
ELSE(${PROJECT_NAME}_BUILD_GUI)
	UNSET(TCL_INCLUDE_PATH CACHE)
	UNSET(TK_INCLUDE_PATH CACHE)
	UNSET(TIX_INCLUDE_PATH CACHE)
	UNSET(TOGL1.7_INCLUDE_PATH CACHE)
	UNSET(OPENGL_INCLUDE_DIR CACHE)
	
	UNSET(TCL_LIBRARY CACHE)
	UNSET(TK_LIBRARY CACHE)
	UNSET(TIX_LIBRARY CACHE)
	UNSET(TOGL1.7_LIBRARY CACHE)
	UNSET(OPENGL_LIBRARIES CACHE)

	UNSET(TCL_TCLSH CACHE)
	UNSET(TK_WISH CACHE)
ENDIF(${PROJECT_NAME}_BUILD_GUI)


################################
# Try to find the ZLib Package #
################################
FIND_PACKAGE(ZLIB REQUIRED)
IF(ZLIB_FOUND)
	LIST(APPEND EXT_LIB_INCLUDES ${ZLIB_INCLUDE_DIRS})
	LIST(APPEND EXT_LIB_LIBRARIES ${ZLIB_LIBRARIES})
ENDIF(ZLIB_FOUND)


###########################################
# Try to find the correct Threads Package #
###########################################
IF(${PROJECT_NAME}_THREAD_LIBRARY STREQUAL "pthreads")
	FIND_PACKAGE(Threads REQUIRED)
	IF(Threads_FOUND)
		LIST(APPEND EXT_LIB_INCLUDES ${THREADS_PTHREADS_INCLUDE_DIR})
		LIST(APPEND EXT_LIB_LIBRARIES ${CMAKE_THREAD_LIBS_INIT})	
	ENDIF(Threads_FOUND)
ELSEIF(${PROJECT_NAME}_THREAD_LIBRARY STREQUAL "pthreads-w32")
	IF(BIT_WIDTH STREQUAL 64)
		SET(THREADS_PTHREADS_WIN32_SUFFIX "_x64")
	ELSE(BIT_WIDTH STREQUAL 64)
		SET(THREADS_PTHREADS_WIN32_SUFFIX "_Win32")
	ENDIF(BIT_WIDTH STREQUAL 64)
	SET(THREADS_USE_PTHREADS_WIN32 TRUE)
	FIND_PACKAGE(Threads REQUIRED)
	IF(Threads_FOUND)
		LIST(APPEND EXT_LIB_INCLUDES ${THREADS_PTHREADS_INCLUDE_DIR})
		LIST(APPEND EXT_LIB_LIBRARIES ${CMAKE_THREAD_LIBS_INIT})		
	ENDIF(Threads_FOUND)
ELSEIF(${PROJECT_NAME}_THREAD_LIBRARY STREQUAL "MFC-Threads")
	FIND_PACKAGE(Threads REQUIRED)
ENDIF(${PROJECT_NAME}_THREAD_LIBRARY STREQUAL "pthreads")


###############################################################
# Find the location of the OpenCASCADE Includes and libraries #
###############################################################
IF(${PROJECT_NAME}_ENABLE_OPENCASCADE_SUPPORT)
	FIND_PACKAGE(OpenCASCADE REQUIRED TKernel TKGeomBase TKMath TKG2d TKG3d TKXSBase TKOffset TKFillet TKShHealing TKMesh TKMeshVS TKTopAlgo TKGeomAlgo TKBool TKPrim TKBO TKIGES TKBRep TKSTEPBase TKSTEP TKSTL TKSTEPAttr TKSTEP209 TKXDESTEP TKXDEIGES TKXCAF TKLCAF FWOSPlugin)
	IF(OpenCASCADE_FOUND)
		ADD_DEFINITIONS("-DOCCGEOMETRY")
		ADD_DEFINITIONS("-D_USE_MATH_DEFINES")
		IF(WIN32 AND MSVC)
			ADD_DEFINITIONS("-DWNT")
		ENDIF(WIN32 AND MSVC)
		
		LIST(APPEND EXT_LIB_INCLUDES ${OpenCASCADE_INCLUDE_DIR})
		LIST(APPEND EXT_LIB_LIBRARIES ${OpenCASCADE_LIBRARIES})
		
		IF(NOT OpenCASCADE_VERSION)
			TRY_RUN(TRY_RUN_RESULT TRY_RUN_COMPILE 
					${PROJECT_BINARY_DIR} ${PROJECT_SOURCE_DIR}/cmake/modules/checkOpenCASCADEVersion.cpp
					CMAKE_FLAGS -DINCLUDE_DIRECTORIES:STRING=${OpenCASCADE_INCLUDE_DIR}
					COMPILE_OUTPUT_VARIABLE OpenCASCADE_COMPILE_OUT
					RUN_OUTPUT_VARIABLE _testRunOutput
					)
			SET(OpenCASCADE_VERSION "${_testRunOutput}" CACHE INTERNAL "Current OpenCASCADE Version" FORCE)
		ENDIF(NOT OpenCASCADE_VERSION)

		IF(OpenCASCADE_VERSION VERSION_GREATER 6.5)
			ADD_DEFINITIONS("-DHAVE_OCC_65")
		ENDIF(OpenCASCADE_VERSION VERSION_GREATER 6.5)		
		
		MESSAGE(STATUS "OpenCASCADE Version: ${OpenCASCADE_VERSION}")
	ELSE(OpenCASCADE_FOUND)
		MESSAGE(FATAL_ERROR "OpenCASCADE Development System not found....!")
	ENDIF(OpenCASCADE_FOUND)
ENDIF(${PROJECT_NAME}_ENABLE_OPENCASCADE_SUPPORT)	


##############################
# Perform Platform checks to #
# populate the config.h file #
##############################
INCLUDE(CheckIncludeFiles)
CHECK_INCLUDE_FILES(dlfcn.h HAVE_DLFCN_H)
CHECK_INCLUDE_FILES(inttypes.h HAVE_INTTYPES_H)
CHECK_INCLUDE_FILES(limits.h HAVE_LIMITS_H)
CHECK_INCLUDE_FILES(memory.h HAVE_MEMORY_H)
CHECK_INCLUDE_FILES(stdint.h HAVE_STDINT_H)
CHECK_INCLUDE_FILES(stdlib.h HAVE_STDLIB_H)
CHECK_INCLUDE_FILES(strings.h HAVE_STRINGS_H)
CHECK_INCLUDE_FILES(string.h HAVE_STRING_H)
CHECK_INCLUDE_FILES(sys/stat.h HAVE_SYS_STAT_H)
CHECK_INCLUDE_FILES(sys/types.h HAVE_SYS_TYPES_H)
CHECK_INCLUDE_FILES(unistd.h HAVE_UNISTD_H)
CHECK_INCLUDE_FILES("stddef.h;stdint.h" STDC_HEADERS)

INCLUDE(CheckFunctionExists)
CHECK_FUNCTION_EXISTS(floor HAVE_FLOOR)
CHECK_FUNCTION_EXISTS(matherr HAVE_MATHERR)
CHECK_FUNCTION_EXISTS(pow HAVE_POW)

IF(WIN32)
	SET(LT_OBJDIR "_libs/")
ELSE(WIN32)
	SET(LT_OBJDIR ".libs/")
ENDIF(WIN32)

SET(VERSION ${NETGEN_VERSION})
SET(PACKAGE ${PROJECT_NAME})
SET(PACKAGE_VERSION ${NETGEN_VERSION})
SET(PACKAGE_STRING ${PROJECT_NAME}-${NETGEN_VERSION})
CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)
ADD_DEFINITIONS("-DHAVE_CONFIG_H")


# Remove duplicate paths from the include and library lists
LIST(REMOVE_DUPLICATES EXT_LIB_INCLUDES)
LIST(REMOVE_DUPLICATES EXT_LIB_LIBRARIES)


SET(NETGEN_LIBSRC_DIR ${PROJECT_SOURCE_DIR}/libsrc)
SET(NETGEN_APP_DIR ${PROJECT_SOURCE_DIR}/ng)
SET(NETGEN_API_DIR ${PROJECT_SOURCE_DIR}/nglib)
SET(NETGEN_DOC_DIR ${PROJECT_SOURCE_DIR}/doc)
SET(NETGEN_TUT_DIR ${PROJECT_SOURCE_DIR}/tutorials)

SET(NETGEN_BASE_INCLUDES "${NETGEN_LIBSRC_DIR}/include")

SET(NETGEN_BUILD_PATH ${PROJECT_SOURCE_DIR}/cmake/build)

SET(NETGEN_API_NAME "nglib")
SET(NETGEN_ICON_FILE "netgen.ico")

############################################
# Special options for Windows MSVC Targets #
############################################
IF(MSVC OR MSVC_IDE)
	ADD_DEFINITIONS("-DNGINTERFACE_EXPORTS")
	ADD_DEFINITIONS("-DNGLIB_EXPORTS")
	ADD_DEFINITIONS("-Dnglib_EXPORTS")
	ADD_DEFINITIONS("-DStandard_EXPORT=")
	ADD_DEFINITIONS("-D_WIN32_WINNT=0x0502")
ENDIF(MSVC OR MSVC_IDE)


	
############################################
# Special options for GCC based Targets    #
############################################	
IF((${${PROJECT_NAME}_LIBRARY_TYPE} STREQUAL "SHARED") OR ${PROJECT_NAME}_BUILD_NGLIB_API)
	IF(NOT (MSVC OR MSVC_IDE))
		SET(CMAKE_CXX_FLAGS "-fPIC")
	ENDIF(NOT (MSVC OR MSVC_IDE))
ENDIF((${${PROJECT_NAME}_LIBRARY_TYPE} STREQUAL "SHARED") OR ${PROJECT_NAME}_BUILD_NGLIB_API)


	
#############################################################
# Setup the basic include paths valid for the whole project #
#############################################################
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})
INCLUDE_DIRECTORIES(${EXT_LIB_INCLUDES})
INCLUDE_DIRECTORIES(${NETGEN_BASE_INCLUDES})


#################################################
# Setup the external libraries required project #
#################################################
SET(NETGEN_INT_LIBS "" CACHE INTERNAL "Internal Libraries to be linked to the executable" FORCE)
SET(NETGEN_VIS_LIBS "" CACHE INTERNAL "Internal Visualisation Libraries to be linked to the executable" FORCE)
SET(NETGEN_EXE_PATH "" CACHE INTERNAL "Points to the .lib file produced by netgen" FORCE)


############################################
# Create each of the constituent libraries #
############################################
ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/general)
ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/gprim)
ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/linalg)
ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/meshing)
ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/interface)
ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/csg)
ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/geom2d)

IF(${PROJECT_NAME}_BUILD_GUI)
	ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/visualization)
ENDIF(${PROJECT_NAME}_BUILD_GUI)	

IF(${PROJECT_NAME}_ENABLE_STL_SUPPORT)
	ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/stlgeom)
ENDIF(${PROJECT_NAME}_ENABLE_STL_SUPPORT)

IF(${PROJECT_NAME}_ENABLE_OPENCASCADE_SUPPORT AND OpenCASCADE_FOUND)
	ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/libsrc/occ)
ENDIF(${PROJECT_NAME}_ENABLE_OPENCASCADE_SUPPORT AND OpenCASCADE_FOUND)


######################################
# Create the main project executable #
######################################
IF(${PROJECT_NAME}_BUILD_GUI)
	ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/ng)
ENDIF(${PROJECT_NAME}_BUILD_GUI)


##########################
# Create the API Library #
##########################
IF(${PROJECT_NAME}_BUILD_NGLIB_API)
	ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/${NETGEN_API_NAME})
ENDIF(${PROJECT_NAME}_BUILD_NGLIB_API)


####################################
# Handle Documentation and Samples #
####################################
ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/doc)
ADD_SUBDIRECTORY(${NETGEN_BUILD_PATH}/tutorials)

