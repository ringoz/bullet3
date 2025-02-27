cmake_minimum_required(VERSION 2.8.12)
set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS true)
cmake_policy(SET CMP0017 NEW)

PROJECT(BULLET_PHYSICS)
FILE (STRINGS "VERSION" BULLET_VERSION)

IF(COMMAND cmake_policy)
   cmake_policy(SET CMP0003 NEW)
   if(POLICY CMP0042)
      # Enable MACOSX_RPATH by default.
      cmake_policy(SET CMP0042 NEW)
   endif(POLICY CMP0042)
ENDIF(COMMAND cmake_policy)

IF (NOT CMAKE_BUILD_TYPE)
# SET(CMAKE_BUILD_TYPE "Debug")
 SET(CMAKE_BUILD_TYPE "Release")
ENDIF (NOT CMAKE_BUILD_TYPE)

SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -D_DEBUG")
#MESSAGE("CMAKE_CXX_FLAGS_DEBUG="+${CMAKE_CXX_FLAGS_DEBUG})

OPTION(USE_DOUBLE_PRECISION "Use double precision"	OFF)
SET(CLAMP_VELOCITIES "0" CACHE STRING "Clamp rigid bodies' velocity to this value, if larger than zero. Useful to prevent floating point errors or in general runaway velocities in complex scenarios")
OPTION(USE_GRAPHICAL_BENCHMARK "Use Graphical Benchmark" ON)
OPTION(BUILD_SHARED_LIBS "Use shared libraries" OFF)
OPTION(USE_SOFT_BODY_MULTI_BODY_DYNAMICS_WORLD "Use btSoftMultiBodyDynamicsWorld" ON)
OPTION(USE_OPENVR "Use OpenVR for virtual reality" OFF)

OPTION(ENABLE_VHACD "Use VHACD in BulletRobotics and pybullet" ON)

OPTION(BULLET2_MULTITHREADING "Build Bullet 2 libraries with mutex locking around certain operations (required for multi-threading)" OFF)
IF (BULLET2_MULTITHREADING)
    OPTION(BULLET2_USE_OPEN_MP_MULTITHREADING "Build Bullet 2 with support for multi-threading with OpenMP (requires a compiler with OpenMP support)" OFF)
    OPTION(BULLET2_USE_TBB_MULTITHREADING "Build Bullet 2 with support for multi-threading with Intel Threading Building Blocks (requires the TBB library to be already installed)" OFF)
    IF (MSVC)
        OPTION(BULLET2_USE_PPL_MULTITHREADING "Build Bullet 2 with support for multi-threading with Microsoft Parallel Patterns Library (requires MSVC compiler)" OFF)
    ENDIF (MSVC)
ENDIF (BULLET2_MULTITHREADING)


IF(NOT WIN32)
	SET(DL ${CMAKE_DL_LIBS})
	IF(CMAKE_SYSTEM_NAME MATCHES "Linux")
		MESSAGE("Linux")
		SET(OSDEF -D_LINUX)
	ELSE(CMAKE_SYSTEM_NAME MATCHES "Linux")
		IF(APPLE)
			MESSAGE("Apple")
			SET(OSDEF -D_DARWIN)
		ELSE(APPLE)
			MESSAGE("BSD?")
			SET(OSDEF -D_BSD)
		ENDIF(APPLE)
	ENDIF(CMAKE_SYSTEM_NAME MATCHES "Linux")
ENDIF(NOT WIN32)

if(MSVC)
  add_compile_options(/EHs-c- /GR-)
else()
  add_compile_options(-fno-exceptions -fno-rtti)
endif()


OPTION(BUILD_CPU_DEMOS "Build original Bullet CPU examples" ON)



OPTION(INTERNAL_UPDATE_SERIALIZATION_STRUCTURES "Internal update serialization structures" OFF)
IF (INTERNAL_UPDATE_SERIALIZATION_STRUCTURES)
ADD_DEFINITIONS( -DBT_INTERNAL_UPDATE_SERIALIZATION_STRUCTURES)
ENDIF (INTERNAL_UPDATE_SERIALIZATION_STRUCTURES)

IF (CLAMP_VELOCITIES)
ADD_DEFINITIONS( -DBT_CLAMP_VELOCITY_TO=${CLAMP_VELOCITIES})
ENDIF (CLAMP_VELOCITIES)

