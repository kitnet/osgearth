CMAKE_MINIMUM_REQUIRED(VERSION 2.6.4 FATAL_ERROR)

if(COMMAND cmake_policy)
    # Works around warnings libraries linked against that don't
    # have absolute paths (e.g. -lpthreads)
    cmake_policy(SET CMP0003 NEW)

    # Works around warnings about escaped quotes in ADD_DEFINITIONS
    # statements.
    cmake_policy(SET CMP0005 OLD)
    
    
    IF(COMMAND cmake_policy)
            IF(${CMAKE_MAJOR_VERSION} GREATER 2)
                # Qt5 qt5_use_modules usage was causing "Policy CMP0043 is not set: Ignore COMPILE_DEFINITIONS_<Config> properties." warnings
                cmake_policy(SET CMP0043 NEW)
            ENDIF()
    ENDIF()

    # disable autolinking to qtmain as we have our own main() functions (new in Qt 5.1)
    if(NOT "${CMAKE_VERSION}" VERSION_LESS 2.8.11)
        cmake_policy(SET CMP0020 OLD)
    endif(NOT "${CMAKE_VERSION}" VERSION_LESS 2.8.11)
endif(COMMAND cmake_policy)

#
# Set up CMake to use Solution Folders in VS.
#
SET_PROPERTY( GLOBAL PROPERTY USE_FOLDERS               ON )
SET_PROPERTY( GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "CMake Targets" )

PROJECT(OSGEARTH)

# We have some custom .cmake scripts not in the official distribution.
# Maybe this can be used override existing behavior if needed?
SET(CMAKE_MODULE_PATH "${OSGEARTH_SOURCE_DIR}/CMakeModules;${CMAKE_MODULE_PATH}")

IF(WIN32 AND NOT ANDROID)
    INCLUDE(Find3rdPartyDependencies)
    message(STATUS "Find3rdPartyDependencies设置文件目录:---${ACTUAL_3RDPARTY_DIR}" ) 
    #SET(ENV{CMAKE_PREFIX_PATH} "C:/Qt/Qt5.8.0/5.8/msvc2015_64/lib/cmake")#设置其他查找库的位置
ENDIF()


SET(OSGEARTH_MAJOR_VERSION 2)
SET(OSGEARTH_MINOR_VERSION 8)
SET(OSGEARTH_PATCH_VERSION 0)
SET(OSGEARTH_SOVERSION     0)

SET(OSGEARTH_PLUGIN_PREFIX "")

IF (CYGWIN)
    SET(OSGEARTH_PLUGIN_PREFIX "cygwin_")
ENDIF()

IF(MINGW)
    SET(OSGEARTH_PLUGIN_PREFIX "mingw_")
ENDIF()


SET(OSGEARTH_VERSION ${OSGEARTH_MAJOR_VERSION}.${OSGEARTH_MINOR_VERSION}.${OSGEARTH_PATCH_VERSION})


# We want to build SONAMES shared librariess
SET(OSGEARTH_SONAMES TRUE)
SET(OPENTHREADS_SONAMES TRUE)

SET(OpenThreads_SOURCE_DIR ${OSGEARTH_SOURCE_DIR})

# Good place to look for the OSG 3rd party libs
SET(THIRD_PARTY_DIR "" CACHE PATH "OSG 3rd-party dependency folder")

# We have some custom .cmake scripts not in the official distribution.
# Maybe this can be used override existing behavior if needed?
SET(CMAKE_MODULE_PATH "${OSGEARTH_SOURCE_DIR}/CMakeModules;${CMAKE_MODULE_PATH}")

# Attempt to detect the GIT revision number.
OPTION(OSGEARTH_EMBED_GIT_SHA "Embeds the GIT SHA in the version code" OFF)
IF (OSGEARTH_EMBED_GIT_SHA)
  include(GetGitRevisionDescription)
  get_git_head_revision(GIT_REFSPEC OSGEARTH_GIT_SHA1)
ENDIF (OSGEARTH_EMBED_GIT_SHA)

#
SET(OSGEARTH_USE_GLES FALSE)

# check for iOS build
OPTION(OSGEARTH_BUILD_PLATFORM_IPHONE "Enable IPhoneSDK Device support" OFF)
OPTION(OSGEARTH_BUILD_PLATFORM_IPHONE_SIMULATOR "Enable IPhoneSDK Simulator support" OFF)

