# FIXME: CMAKE SUPPORT IN SDL2_ttf IS VERY INCOMPLETE YET !!!
#
# FIXME: make it able build against system libs or the included ones.
# FIXME: handle library versioning.
# FIXME: test accross different target platforms.

if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "Prevented in-tree built. Please create a build directory outside of the SDL_ttf source code and call cmake from there")
endif()

##### general #####
cmake_minimum_required(VERSION 3.0)
project(SDL2_ttf C)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/")

include(DownloadProject)
if(MSVC)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
endif()
set(SDL_SHARED "ON" CACHE BOOL "SDL Shared lib" FORCE)
set(SDL_STATIC "ON" CACHE BOOL "SDL Static lib" FORCE)


download_project(
        PROJ SDL
        GIT_REPOSITORY      git@github.com:libsdl-org/SDL.git
        GIT_TAG             main
        ${UPDATE_DISCONNECTED_IF_AVAILABLE}
)

add_subdirectory(${SDL_SOURCE_DIR}  ${SDL_BINARY_DIR})

if (ANDROID)
    option(TTF_WITH_HARFBUZZ "use harfbuzz to improve text shaping" OFF)

    add_library(SDL2_ttf SHARED)
    target_sources(SDL2_ttf PRIVATE SDL_ttf.c)

    if (TTF_WITH_HARFBUZZ)
        set(HARFBUZZ_INCLUDE_DIRS ../../external/harfbuzz-2.8.0/src)
        set(HARFBUZZ_LIBRARIES harfbuzz)
        set(FREETYPE_INCLUDE_DIRS ../../external/freetype-2.11.1/include)
        set(FREETYPE_LIBRARY freetype)

        add_definitions("-DTTF_USE_HARFBUZZ=1")
        set(HB_BUILD_TESTS OFF CACHE BOOL "" FORCE)
        set(HB_HAVE_FREETYPE ON CACHE BOOL "" FORCE)
        set(FT_WITH_HARFBUZZ ON CACHE BOOL "" FORCE)

        add_subdirectory(external/harfbuzz-2.8.0)
        target_include_directories(SDL2_ttf PUBLIC external/harfbuzz-2.8.0/src)
        target_link_libraries(SDL2_ttf PRIVATE harfbuzz)

    else()
        set(FT_WITH_HARFBUZZ OFF CACHE BOOL "" FORCE)
    endif()


    target_include_directories(SDL2_ttf PUBLIC .)
    target_include_directories(SDL2_ttf external/freetype-2.11.1/include)
    target_link_libraries(SDL2_ttf PRIVATE freetype SDL2)

else()

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
message(STATUS "Add freetype...")
add_subdirectory(external/freetype-2.11.1 ${PROJECT_SOURCE_DIR}/external/freetype-2.11.1/build)

if (NOT (TARGET SDL2 OR TARGET SDL2-static))
  find_package(SDL2 REQUIRED)
endif()
find_package(PkgConfig QUIET)

set(SDL_TTF_MAJOR_VERSION 2)
set(SDL_TTF_MINOR_VERSION 0)
set(SDL_TTF_MICRO_VERSION 18)
set(SDL_TTF_INTERFACE_AGE 0)
set(SDL_TTF_BINARY_AGE 18)
set(SDL_TTF_VERSION "${SDL_TTF_MAJOR_VERSION}.${SDL_TTF_MINOR_VERSION}.${SDL_TTF_MICRO_VERSION}")

##### library generation #####
add_library(SDL2_ttf SDL_ttf.c SDL_ttf.h)
if (BUILD_SHARED_LIBS)
  target_link_libraries(SDL2_ttf SDL2::SDL2)
else()
  target_link_libraries(SDL2_ttf SDL2::SDL2-static)
  if (NOT PSP)
    set_target_properties(SDL2_ttf PROPERTIES POSITION_INDEPENDENT_CODE ON)
  endif()
endif()

target_link_libraries(SDL2_ttf freetype)
target_include_directories(SDL2_ttf PRIVATE external/freetype-2.11.1/include)
target_include_directories(SDL2_ttf PUBLIC $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/SDL2>)

include (GenerateExportHeader)
GENERATE_EXPORT_HEADER(${PROJECT_NAME}           # generates the export header shared_EXPORTS.h automatically
        BASE_NAME ${PROJECT_NAME}
        EXPORT_MACRO_NAME ${PROJECT_NAME}_EXPORTS
        EXPORT_FILE_NAME ${CMAKE_BINARY_DIR}/exports/${PROJECT_NAME}_EXPORTS.h
        STATIC_DEFINE SHARED_EXPORTS_BUILT_AS_STATIC)


file(GLOB SHOWFONT_SRC *.c *.h)
add_executable(showfont ${SHOWFONT_SRC})
target_link_libraries(showfont SDL2_ttf)
target_include_directories(showfont PRIVATE external/freetype-2.11.1/include)

install(
  TARGETS SDL2_ttf
  EXPORT SDL2_ttfTargets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
install(
  FILES SDL_ttf.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/SDL2
)

##### export files #####
if (APPLE)
  set(PKG_PREFIX "SDL2_ttf.framework/Resources")
elseif (WIN32)
  set(PKG_PREFIX "cmake")
else ()
  set(PKG_PREFIX "lib/cmake/SDL2")
endif ()

write_basic_package_version_file("${CMAKE_BINARY_DIR}/SDL2_ttfConfigVersion.cmake"
  VERSION ${SDL_TTF_VERSION}
  COMPATIBILITY AnyNewerVersion
)

install(
  EXPORT SDL2_ttfTargets
  FILE SDL2_ttfTargets.cmake
  NAMESPACE SDL2_ttf::
  DESTINATION ${PKG_PREFIX}
)
install(
  FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/SDL2_ttfConfig.cmake
    ${CMAKE_BINARY_DIR}/SDL2_ttfConfigVersion.cmake
  DESTINATION ${PKG_PREFIX}
  COMPONENT Devel
)

##### pkg-config #####
if (PKG_CONFIG_FOUND)
  set(SDL_VERSION 2.0.0)
  set(prefix ${CMAKE_INSTALL_PREFIX})
  set(exec_prefix "\${prefix}")
  set(libdir "\${exec_prefix}/lib${LIB_SUFFIX}")
  set(bindir "\${exec_prefix}/bin")
  set(includedir "\${prefix}/include")

  configure_file("${SDL_TTF_SOURCE_DIR}/SDL2_ttf.pc.in"
    "${SDL_TTF_BINARY_DIR}/SDL2_ttf.pc" @ONLY)

  if (CMAKE_SYSTEM_NAME MATCHES FreeBSD)
    # FreeBSD uses ${PREFIX}/libdata/pkgconfig
    install(FILES ${SDL_TTF_BINARY_DIR}/SDL2_ttf.pc DESTINATION "libdata/pkgconfig")
  else ()
    install(FILES ${SDL_TTF_BINARY_DIR}/SDL2_ttf.pc
      DESTINATION "lib${LIB_SUFFIX}/pkgconfig")
  endif ()
endif ()

endif()