IF (USE_DOUBLE_PRECISION)
ADD_DEFINITIONS( -DBT_USE_DOUBLE_PRECISION)
SET( BULLET_DOUBLE_DEF "-DBT_USE_DOUBLE_PRECISION")
ENDIF (USE_DOUBLE_PRECISION)

IF (NOT USE_SOFT_BODY_MULTI_BODY_DYNAMICS_WORLD)
ADD_DEFINITIONS(-DSKIP_SOFT_BODY_MULTI_BODY_DYNAMICS_WORLD)
ENDIF ()

IF(USE_GRAPHICAL_BENCHMARK)
ADD_DEFINITIONS( -DUSE_GRAPHICAL_BENCHMARK)
ENDIF (USE_GRAPHICAL_BENCHMARK)

IF(BULLET2_MULTITHREADING)
	ADD_DEFINITIONS( -DBT_THREADSAFE=1 )
	IF (NOT MSVC)
		SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
	ENDIF (NOT MSVC)
	IF (NOT WIN32)
		FIND_PACKAGE(Threads)
		LINK_LIBRARIES( ${CMAKE_THREAD_LIBS_INIT} )
	ENDIF (NOT WIN32)
ENDIF (BULLET2_MULTITHREADING)

IF (BULLET2_USE_OPEN_MP_MULTITHREADING)
    ADD_DEFINITIONS("-DBT_USE_OPENMP=1")
    IF (MSVC)
        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /openmp")
    ELSE (MSVC)
        # GCC, Clang
        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fopenmp")
    ENDIF (MSVC)
ENDIF (BULLET2_USE_OPEN_MP_MULTITHREADING)

IF (BULLET2_USE_TBB_MULTITHREADING)
    SET (BULLET2_TBB_INCLUDE_DIR "not found" CACHE PATH "Directory for Intel TBB includes.")
    SET (BULLET2_TBB_LIB_DIR "not found" CACHE PATH "Directory for Intel TBB libraries.")
    find_library(TBB_LIBRARY tbb PATHS ${BULLET2_TBB_LIB_DIR})
    find_library(TBBMALLOC_LIBRARY tbbmalloc PATHS ${BULLET2_TBB_LIB_DIR})
    ADD_DEFINITIONS("-DBT_USE_TBB=1")
    INCLUDE_DIRECTORIES( ${BULLET2_TBB_INCLUDE_DIR} )
    LINK_LIBRARIES( ${TBB_LIBRARY} ${TBBMALLOC_LIBRARY} )
ENDIF (BULLET2_USE_TBB_MULTITHREADING)

IF (BULLET2_USE_PPL_MULTITHREADING)
    ADD_DEFINITIONS("-DBT_USE_PPL=1")
ENDIF (BULLET2_USE_PPL_MULTITHREADING)

IF(COMMAND cmake_policy)
   cmake_policy(SET CMP0003 NEW)
ENDIF(COMMAND cmake_policy)


OPTION(BUILD_BULLET3 "Set when you want to build Bullet 3" ON)

# Optional Python configuration
# Will not probe environment for Python configuration (which can abort the
# build process) unless you explicitly turn on BUILD_PYBULLET.
OPTION(BUILD_PYBULLET "Set when you want to build pybullet (Python bindings for Bullet)" OFF)
IF(BUILD_PYBULLET)
	SET(PYTHON_VERSION_PYBULLET "" CACHE STRING "Python version pybullet will use.")
	SET(Python_ADDITIONAL_VERSIONS 3 3.6 3.5 3.4 3.3 3.2 3.1 3.0 2.7 2.7.12 2.7.10 2.7.3 )
	SET_PROPERTY(CACHE PYTHON_VERSION_PYBULLET PROPERTY STRINGS ${Python_ADDITIONAL_VERSIONS})
	SET(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/build3/cmake ${CMAKE_MODULE_PATH})
	OPTION(EXACT_PYTHON_VERSION "Require Python and match PYTHON_VERSION_PYBULLET exactly, e.g. 2.7.12" OFF)
	IF(EXACT_PYTHON_VERSION)
	    set(EXACT_PYTHON_VERSION_FLAG EXACT REQUIRED)
	ENDIF(EXACT_PYTHON_VERSION)
	# first find the python interpreter
	FIND_PACKAGE(PythonInterp ${PYTHON_VERSION_PYBULLET} ${EXACT_PYTHON_VERSION_FLAG})
	# python library should exactly match that of the interpreter
	# the following can result in fatal error if you don't have the right python configuration
	FIND_PACKAGE(PythonLibs ${PYTHON_VERSION_STRING} EXACT)