IF(OSGEARTH_BUILD_PLATFORM_IPHONE OR OSGEARTH_BUILD_PLATFORM_IPHONE_SIMULATOR)
    SET(OSGEARTH_USE_GLES TRUE)
    #you need to manually set the default sdk version here
    SET (IPHONE_SDKVER "10.2" CACHE STRING "iOS SDK-Version")
    SET (IPHONE_VERSION_MIN "8.0" CACHE STRING "IOS minimum os version, use 7.0 or greater to get 64bit support")
    SET (IPHONE_PLATFORMSROOT "/Applications/Xcode.app/Contents/Developer/Platforms")

    IF(OSGEARTH_BUILD_PLATFORM_IPHONE)
        SET(CMAKE_OSX_SYSROOT "${IPHONE_PLATFORMSROOT}/iPhoneOS.platform/Developer/SDKs/iPhoneOS${IPHONE_SDKVER}.sdk" CACHE STRING "System root for iOS" FORCE)
        SET(CMAKE_OSX_ARCHITECTURES "armv7;armv7s;arm64" CACHE STRING "Build architectures for iOS" FORCE)
    ELSE()
        SET(CMAKE_OSX_SYSROOT "${IPHONE_PLATFORMSROOT}/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator${IPHONE_SDKVER}.sdk" CACHE STRING "System root for iOS" FORCE)
        SET(CMAKE_OSX_ARCHITECTURES "i386" CACHE STRING "Build architectures for iOS Simulator" FORCE)
    ENDIF()
ENDIF ()


# check if STLport is required
OPTION(USE_STLPORT "Set to ON to build OSGEARTH with stlport instead of the default STL library." OFF)
IF(USE_STLPORT)
	INCLUDE_DIRECTORIES(${STLPORT_DIR}/stlport)
	LINK_DIRECTORIES(${STLPORT_DIR}/lib)
ENDIF()

# Okay, here's the problem: On some platforms, linking against OpenThreads
# is not enough and explicit linking to the underlying thread library
# is also required (e.g. FreeBSD). But OpenThreads may be built with different
# backends (Pthreads, Sproc, Windows) so we don't know what the underlying
# thread library is because some platforms support multiple backends (e.g.
# IRIX supports Sproc and Pthreads). Linking all libraries won't work
# because the libraries may be incompatible.
# So the current solution is to attempt best guess linking and exempt certain
# cases. With IRIX, we're going to hope explicit linking to the underlying
# library is not necessary. We currently don't case for pthreads on Windows
# which might be an issue on things like Cygwin. This may need to be fixed.
FIND_PACKAGE(Threads)
IF(CMAKE_SYSTEM MATCHES IRIX)
    # Erase CMAKE_THREAD_LIBS_INIT and hope it works
    SET(CMAKE_THREAD_LIBS_INIT "" CACHE INTERNAL "")
ENDIF(CMAKE_SYSTEM MATCHES IRIX)

IF (OSGEARTH_USE_GLES)
    FIND_PACKAGE(OpenGLES) 
ELSE ()
    FIND_PACKAGE(OpenGL)
ENDIF (OSGEARTH_USE_GLES)

FIND_PACKAGE(CURL)
FIND_PACKAGE(GDAL)
FIND_PACKAGE(GEOS)
FIND_PACKAGE(Sqlite3)
FIND_PACKAGE(ZLIB)
FIND_PACKAGE(Poco)
FIND_PACKAGE(TIFF)
FIND_PACKAGE(LevelDB)
FIND_PACKAGE(RocksDB)

FIND_PACKAGE(SilverLining QUIET)
FIND_PACKAGE(Triton QUIET)

FIND_PACKAGE(Protobuf QUIET)

# JavaScript Engines:
SET(V8_DIR "" CACHE PATH "set to base V8 install path")
FIND_PACKAGE(V8)

SET (WITH_EXTERNAL_DUKTAPE FALSE CACHE BOOL "Use bundled or system wide version of Duktape")
IF (WITH_EXTERNAL_DUKTAPE)
    FIND_PACKAGE(Duktape)
ENDIF (WITH_EXTERNAL_DUKTAPE)

OPTION(USE_V8 "Use V8 instead of Duktape if V8 is found" OFF)

