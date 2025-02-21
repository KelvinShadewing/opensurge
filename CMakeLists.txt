#
# Open Surge Engine
# CMakeLists.txt - CMake script
# Copyright (C) 2008-2021  Alexandre Martins <alemartf@gmail.com>
# http://opensurge2d.org
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 3 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
#

# ------------------------------------------
# Prelude
# ------------------------------------------

# Initializing...
CMAKE_MINIMUM_REQUIRED(VERSION 3.1)
PROJECT(opensurge)
SET(GAME_UNIXNAME "${CMAKE_PROJECT_NAME}")
SET(GAME_NAME "Open Surge")
FUNCTION(READ_VERSION VERSION_STRING)
  SET(VERSION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/src/core/global.h")
  SET(VERSION_MAIN_REGEX "#define[ \t]+GAME_VERSION_SUP[ \t]+(.+)")
  SET(VERSION_MAJOR_REGEX "#define[ \t]+GAME_VERSION_SUB[ \t]+(.+)")
  SET(VERSION_MINOR_REGEX "#define[ \t]+GAME_VERSION_WIP[ \t]+(.+)")
  SET(VERSION_PATCH_REGEX "#define[ \t]+GAME_VERSION_FIX[ \t]+(.+)")
  FILE(STRINGS "${VERSION_FILE}" GAME_VERSION_SUP REGEX "${VERSION_MAIN_REGEX}")
  FILE(STRINGS "${VERSION_FILE}" GAME_VERSION_SUB REGEX "${VERSION_MAJOR_REGEX}")
  FILE(STRINGS "${VERSION_FILE}" GAME_VERSION_WIP REGEX "${VERSION_MINOR_REGEX}")
  FILE(STRINGS "${VERSION_FILE}" GAME_VERSION_FIX REGEX "${VERSION_PATCH_REGEX}")
  STRING(REGEX REPLACE ${VERSION_MAIN_REGEX} "\\1" GAME_VERSION_SUP "${GAME_VERSION_SUP}")
  STRING(REGEX REPLACE ${VERSION_MAJOR_REGEX} "\\1" GAME_VERSION_SUB "${GAME_VERSION_SUB}")
  STRING(REGEX REPLACE ${VERSION_MINOR_REGEX} "\\1" GAME_VERSION_WIP "${GAME_VERSION_WIP}")
  STRING(REGEX REPLACE ${VERSION_PATCH_REGEX} "\\1" GAME_VERSION_FIX "${GAME_VERSION_FIX}")
  SET(${VERSION_STRING} "${GAME_VERSION_SUP}.${GAME_VERSION_SUB}.${GAME_VERSION_WIP}.${GAME_VERSION_FIX}" PARENT_SCOPE)
ENDFUNCTION()
READ_VERSION(GAME_VERSION)
MESSAGE("${GAME_NAME} version ${GAME_VERSION}")
SET(HELP "You might get some luck by tweaking CMake options with ccmake | cmake-gui, or you can get help at opensurge2d.org")

# Default config
SET(CMAKE_C_STANDARD 99)
IF(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  SET(CMAKE_BUILD_TYPE "Release" CACHE STRING "Debug | Release | MinSizeRel | RelWithDebInfo" FORCE)
  SET_PROPERTY(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
ENDIF()
SET(DEFS "") # #define'd stuff

# User options
OPTION(USE_A5 "Use the Allegro 5 backend" ON)
OPTION(LEGACY_USE_STATIC "Use static libs whenever possible (legacy; Allegro 4 only)" ON)
OPTION(LEGACY_USE_OPENAL "Use OpenAL for audio playback (deprecated; Allegro 4 only)" OFF)
OPTION(SURGESCRIPT_STATIC "Link SurgeScript statically" OFF)
OPTION(ALLEGRO_STATIC "Use the static version of Allegro 5 (Windows only)" OFF)
OPTION(ALLEGRO_MONOLITH "Use the monolith version of Allegro 5" OFF)
IF(UNIX)
  OPTION(DESKTOP_INSTALL "Add Open Surge to the Desktop menu when installing" ON)
ENDIF()

# User-specified paths
SET(ALLEGRO_LIBRARY_PATH "${CMAKE_LIBRARY_PATH}" CACHE PATH "Where to look for Allegro & its dependencies")
SET(ALLEGRO_INCLUDE_PATH "${CMAKE_INCLUDE_PATH}" CACHE PATH "Custom include directory for Allegro (where to look for the header files)")
SET(SURGESCRIPT_LIBRARY_PATH "${CMAKE_LIBRARY_PATH}" CACHE PATH "Where to look for SurgeScript")
SET(SURGESCRIPT_INCLUDE_PATH "${CMAKE_INCLUDE_PATH}" CACHE PATH "Custom include directory for SurgeScript (where to look for the header files)")
IF(UNIX AND CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  SET(CMAKE_INSTALL_PREFIX "/usr" CACHE PATH "Install path prefix (prepended onto install directories)" FORCE)
ENDIF()
IF(UNIX)
  SET(GAME_BINDIR "games" CACHE PATH "Directory in which the game executable will be stored when installing")
  SET(GAME_DATADIR "share/games/${GAME_UNIXNAME}" CACHE PATH "Directory in which the game data will be stored when installing")
  SET(DESKTOP_ICON_PATH "/usr/share/pixmaps" CACHE PATH "Where to store the icon file when installing")
  SET(DESKTOP_ENTRY_PATH "/usr/share/applications" CACHE PATH "Where to store the .desktop file when installing")
  SET(DESKTOP_METAINFO_PATH "/usr/share/metainfo" CACHE PATH "Where to store the metainfo file. You shouldn't change this (check the AppStream spec).")
ENDIF()

# Compiling the development version?
OPTION(BLEEDING_EDGE "Enable if using the development version" OFF)
SET(BLEEDING_EDGE_VERSION "custom" CACHE STRING "Build version (bleeding edge)")
IF(BLEEDING_EDGE)
  SET(DEFS "${DEFS}" "GAME_BUILD_VERSION=\"${BLEEDING_EDGE_VERSION}-dev\"")
ENDIF()

# data folder (game assets)
IF(UNIX)
  IF(IS_ABSOLUTE "${GAME_DATADIR}")
    SET(DEFS "${DEFS}" "GAME_DATADIR=\"${GAME_DATADIR}\"")
  ELSEIF(IS_ABSOLUTE "${CMAKE_INSTALL_PREFIX}/${GAME_DATADIR}")
    SET(DEFS "${DEFS}" "GAME_DATADIR=\"${CMAKE_INSTALL_PREFIX}/${GAME_DATADIR}\"")
  ELSE()
    MESSAGE(FATAL_ERROR "GAME_DATADIR is not an absolute path: ${CMAKE_INSTALL_PREFIX}/${GAME_DATADIR}")
  ENDIF()
ENDIF()

# Use the Allegro 5 backend
IF(USE_A5)
  SET(DEFS ${DEFS} "A5BUILD")
  MESSAGE(STATUS "Building using the Allegro 5 backend...")
ELSE()
  MESSAGE(STATUS "Building using the legacy Allegro 4 backend...")
ENDIF()

# legacy option: prefer static libs
IF(NOT USE_A5 AND LEGACY_USE_STATIC)
  IF(WIN32)
    SET(CMAKE_FIND_LIBRARY_SUFFIXES .lib .a ${CMAKE_FIND_LIBRARY_SUFFIXES})
  ELSE()
    SET(CMAKE_FIND_LIBRARY_SUFFIXES .a ${CMAKE_FIND_LIBRARY_SUFFIXES})
  ENDIF()
ENDIF()

# Generate files from templates
FUNCTION(generate_file TEMPLATE)
  CONFIGURE_FILE("src/misc/${TEMPLATE}.in" "src/misc/${TEMPLATE}" @ONLY)
ENDFUNCTION()

# Generate the content of the credits screen
SET(COPYRIGHT_DATA_CSV "")
FILE(STRINGS "src/misc/copyright_data.csv" COPYRIGHT_DATA_CSV_LIST ENCODING UTF-8)
FOREACH(line IN LISTS COPYRIGHT_DATA_CSV_LIST)
  SET(COPYRIGHT_DATA_CSV "${COPYRIGHT_DATA_CSV}\n\"${line}\\n\"")
ENDFOREACH()
generate_file("credits.c")

# Generate desktop & metadata files
# They require the absolute path of the icon
IF(DEFINED DESKTOP_ICON_PATH)
  SET(DESKTOP_ICON_FULLPATH "${DESKTOP_ICON_PATH}/opensurge.png")
  generate_file("opensurge.desktop")
  generate_file("opensurge.appdata.xml")
ENDIF()

# ------------------------------------------
# Finding the required libraries
# ------------------------------------------
IF(USE_A5)

  # Find Allegro 5 & addons
  SET(REQUIRED_A5_LIBS
    allegro
    allegro_image
    allegro_primitives
    allegro_font
    allegro_ttf
    allegro_acodec
    allegro_audio
    allegro_dialog
    allegro_memfile
  )

  # OSX requires allegro_main
  IF(APPLE)
    LIST(APPEND REQUIRED_A5_LIBS allegro_main)
  ENDIF()

  # Use the monolith version
  IF(ALLEGRO_MONOLITH)
    SET(REQUIRED_A5_LIBS allegro_monolith)
  ENDIF()

  # Use static libs
  IF(ALLEGRO_STATIC)
    # add -static suffix
    SET(A5_LIBS "")
    FOREACH(A5_LIB ${REQUIRED_A5_LIBS})
      LIST(APPEND A5_LIBS "${A5_LIB}-static")
    ENDFOREACH()
    SET(REQUIRED_A5_LIBS ${A5_LIBS})

    # define ALLEGRO_STATICLINK
    LIST(APPEND DEFS "ALLEGRO_STATICLINK")

    # require Win32 libs
    IF(WIN32)
      LIST(APPEND REQUIRED_A5_LIBS
        mingw32
        #dumb
        #FLAC
        vorbisfile
        vorbis
        freetype
        ogg
        #physfs
        png16
        z
        #zlibstatic
        gdiplus
        uuid
        kernel32
        winmm
        psapi
        opengl32
        glu32
        user32
        comdlg32
        gdi32
        shell32
        ole32
        advapi32
        ws2_32
        shlwapi
        dsound
        #d3dx9 # needed?
        jpeg
        #opusfile
        #opus
        #webp
      )
    ELSE()
      MESSAGE(FATAL_ERROR "Static linking is currently unavailable for this platform.")
    ENDIF()
  ENDIF()

  # Write to LALLEGRO5
  SET(LALLEGRO5 "")
  FOREACH(A5_LIB ${REQUIRED_A5_LIBS})
    STRING(TOUPPER "L${A5_LIB}" _A5_LIB)
    FIND_LIBRARY(${_A5_LIB} NAMES "${A5_LIB}" PATHS ${ALLEGRO_LIBRARY_PATH})
    IF(NOT ${_A5_LIB})
      MESSAGE(FATAL_ERROR "Can't find Allegro 5 lib (lib${A5_LIB}). ${HELP}")
    ELSE()
      MESSAGE(STATUS "Found lib${A5_LIB} at ${${_A5_LIB}}...")
      SET(LALLEGRO5 "${LALLEGRO5}" "${${_A5_LIB}}")
    ENDIF()
  ENDFOREACH()

ELSE()
  
  # Allegro 4: liballeg
  FIND_LIBRARY(LALLEG NAMES alleg PATHS ${ALLEGRO_LIBRARY_PATH})
  IF(NOT LALLEG)
    MESSAGE(FATAL_ERROR "Fatal error: couldn't find the Allegro game programming library (liballeg)! ${HELP}")
  ELSE(NOT LALLEG)
    IF(UNIX)
      SET(ALLEGRO_RECOMMENDED_VERSION "4.4.3")
      EXECUTE_PROCESS(COMMAND allegro-config --version OUTPUT_VARIABLE ALLEGRO_UNIX_VERSION)
      IF(ALLEGRO_UNIX_VERSION)
        STRING(REGEX REPLACE "[ \r\n\t]+$" "" ALLEGRO_UNIX_VERSION ${ALLEGRO_UNIX_VERSION})
      ENDIF(ALLEGRO_UNIX_VERSION)
      IF(NOT ALLEGRO_UNIX_VERSION STREQUAL ALLEGRO_RECOMMENDED_VERSION)
        MESSAGE("\nWARNING: your Allegro version is ${ALLEGRO_UNIX_VERSION}. The recommended version is ${ALLEGRO_RECOMMENDED_VERSION}.\n         ${GAME_NAME} may not compile properly!\n")
      ENDIF(NOT ALLEGRO_UNIX_VERSION STREQUAL ALLEGRO_RECOMMENDED_VERSION)
    ENDIF(UNIX)
  ENDIF(NOT LALLEG)

  # loadpng: libloadpng, libpng and zlib
  FIND_LIBRARY(LLOADPNG NAMES loadpng PATHS ${ALLEGRO_LIBRARY_PATH})
  IF(NOT LLOADPNG)
    MESSAGE(FATAL_ERROR "Fatal error: libloadpng not found! ${HELP}")
  ENDIF(NOT LLOADPNG)

  FIND_LIBRARY(LPNG NAMES png PATHS ${ALLEGRO_LIBRARY_PATH})
  IF(NOT LPNG)
    MESSAGE(FATAL_ERROR "Fatal error: libpng not found! ${HELP}")
  ENDIF(NOT LPNG)

  FIND_LIBRARY(LZ NAMES z PATHS ${ALLEGRO_LIBRARY_PATH})
  IF(NOT LZ)
    MESSAGE(FATAL_ERROR "Fatal error: zlib not found! ${HELP}")
  ENDIF(NOT LZ)

  # JPGalleg: libjpgalleg
  FIND_LIBRARY(LJPGALLEG NAMES jpgalleg PATHS ${ALLEGRO_LIBRARY_PATH})
  IF(NOT LJPGALLEG)
    MESSAGE(FATAL_ERROR "Fatal error: libjpgalleg not found! ${HELP}")
  ENDIF(NOT LJPGALLEG)

  # Ogg/Vorbis: libogg, libvorbis, libvorbisfile
  FIND_LIBRARY(LOGG NAMES ogg PATHS ${ALLEGRO_LIBRARY_PATH})
  IF(NOT LOGG)
    MESSAGE(FATAL_ERROR "Fatal error: libogg not found! ${HELP}")
  ENDIF(NOT LOGG)

  FIND_LIBRARY(LVORBIS NAMES vorbis PATHS ${ALLEGRO_LIBRARY_PATH})
  IF(NOT LVORBIS)
    MESSAGE(FATAL_ERROR "Fatal error: libvorbis not found! ${HELP}")
  ENDIF(NOT LVORBIS)

  FIND_LIBRARY(LVORBISFILE NAMES vorbisfile PATHS ${ALLEGRO_LIBRARY_PATH})
  IF(NOT LVORBISFILE)
    MESSAGE(FATAL_ERROR "Fatal error: libvorbisfile not found! ${HELP}")
  ENDIF(NOT LVORBISFILE)

  # LOGG: liblogg
  IF(NOT LEGACY_USE_OPENAL)
    MESSAGE(STATUS "Building using Allegro for audio playback...")
    FIND_LIBRARY(LLOGG NAMES logg PATHS ${ALLEGRO_LIBRARY_PATH})
    IF(NOT LLOGG)
      MESSAGE(FATAL_ERROR "Fatal error: liblogg not found! ${HELP}")
    ENDIF(NOT LLOGG)
    SET(AUDIO_LIBS logg vorbisfile vorbis ogg)
  ENDIF(NOT LEGACY_USE_OPENAL)

  # OpenAL & ALURE
  IF(LEGACY_USE_OPENAL)
    MESSAGE(STATUS "Building using OpenAL for audio playback...")
    FIND_LIBRARY(LOPENAL NAMES openal OpenAL OpenAL32 al PATHS ${ALLEGRO_LIBRARY_PATH})
    IF(NOT LOPENAL)
      MESSAGE(FATAL_ERROR "Fatal error: can't find OpenAL! ${HELP}")
    ENDIF(NOT LOPENAL)
    FIND_LIBRARY(LALURE NAMES alure-static ALURE32-static PATHS ${ALLEGRO_LIBRARY_PATH}) # try the static version first
    IF(NOT LALURE)
      FIND_LIBRARY(LALURE NAMES alure PATHS ${ALLEGRO_LIBRARY_PATH})
      IF(NOT LALURE)
        MESSAGE(FATAL_ERROR "Fatal error: can't find ALURE! ${HELP}")
      ENDIF(NOT LALURE)
    ENDIF(NOT LALURE)

    IF(UNIX)
      FIND_LIBRARY(LPTHREAD NAMES pthread PATHS ${ALLEGRO_LIBRARY_PATH})
      IF(NOT LPTHREAD)
        MESSAGE(FATAL_ERROR "Fatal error: can't find libpthread! ${HELP}")
      ENDIF(NOT LPTHREAD)
      IF(NOT ${CMAKE_SYSTEM_NAME} MATCHES ".*BSD|DragonFly.*")
        FIND_LIBRARY(LDL NAMES dl PATHS ${ALLEGRO_LIBRARY_PATH})
        IF(NOT LDL)
          MESSAGE(FATAL_ERROR "Fatal error: can't find libdl! ${HELP}")
        ENDIF(NOT LDL)
      ENDIF(NOT ${CMAKE_SYSTEM_NAME} MATCHES ".*BSD|DragonFly.*")
    ELSE(UNIX)
      SET(LPTHREAD "") # TODO?
      SET(LDL "")
    ENDIF(UNIX)

    SET(DEFS ${DEFS} __USE_OPENAL__ ALURE_STATIC_LIBRARY)
    SET(AUDIO_LIBS ${LALURE} ${LOPENAL} vorbisfile vorbis ogg stdc++ ${LPTHREAD} ${LDL})
  ENDIF(LEGACY_USE_OPENAL)

  # Alfont: libalfont
  FIND_LIBRARY(LALFONT NAMES alfont PATHS ${ALLEGRO_LIBRARY_PATH})
  IF(NOT LALFONT)
    MESSAGE(FATAL_ERROR "Fatal error: modified alfont not found! ${HELP}")
  ENDIF(NOT LALFONT)

ENDIF()

# SurgeScript: libsurgescript
SET(LSURGESCRIPT_NAME surgescript)
IF(SURGESCRIPT_STATIC)
  SET(LSURGESCRIPT_NAME "${LSURGESCRIPT_NAME}-static")
ENDIF()

FIND_LIBRARY(LSURGESCRIPT NAMES "${LSURGESCRIPT_NAME}" PATHS ${SURGESCRIPT_LIBRARY_PATH})
IF(NOT LSURGESCRIPT)
  MESSAGE(FATAL_ERROR "Fatal error: can't find lib${LSURGESCRIPT_NAME}! ${HELP}")
ELSE()
  MESSAGE(STATUS "Found lib${LSURGESCRIPT_NAME} at ${LSURGESCRIPT}...")
ENDIF()

# setup preprocessor #define's
IF(MSVC)
  FOREACH(d ${DEFS})
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /D${d}")
  ENDFOREACH()
ELSE()
  FOREACH(d ${DEFS})
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D'${d}'")
  ENDFOREACH()
ENDIF()

# ------------------------------------------
# Listing the source files
# ------------------------------------------

# adding sources
SET(GAME_SRCS
  src/core/nanoparser/nanoparser.c
  src/core/whereami/whereami.c
  src/core/utf8/utf8.c
  src/core/zip/zip.c
  src/core/assetfs.c
  src/core/audio.c
  src/core/color.c
  src/core/commandline.c
  src/core/csv.c
  src/core/engine.c
  src/core/font.c
  src/core/image.c
  src/core/input.c
  src/core/inputmap.c
  src/core/install.c
  src/core/lang.c
  src/core/logfile.c
  src/core/modmanager.c
  src/core/web.c
  src/core/prefs.c
  src/core/quest.c
  src/core/resourcemanager.c
  src/core/scene.c
  src/core/screenshot.c
  src/core/fadefx.c
  src/core/sprite.c
  src/core/storyboard.c
  src/core/stringutil.c
  src/core/timer.c
  src/core/util.c
  src/core/v2d.c
  src/core/video.c

  src/scenes/util/editorcmd.c
  src/scenes/util/editorgrp.c
  src/scenes/util/grouptree.c
  src/scenes/confirmbox.c
  src/scenes/credits.c
  src/scenes/editorhelp.c
  src/scenes/editorpal.c
  src/scenes/gameover.c
  src/scenes/intro.c
  src/scenes/langselect.c
  src/scenes/level.c
  src/scenes/options.c
  src/scenes/pause.c
  src/scenes/quest.c
  src/scenes/questselect.c
  src/scenes/stageselect.c

  src/scripting/scripting.c
  src/scripting/application.c
  src/scripting/surgeengine.c
  src/scripting/actor.c
  src/scripting/animation.c
  src/scripting/brick.c
  src/scripting/camera.c
  src/scripting/collisions.c
  src/scripting/console.c
  src/scripting/events.c
  src/scripting/input.c
  src/scripting/lang.c
  src/scripting/level.c
  src/scripting/levelmanager.c
  src/scripting/mouse.c
  src/scripting/music.c
  src/scripting/obstaclemap.c
  src/scripting/player.c
  src/scripting/prefs.c
  src/scripting/screen.c
  src/scripting/sensor.c
  src/scripting/sound.c
  src/scripting/text.c
  src/scripting/time.c
  src/scripting/transform.c
  src/scripting/vector2.c
  src/scripting/web.c
  src/physics/obstacle.c
  src/physics/obstaclemap.c
  src/physics/physicsactor.c
  src/physics/sensor.c
  src/physics/sensorstate.c
  src/physics/collisionmask.c

  src/entities/legacy/item.c
  src/entities/legacy/enemy.c
  src/entities/legacy/object_compiler.c
  src/entities/legacy/object_decorators.c
  src/entities/legacy/object_machine.c
  src/entities/legacy/object_vm.c
  src/entities/legacy/nanocalc/nanocalc.c
  src/entities/legacy/nanocalc/nanocalc_addons.c
  src/entities/legacy/nanocalc/nanocalcext.c

  src/entities/actor.c
  src/entities/background.c
  src/entities/brick.c
  src/entities/camera.c
  src/entities/character.c
  src/entities/entitymanager.c
  src/entities/player.c
  src/entities/particle.c
  src/entities/renderqueue.c

  src/misc/icon.c
  ${CMAKE_CURRENT_BINARY_DIR}/src/misc/credits.c

  src/main.c
)

# adding headers
SET(GAME_HEADERS
  src/core/nanoparser/nanoparser.h
  src/core/whereami/whereami.h
  src/core/utf8/utf8.h
  src/core/zip/zip.c
  src/core/assetfs.h
  src/core/audio.h
  src/core/color.h
  src/core/commandline.h
  src/core/csv.h
  src/core/engine.h
  src/core/fasthash.h
  src/core/font.h
  src/core/global.h
  src/core/hashtable.h
  src/core/image.h
  src/core/input.h
  src/core/inputmap.h
  src/core/install.h
  src/core/lang.h
  src/core/logfile.h
  src/core/modmanager.h
  src/core/web.h
  src/core/prefs.h
  src/core/quest.h
  src/core/resourcemanager.h
  src/core/scene.h
  src/core/screenshot.h
  src/core/fadefx.h
  src/core/spatialhash.h
  src/core/sprite.h
  src/core/storyboard.h
  src/core/stringutil.h
  src/core/timer.h
  src/core/util.h
  src/core/video.h
  src/core/v2d.h

  src/scenes/util/editorcmd.h
  src/scenes/util/editorgrp.h
  src/scenes/util/grouptree.h
  src/scenes/confirmbox.h
  src/scenes/editorhelp.h
  src/scenes/editorpal.h
  src/scenes/credits.h
  src/scenes/gameover.h
  src/scenes/intro.h
  src/scenes/langselect.h
  src/scenes/level.h
  src/scenes/options.h
  src/scenes/pause.h
  src/scenes/quest.h
  src/scenes/questselect.h
  src/scenes/stageselect.h

  src/scripting/scripting.h
  src/physics/obstacle.h
  src/physics/obstaclemap.h
  src/physics/physicsactor.h
  src/physics/sensor.h
  src/physics/sensorstate.h
  src/physics/collisionmask.h

  src/entities/legacy/item.h
  src/entities/legacy/enemy.h
  src/entities/legacy/object_compiler.h
  src/entities/legacy/object_decorators.h
  src/entities/legacy/object_machine.h
  src/entities/legacy/object_vm.h
  src/entities/legacy/nanocalc/nanocalc.h
  src/entities/legacy/nanocalc/nanocalc_addons.h
  src/entities/legacy/nanocalc/nanocalcext.h

  src/entities/actor.h
  src/entities/background.h
  src/entities/brick.h
  src/entities/camera.h
  src/entities/character.h
  src/entities/entitymanager.h
  src/entities/object_compiler.h
  src/entities/object_vm.h
  src/entities/player.h
  src/entities/particle.h
  src/entities/renderqueue.h
  src/entities/sfx.h

  src/misc/iconwin.rc
)

# Allegro 4 specifics
IF(NOT USE_A5)
  IF(UNIX)
    SET(GAME_SRCS ${GAME_SRCS} src/misc/icon_a4.c)
  ENDIF()
  SET(GAME_SRCS ${GAME_SRCS} src/core/hqx/hqx.c)
  SET(GAME_HEADERS ${GAME_HEADERS} src/core/hqx/hqx.h)
ENDIF()

# ------------------------------------------
# Creating the executable
# ------------------------------------------

# A5 build
IF(USE_A5)

  IF(WIN32)

    # Windows executable
    IF(MINGW)
      IF(NOT CMAKE_RC_COMPILER)
        SET(CMAKE_RC_COMPILER windres)
      ENDIF(NOT CMAKE_RC_COMPILER)
      ADD_EXECUTABLE(${GAME_UNIXNAME} WIN32 ${GAME_SRCS})
      TARGET_LINK_LIBRARIES(${GAME_UNIXNAME} m ${LSURGESCRIPT} ${LALLEGRO5})
      TARGET_INCLUDE_DIRECTORIES(${GAME_UNIXNAME} PUBLIC ${SURGESCRIPT_INCLUDE_PATH} ${ALLEGRO_INCLUDE_PATH})
      SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES COMPILE_FLAGS "-Wall")
      EXECUTE_PROCESS(COMMAND ${CMAKE_RC_COMPILER} -O coff -o "${CMAKE_CURRENT_BINARY_DIR}/iconwin.res" -i "${CMAKE_SOURCE_DIR}/src/misc/iconwin.rc" -I "${CMAKE_SOURCE_DIR}")
      SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES LINK_FLAGS "-static-libgcc -static-libstdc++ \"${CMAKE_CURRENT_BINARY_DIR}/iconwin.res\"")
    ELSEIF(MSVC)
      ADD_EXECUTABLE(${GAME_UNIXNAME} WIN32 ${GAME_SRCS} ${GAME_HEADERS})
      TARGET_LINK_LIBRARIES(${GAME_UNIXNAME} ${LSURGESCRIPT} ${LALLEGRO5})
      TARGET_INCLUDE_DIRECTORIES(${GAME_UNIXNAME} PUBLIC ${SURGESCRIPT_INCLUDE_PATH} ${ALLEGRO_INCLUDE_PATH})
      SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES COMPILE_FLAGS "/D_CRT_SECURE_NO_DEPRECATE /D_CRT_SECURE_NO_WARNINGS ${CMAKE_C_FLAGS}")
    ELSE()
      MESSAGE("*** Unrecognized compiler ***") # e.g., clang?
      ADD_EXECUTABLE(${GAME_UNIXNAME} WIN32 ${GAME_SRCS})
      TARGET_LINK_LIBRARIES(${GAME_UNIXNAME} m ${LSURGESCRIPT} ${LALLEGRO5})
      TARGET_INCLUDE_DIRECTORIES(${GAME_UNIXNAME} PUBLIC ${SURGESCRIPT_INCLUDE_PATH} ${ALLEGRO_INCLUDE_PATH})
      SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES COMPILE_FLAGS "-Wall")     
    ENDIF()

    # Bugfix
    IF(ALLEGRO_STATIC)
      SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES LINKER_LANGUAGE CXX)
    ENDIF()

  ELSEIF(APPLE)

    # OSX executable
    ADD_EXECUTABLE(${GAME_UNIXNAME} MACOSX_BUNDLE ${GAME_SRCS})
    TARGET_LINK_LIBRARIES(${GAME_UNIXNAME} m ${LSURGESCRIPT} ${LALLEGRO5})
    TARGET_INCLUDE_DIRECTORIES(${GAME_UNIXNAME} PUBLIC ${SURGESCRIPT_INCLUDE_PATH} ${ALLEGRO_INCLUDE_PATH})
    SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES COMPILE_FLAGS "-Wall")

  ELSEIF(UNIX)

    # *nix executable
    ADD_EXECUTABLE(${GAME_UNIXNAME} ${GAME_SRCS})
    TARGET_LINK_LIBRARIES(${GAME_UNIXNAME} m ${LSURGESCRIPT} ${LALLEGRO5})
    TARGET_INCLUDE_DIRECTORIES(${GAME_UNIXNAME} PUBLIC ${SURGESCRIPT_INCLUDE_PATH} ${ALLEGRO_INCLUDE_PATH})
    SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES COMPILE_FLAGS "-Wall")

  ENDIF()