ENDIF(BUILD_PYBULLET)

OPTION(BUILD_ENET "Set when you want to build apps with enet UDP networking support" ON)
OPTION(BUILD_CLSOCKET "Set when you want to build apps with enet TCP networking support" ON)


IF(BUILD_PYBULLET)
	FIND_PACKAGE(PythonLibs)

	OPTION(BUILD_PYBULLET_NUMPY "Set when you want to build pybullet with NumPy support" OFF)
	OPTION(BUILD_PYBULLET_ENET "Set when you want to build pybullet with enet UDP networking support" ON)
	OPTION(BUILD_PYBULLET_CLSOCKET "Set when you want to build pybullet with enet TCP networking support" ON)

	OPTION(BUILD_PYBULLET_MAC_USE_PYTHON_FRAMEWORK "Set when you want to use the Python Framework on Mac" OFF)

	IF(BUILD_PYBULLET_NUMPY)
		#include(FindNumPy)
		FIND_PACKAGE(NumPy)
		if (PYTHON_NUMPY_FOUND)
			message("NumPy found")
			add_definitions(-DPYBULLET_USE_NUMPY)
		else()
			message("NumPy not found")
		endif()
	ENDIF()

	IF(WIN32)
		SET(BUILD_SHARED_LIBS OFF CACHE BOOL "Shared Libs" FORCE)
	ELSE(WIN32)
		SET(BUILD_SHARED_LIBS ON CACHE BOOL "Shared Libs" FORCE)
	ENDIF(WIN32)

	IF(APPLE)
		OPTION(BUILD_PYBULLET_MAC_USE_PYTHON_FRAMEWORK "Set when you want to use the Python Framework on Mac" ON)
		IF(NOT BUILD_PYBULLET_MAC_USE_PYTHON_FRAMEWORK)
			add_definitions(-DB3_NO_PYTHON_FRAMEWORK)
		ENDIF(NOT BUILD_PYBULLET_MAC_USE_PYTHON_FRAMEWORK)
		OPTION(BUILD_PYBULLET_SHOW_PY_VERSION "Set when you want to show the PY_MAJOR_VERSION and PY_MAJOR_VERSION using #pragme message." OFF)
		IF(BUILD_PYBULLET_SHOW_PY_VERSION)
			add_definitions(-DB3_DUMP_PYTHON_VERSION)
		ENDIF()

	ENDIF(APPLE)

ENDIF(BUILD_PYBULLET)

IF(NOT WIN32 AND NOT APPLE)
    OPTION(BUILD_EGL "Build OpenGL/EGL" ON)
    IF(BUILD_EGL)
        ADD_DEFINITIONS(-DBT_USE_EGL)
    ENDIF(BUILD_EGL)
ENDIF()

IF(BUILD_BULLET3)
	 IF(APPLE)
		MESSAGE("Mac OSX Version is ${_CURRENT_OSX_VERSION}")
		IF(_CURRENT_OSX_VERSION  VERSION_LESS 10.9)
			MESSAGE("Mac OSX below 10.9 has no OpenGL 3 support so please disable the BUILD_OPENGL3_DEMOS option")
			#unset(BUILD_OPENGL3_DEMOS CACHE)

			OPTION(BUILD_OPENGL3_DEMOS "Set when you want to build the OpenGL3+ demos" OFF)
		ELSE()
			OPTION(BUILD_OPENGL3_DEMOS "Set when you want to build the OpenGL3+ demos" ON)
		ENDIF()
	ELSE()
		OPTION(BUILD_OPENGL3_DEMOS "Set when you want to build Bullet 3 OpenGL3+ demos" ON)
	ENDIF()
ELSE(BUILD_BULLET3)
	unset(BUILD_OPENGL3_DEMOS CACHE)
	OPTION(BUILD_OPENGL3_DEMOS "Set when you want to build Bullet 3 OpenGL3+ demos" OFF)
ENDIF(BUILD_BULLET3)
IF(BUILD_OPENGL3_DEMOS)
	IF(EXISTS ${BULLET_PHYSICS_SOURCE_DIR}/Demos3 AND IS_DIRECTORY ${BULLET_PHYSICS_SOURCE_DIR}/Demos3)
		SUBDIRS(Demos3)
	ENDIF()