OPTION(OSGEARTH_USE_QT "Enable to use Qt (build Qt-dependent libraries, plugins and examples)" OFF)
OPTION(OSGEARTH_QT_BUILD_LEGACY_WIDGETS "Build the legacy Qt widgets" OFF)
OPTION(DESIRED_QT_VERSION "The desired Qt version to use")

IF(OSGEARTH_USE_QT)
  # To select a specific version of QT define DESIRED_QT_VERSION
  # via cmake -DDESIRED_QT_VERSION=5
  # QUIET option disables messages if the package cannot be found.
  IF  (DESIRED_QT_VERSION)
      IF  (DESIRED_QT_VERSION MATCHES 5)
          FIND_PACKAGE(Qt5Widgets REQUIRED)
      ELSEIF (DESIRED_QT_VERSION MATCHES 4)
          FIND_PACKAGE(Qt4 REQUIRED)
      ENDIF()
  ELSE()
      FIND_PACKAGE(Qt5Widgets QUIET)
      IF ( NOT Qt5Widgets_FOUND )
          FIND_PACKAGE(Qt4 REQUIRED)
      ENDIF()
  ENDIF()
ENDIF(OSGEARTH_USE_QT)


IF(Qt5Widgets_FOUND)
  message(STATUS "CmakeFile--osgQt-Qt5Widgets_FOUND" )
  FIND_PACKAGE(Qt5Core QUIET)
  FIND_PACKAGE(Qt5Gui QUIET)
  FIND_PACKAGE(Qt5OpenGL QUIET)
  IF ( Qt5Core_FOUND AND Qt5Widgets_FOUND AND Qt5Gui_FOUND AND Qt5OpenGL_FOUND )
    SET(QT_INCLUDES ${Qt5Widgets_INCLUDE_DIRS} ${Qt5OpenGL_INCLUDE_DIRS})
  ENDIF ()
ELSEIF(QT4_FOUND)
  INCLUDE(${QT_USE_FILE})
  SET(QT_INCLUDES ${QT_INCLUDES} ${QT_INCLUDE_DIR} ${QT_QTCORE_INCLUDE_DIR} ${QT_QTGUI_INCLUDE_DIR}${QT_QTOPENGL_INCLUDE_DIR} )
  SET(QT_ALL_LIBRARIES ${QT_LIBRARIES} ${QT_QTCORE_LIBRARY} ${QT_QTWEBKIT_LIBRARY} ${QT_QTNETWORK_LIBRARY} ${QT_QTXML_LIBRARY} ${QT_QTOPENGL_LIBRARY})
ENDIF ()

# option to install shaders:
OPTION(OSGEARTH_INSTALL_SHADERS "Whether to deploy GLSL shaders when doing a Make INSTALL" OFF)

SET (WITH_EXTERNAL_TINYXML FALSE CACHE BOOL "Use bundled or system wide version of TinyXML")
IF (WITH_EXTERNAL_TINYXML)
    FIND_PACKAGE(TinyXML)
ENDIF (WITH_EXTERNAL_TINYXML)

IF(UNIX)
    # Not sure what this will do on Cygwin and Msys
    # Also, remember OS X X11 is a user installed option so it may not exist.
    FIND_PACKAGE(X11)
    # Some Unicies need explicit linkage to the Math library or the build fails.
    FIND_LIBRARY(MATH_LIBRARY m)
ENDIF(UNIX)

# Common global definitions
#ADD_DEFINITIONS(-D)
# Platform specific definitions


IF(WIN32)
  FIND_PACKAGE(GLCORE)
  IF(GLCORE_FOUND)
      INCLUDE_DIRECTORIES( ${GLCORE_INCLUDE_DIR} )
  ENDIF()    

  IF(MSVC)
        # This option is to enable the /MP switch for Visual Studio 2005 and above compilers
        OPTION(WIN32_USE_MP "Set to ON to build osgEarth with the /MP option (Visual Studio 2005 and above)." OFF)
        MARK_AS_ADVANCED(WIN32_USE_MP)
        IF(WIN32_USE_MP)
            SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
        ENDIF(WIN32_USE_MP)

        # turn off various warnings
        # foreach(warning 4244 4251 4267 4275 4290 4786 4305 4996)
        #     SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd${warning}")
        # endforeach(warning)

        # More MSVC specific compilation flags
        ADD_DEFINITIONS(-D_SCL_SECURE_NO_WARNINGS)
        ADD_DEFINITIONS(-D_CRT_SECURE_NO_DEPRECATE)
    ENDIF(MSVC)