ELSE()

  IF(UNIX)

    # *nix executable (A4)
    ADD_EXECUTABLE(${GAME_UNIXNAME} ${GAME_SRCS})
    EXECUTE_PROCESS(COMMAND allegro-config --libs OUTPUT_VARIABLE ALLEGRO_UNIX_LIBS)
    IF(ALLEGRO_UNIX_LIBS)
      STRING(REGEX REPLACE "[ \r\n\t]+$" "" ALLEGRO_UNIX_LIBS ${ALLEGRO_UNIX_LIBS})
    ELSE()
      MESSAGE(FATAL_ERROR "Couldn't run \"allegro-config --libs\"")
    ENDIF()
    SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES LINK_FLAGS ${ALLEGRO_UNIX_LIBS})
    TARGET_LINK_LIBRARIES(${GAME_UNIXNAME} m ${AUDIO_LIBS} ${LSURGESCRIPT} ${LJPGALLEG} ${LLOADPNG} ${LPNG} ${LZ} ${LALFONT} ${LALLEG})
    TARGET_INCLUDE_DIRECTORIES(${GAME_UNIXNAME} PUBLIC ${SURGESCRIPT_INCLUDE_PATH} ${ALLEGRO_INCLUDE_PATH})
    SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES COMPILE_FLAGS "-Wall -O2 ${CMAKE_C_FLAGS}")

  ELSEIF(WIN32)

    # Windows executable (A4)
    IF(MSVC)
      ADD_EXECUTABLE(${GAME_UNIXNAME} WIN32 ${GAME_SRCS} ${GAME_HEADERS})
      SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES COMPILE_FLAGS "/D_CRT_SECURE_NO_DEPRECATE /D__WIN32__ /D__MSVC__ ${CMAKE_C_FLAGS}")
      TARGET_LINK_LIBRARIES(${GAME_UNIXNAME} ${AUDIO_LIBS} ${LSURGESCRIPT} ${LJPGALLEG} ${LLOADPNG} ${LALFONT} ${LALLEG} ${LPNG} ${LZ})
      TARGET_INCLUDE_DIRECTORIES(${GAME_UNIXNAME} PUBLIC ${SURGESCRIPT_INCLUDE_PATH} ${ALLEGRO_INCLUDE_PATH})
    ELSEIF(MINGW)
      ADD_EXECUTABLE(${GAME_UNIXNAME} WIN32 ${GAME_SRCS})
      SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES COMPILE_FLAGS "-Wall -O2 -D__WIN32__ ${CMAKE_C_FLAGS}")
      TARGET_LINK_LIBRARIES(${GAME_UNIXNAME} ${AUDIO_LIBS} ${LSURGESCRIPT} ${LJPGALLEG} ${LLOADPNG} ${LALFONT} ${LALLEG} ${LPNG} ${LZ} m)
      TARGET_INCLUDE_DIRECTORIES(${GAME_UNIXNAME} PUBLIC ${SURGESCRIPT_INCLUDE_PATH} ${ALLEGRO_INCLUDE_PATH})
      IF(NOT CMAKE_RC_COMPILER)
        SET(CMAKE_RC_COMPILER windres)
      ENDIF()
      EXECUTE_PROCESS(COMMAND ${CMAKE_RC_COMPILER} -O coff -o "${CMAKE_CURRENT_BINARY_DIR}/iconwin.res" -i "${CMAKE_SOURCE_DIR}/src/misc/iconwin.rc" -I "${CMAKE_SOURCE_DIR}")
      SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES LINK_FLAGS "-static-libgcc \"${CMAKE_CURRENT_BINARY_DIR}/iconwin.res\"")
    ELSE()
      ADD_EXECUTABLE(${GAME_UNIXNAME} WIN32 ${GAME_SRCS})
      SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES COMPILE_FLAGS "-D__WIN32__ ${CMAKE_C_FLAGS}")
      TARGET_LINK_LIBRARIES(${GAME_UNIXNAME} ${AUDIO_LIBS} ${LSURGESCRIPT} ${LJPGALLEG} ${LLOADPNG} ${LALFONT} ${LALLEG} ${LPNG} ${LZ} m)
      TARGET_INCLUDE_DIRECTORIES(${GAME_UNIXNAME} PUBLIC ${SURGESCRIPT_INCLUDE_PATH} ${ALLEGRO_INCLUDE_PATH})
    ENDIF()

  ENDIF()