ELSE()
	ADD_DEFINITIONS(-DNO_OPENGL3)
ENDIF(BUILD_OPENGL3_DEMOS)

OPTION(BUILD_BULLET2_DEMOS "Set when you want to build the Bullet 2 demos" ON)
IF(BUILD_BULLET2_DEMOS)

	IF(EXISTS ${BULLET_PHYSICS_SOURCE_DIR}/examples AND IS_DIRECTORY ${BULLET_PHYSICS_SOURCE_DIR}/examples)
		SUBDIRS(examples)
	ENDIF()

ENDIF(BUILD_BULLET2_DEMOS)



OPTION(BUILD_EXTRAS "Set when you want to build the extras" ON)
IF(BUILD_EXTRAS)
  SUBDIRS(Extras)
ENDIF(BUILD_EXTRAS)



SUBDIRS(src)

IF("${CMAKE_GENERATOR}" MATCHES "Unix Makefiles")
	OPTION(INSTALL_LIBS "Set when you want to install libraries" ON)
ELSE()
	IF(APPLE AND FRAMEWORK)
		OPTION(INSTALL_LIBS "Set when you want to install libraries" ON)
	ELSE()
#by default, don't enable the 'INSTALL' option for Xcode and MSVC projectfiles
		OPTION(INSTALL_LIBS "Set when you want to install libraries" OFF)
	ENDIF()
ENDIF()


IF(INSTALL_LIBS)
	#INSTALL of other files requires CMake 2.6
	IF(BUILD_EXTRAS)
		IF (${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} GREATER 2.5)
			OPTION(INSTALL_EXTRA_LIBS "Set when you want extra libraries installed" ON)
		ENDIF (${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} GREATER 2.5)
	ENDIF(BUILD_EXTRAS)

	SET (LIB_SUFFIX "" CACHE STRING "Define suffix of directory name (32/64)" )
	SET (LIB_DESTINATION "lib${LIB_SUFFIX}" CACHE STRING "Library directory name")
	## the following are directories where stuff will be installed to
	SET(INCLUDE_INSTALL_DIR "include/bullet/" CACHE PATH "The subdirectory to the header prefix")
	SET(PKGCONFIG_INSTALL_PREFIX "lib${LIB_SUFFIX}/pkgconfig/" CACHE STRING "Base directory for pkgconfig files")
	IF(NOT MSVC)
	  CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/bullet.pc.cmake ${CMAKE_CURRENT_BINARY_DIR}/bullet.pc @ONLY)
  	INSTALL(
		FILES
		${CMAKE_CURRENT_BINARY_DIR}/bullet.pc
		DESTINATION
		${PKGCONFIG_INSTALL_PREFIX})
	ENDIF(NOT MSVC)
ENDIF()


OPTION(BUILD_UNIT_TESTS "Build Unit Tests"	ON)

IF (BUILD_UNIT_TESTS)
	ENABLE_TESTING()
	SUBDIRS(test)
ENDIF()

set (BULLET_CONFIG_CMAKE_PATH lib${LIB_SUFFIX}/cmake/bullet )
list (APPEND BULLET_DEFINITIONS ${BULLET_DOUBLE_DEF})
list (APPEND BULLET_LIBRARIES LinearMath)
list (APPEND BULLET_LIBRARIES Bullet3Common)
list (APPEND BULLET_LIBRARIES BulletInverseDynamics)
list (APPEND BULLET_LIBRARIES BulletCollision)
list (APPEND BULLET_LIBRARIES BulletDynamics)
list (APPEND BULLET_LIBRARIES BulletSoftBody)
set (BULLET_USE_FILE ${BULLET_CONFIG_CMAKE_PATH}/UseBullet.cmake)
configure_file 	( ${CMAKE_CURRENT_SOURCE_DIR}/BulletConfig.cmake.in
					${CMAKE_CURRENT_BINARY_DIR}/BulletConfig.cmake
					@ONLY ESCAPE_QUOTES
				)
OPTION(INSTALL_CMAKE_FILES "Install generated CMake files" ON)

IF (INSTALL_CMAKE_FILES)
	install ( FILES ${CMAKE_CURRENT_SOURCE_DIR}/UseBullet.cmake
		${CMAKE_CURRENT_BINARY_DIR}/BulletConfig.cmake
		DESTINATION ${BULLET_CONFIG_CMAKE_PATH}
	)
ENDIF (INSTALL_CMAKE_FILES)