ENDIF(WIN32)

########################################################################################################
##### these were settings located in SetupCommon.cmake used in Luigi builds.... find out what are useful
########################################################################################################
#luigi#SET(CMAKE_VERBOSE_MAKEFILE TRUE)
#luigi#SET(CMAKE_SKIP_RPATH TRUE)
#luigi#SET(CMAKE_SKIP_RULE_DEPENDENCY TRUE)
#luigi#IF(UNIX)
#luigi#    LIST_CONTAINS(contains "g++" ${CMAKE_CXX_COMPILER_LIST})
#luigi#    IF (contains)
#luigi#        MESSAGE(${MY_MESSAGE_DEFAULT} "${CMAKE_CURRENT_LIST_FILE}:${CMAKE_CURRENT_LIST_LINE} setting  CMAKE_CXX_COMPILER to g++")
#luigi#        SET(CMAKE_CXX_COMPILER "g++")
#luigi#        SET(CMAKE_CXX_COMPILER_LOADED 2)
#luigi#        SET(CMAKE_CXX_COMPILER_WORKS 2)
#luigi#    ENDIF (contains)
#luigi#    SET(CMAKE_CXX_FLAGS_RELEASE "-O2")
#luigi#    SET(CMAKE_CXX_FLAGS_DEBUG "-ggdb -gstabs")
#luigi#ENDIF(UNIX)
########################################################################################################

# Common to all platforms:

SET(CMAKE_DEBUG_POSTFIX  "d" CACHE STRING "add a postfix, usually d on windows")
SET(CMAKE_RELEASE_POSTFIX "" CACHE STRING "add a postfix, usually empty on windows")
SET(CMAKE_RELWITHDEBINFO_POSTFIX "rd" CACHE STRING "add a postfix, usually empty on windows")
SET(CMAKE_MINSIZEREL_POSTFIX "s" CACHE STRING "add a postfix, usually empty on windows")

INCLUDE(OsgEarthMacroUtils)

DETECT_OSG_VERSION()

IF (NOT OPENSCENEGRAPH_VERSION)
	SET(OPENSCENEGRAPH_VERSION ${OPENSCENEGRAPH_MAJOR_VERSION}.${OPENSCENEGRAPH_MINOR_VERSION}.${OPENSCENEGRAPH_PATCH_VERSION})
ENDIF(NOT OPENSCENEGRAPH_VERSION)

FIND_PACKAGE(OSG)

# Make the headers visible to everything
INCLUDE_DIRECTORIES(
    ${OSGEARTH_SOURCE_DIR}/src
    ${OSG_INCLUDE_DIR}
)

#FIND_PACKAGE(GDAL)

################################################################################
# Create bin and lib directories if required