ENDIF()

# Target properties
SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES PROJECT_NAME "${GAME_NAME}")
SET_TARGET_PROPERTIES(${GAME_UNIXNAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}")

# Installing on *nix
IF(UNIX)
  INSTALL(CODE "MESSAGE(\"Installing ${GAME_NAME} ${GAME_VERSION}... Make sure that you have the appropriate privileges.\")")
  INSTALL(FILES LICENSE README.md CHANGES.md CONTRIBUTING.md logo.png surge.png surge.rocks DESTINATION "${GAME_DATADIR}")
  INSTALL(DIRECTORY characters scripts sprites images levels musics quests samples scripts themes languages fonts inputs licenses DESTINATION "${GAME_DATADIR}" PATTERN ".git" EXCLUDE)
  INSTALL(FILES src/misc/opensurge.png DESTINATION "${DESKTOP_ICON_PATH}")
  INSTALL(FILES "${CMAKE_CURRENT_BINARY_DIR}/src/misc/opensurge.appdata.xml" DESTINATION "${DESKTOP_METAINFO_PATH}")
  IF(DESKTOP_INSTALL)
    INSTALL(FILES "${CMAKE_CURRENT_BINARY_DIR}/src/misc/opensurge.desktop" DESTINATION "${DESKTOP_ENTRY_PATH}")
  ENDIF()
  INSTALL(TARGETS "${GAME_UNIXNAME}" RUNTIME DESTINATION "${GAME_BINDIR}")

  # Success!
  IF(IS_ABSOLUTE "${GAME_BINDIR}")
    INSTALL(CODE "MESSAGE(\"Installed ${GAME_UNIXNAME} to ${GAME_BINDIR}.\")")
  ELSE()
    INSTALL(CODE "MESSAGE(\"Installed ${GAME_UNIXNAME} to ${CMAKE_INSTALL_PREFIX}/${GAME_BINDIR}.\")")
  ENDIF()
ENDIF()