IF("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
   FILE(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/bin ${CMAKE_BINARY_DIR}/lib)
ENDIF("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")


################################################################################
# Installation stuff
IF (WIN32)  
    MESSAGE(STATUS "Now is windows")  
ELSEIF (APPLE)  
    MESSAGE(STATUS "Now is Apple systens.")  
ELSEIF (UNIX)  
    MESSAGE(STATUS "Now is UNIX-like OS's. Including aPPLE os x  and CygWin")  
ENDIF ()  

IF(UNIX AND NOT WIN32 AND NOT APPLE)
  IF(CMAKE_SIZEOF_VOID_P MATCHES "8")
      SET(LIB_POSTFIX "64" CACHE STRING "suffix for 32/64 dir placement")
      MARK_AS_ADVANCED(LIB_POSTFIX)
  ENDIF(CMAKE_SIZEOF_VOID_P MATCHES "8")
ENDIF(UNIX AND NOT WIN32 AND NOT APPLE)
IF(NOT DEFINED LIB_POSTFIX)
    SET(LIB_POSTFIX "")
ENDIF(NOT DEFINED LIB_POSTFIX)

#SET(OUTPUT_BINDIR ${PROJECT_BINARY_DIR}/bin/${CMAKE_SYSTEM_NAME})
SET(OUTPUT_BINDIR ${PROJECT_BINARY_DIR}/bin)
MAKE_DIRECTORY(${OUTPUT_BINDIR})
SET(EXECUTABLE_OUTPUT_PATH ${OUTPUT_BINDIR})

#SET(OUTPUT_LIBDIR ${PROJECT_BINARY_DIR}/lib/${CMAKE_SYSTEM_NAME})
SET(OUTPUT_LIBDIR ${PROJECT_BINARY_DIR}/lib)
MAKE_DIRECTORY(${OUTPUT_LIBDIR})
SET(LIBRARY_OUTPUT_PATH ${OUTPUT_LIBDIR})

LINK_DIRECTORIES( ${LINK_DIRECTORIES} ${OUTPUT_LIBDIR} )


#SET(INSTALL_BINDIR VIRTUALPLANETBUILDER/bin)
#SET(INSTALL_INCDIR VIRTUALPLANETBUILDER/include)
#SET(INSTALL_LIBDIR VIRTUALPLANETBUILDER/lib)
#SET(INSTALL_DOCDIR VIRTUALPLANETBUILDER/doc)

################################################################################
# User Options


# Expose CMAKE_INCLUDE_PATH and CMAKE_LIBARY_PATH to the GUI so users
# may set these values without needing to manipulate the environment.
SET(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH} CACHE STRING "You may add additional search paths here. Use ; to separate multiple paths.")
SET(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} CACHE STRING "You may add additional search paths here. Use ; to separate multiple paths.")
# We are proposing that a new variable called CMAKE_PREFIX_PATH be introduced
# to CMake to compliment CMAKE_INCLUDE_PATH and CMAKE_LIBRARY_PATH.
# A formal feature request has been submited to CMake, Bug #4947.
# It is intended for those users who have common prefixes for their INCLUDE
# and LIBRARY locations. So if users have headers in /usr/local/include
# and libraries in /usr/local/lib, the common prefix is /usr/local.
# It should also cover the case where headers and libraries are
# in the same directory.
# Our proposal expects that FIND_* commands will automatically search for
# CMAKE_PREFIX_PATH right after CMAKE_INCLUDE_PATH or CMAKE_LIBRARY_PATH.
# Obviously, since CMake does not currently support this, we must write
# our Find*.cmake modules to explicitly support this. Otherwise, this variable
# will have no impact.
# This is unofficial so this may be removed or changed at anytime.
#SET(${CMAKE_PREFIX_PATH} "C:/Qt/Qt5.8.0/5.8/msvc2015_64/lib/cmake")#设置其他查找库的位置
SET(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} CACHE STRING "(EXPERIMENTAL) You may add additional search paths here. Use ; to separate multiple paths.")

# Dynamic vs Static Linking
OPTION(DYNAMIC_OSGEARTH "Set to ON to build OSGEARTH for dynamic linking.  Use OFF for static." ON)
IF   (DYNAMIC_OSGEARTH)
    SET(OSGEARTH_USER_DEFINED_DYNAMIC_OR_STATIC "SHARED")
ELSE (DYNAMIC_OSGEARTH)
    SET(OSGEARTH_USER_DEFINED_DYNAMIC_OR_STATIC "STATIC")
ENDIF(DYNAMIC_OSGEARTH)

#Whether or not to copy osgEarth to the OSG_DIR during installation
# Dynamic vs Static Linking
OPTION(INSTALL_TO_OSG_DIR "Set to ON to install osgearth to the path defined in the OSG_DIR" OFF)
IF(INSTALL_TO_OSG_DIR)
    SET(OSGEARTH_INSTALL_TO_OSG_DIR TRUE)
ELSE (INSTALL_TO_OSG_DIR)
    SET(OSGEARTH_INSTALL_TO_OSG_DIR FALSE)
ENDIF(INSTALL_TO_OSG_DIR)


# osgEarth Examples
OPTION(BUILD_OSGEARTH_EXAMPLES "Enable to build osgEarth Examples" ON)

# OE Core
ADD_SUBDIRECTORY(src)

# VPB Applications
# OPTION(BUILD_APPLICATIONS "Enable to build OSGEARTH Applications" ON)
# IF   (BUILD_APPLICATIONS)
    # ADD_SUBDIRECTORY(applications)
# ENDIF(BUILD_APPLICATIONS)



# Set defaults for Universal Binaries. We want 32-bit Intel/PPC on 10.4
# and 32/64-bit Intel/PPC on >= 10.5. Anything <= 10.3 doesn't support.
IF(APPLE)

    # These are just defaults/recommendations, but how we want to build
    # out of the box. But the user needs to be able to change these options.
    # So we must only set the values the first time CMake is run, or we
    # will overwrite any changes the user sets.
    # FORCE is used because the options are not reflected in the UI otherwise.
    # Seems like a good place to add version specific compiler flags too.
    IF(NOT OSGEARTH_CONFIG_HAS_BEEN_RUN_BEFORE)
        # This is really fragile, but CMake doesn't provide the OS system
        # version information we need. (Darwin versions can be changed
        # independently of OS X versions.)
        # It does look like CMake handles the CMAKE_OSX_SYSROOT automatically.
        IF(EXISTS /Developer/SDKs/10.5.sdk)
            SET(CMAKE_OSX_ARCHITECTURES "ppc;i386;ppc64;x86_64" CACHE STRING "Build architectures for OSX" FORCE)
            SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mmacosx-version-min=10.5 -ftree-vectorize -fvisibility-inlines-hidden" CACHE STRING "Flags used by the compiler during all build types." FORCE)
        ELSE(EXISTS /Developer/SDKs/10.5.sdk)
            IF(EXISTS /Developer/SDKs/MacOSX10.4u.sdk)
                SET(CMAKE_OSX_ARCHITECTURES "ppc;i386" CACHE STRING "Build architectures for OSX" FORCE)
                SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mmacosx-version-min=10.4 -ftree-vectorize -fvisibility-inlines-hidden" CACHE STRING "Flags used by the compiler during all build types." FORCE)
            ELSE(EXISTS /Developer/SDKs/MacOSX10.4u.sdk)
                # No Universal Binary support
                # Should break down further to set the -mmacosx-version-min,
                # but the SDK detection is too unreliable here.
            ENDIF(EXISTS /Developer/SDKs/MacOSX10.4u.sdk)
        ENDIF(EXISTS /Developer/SDKs/10.5.sdk)
    ENDIF(NOT OSGEARTH_CONFIG_HAS_BEEN_RUN_BEFORE)


    OPTION(OSGEARTH_BUILD_APPLICATION_BUNDLES "Enable the building of applications and examples as OSX Bundles" OFF)
    OPTION(OSGEARTH_BUILD_FRAMEWORKS "Compile frameworks instead of dylibs" OFF)
    SET(OSGEARTH_BUILD_FRAMEWORKS_INSTALL_NAME_DIR "@executable_path/../Frameworks" CACHE STRING "Install name dir for compiled frameworks")

ENDIF(APPLE)


# This needs to be run very last so other parts of the scripts can take
# advantage of this.
IF(NOT OSGEARTH_CONFIG_HAS_BEEN_RUN_BEFORE)
    SET(OSGEARTH_CONFIG_HAS_BEEN_RUN_BEFORE 1 CACHE INTERNAL "Flag to track whether this is the first time running CMake or if CMake has been configured before")
ENDIF(NOT OSGEARTH_CONFIG_HAS_BEEN_RUN_BEFORE)


#-----------------------------------------------------------------------------
### uninstall target
#-----------------------------------------------------------------------------
CONFIGURE_FILE(
  "${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules/cmake_uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
  IMMEDIATE @ONLY)
ADD_CUSTOM_TARGET(uninstall
  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")



#------------------------------------------------------------------------------
# Docs.
add_subdirectory(docs)



#------------------------------------------------------------------------------
# Packaging setup.

SET(CPACK_PACKAGE_VERSION       ${OSGEARTH_VERSION}       )
SET(CPACK_PACKAGE_VERSION_MAJOR ${OSGEARTH_MAJOR_VERSION} )
SET(CPACK_PACKAGE_VERSION_MINOR ${OSGEARTH_MINOR_VERSION} )
SET(CPACK_PACKAGE_VERSION_PATCH ${OSGEARTH_PATCH_VERSION} )
SET(CPACK_RESOURCE_FILE_LICENSE ${PROJECT_SOURCE_DIR}/LICENSE.txt )
SET(CPACK_MONOLITHIC_INSTALL    1 )                                        # create a single, monolithic package.
    
INCLUDE(CPack)
