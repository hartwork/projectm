if(NOT ENABLE_PLAYLIST)
    return()
endif()

add_library(projectM_playlist_main OBJECT
        Filter.cpp
        Filter.hpp
        Item.cpp
        Item.hpp
        Playlist.cpp
        Playlist.hpp
        PlaylistCWrapper.cpp
        PlaylistCWrapper.hpp
        api/libprojectM/playlist.h
        api/libprojectM/playlist_callbacks.h
        api/libprojectM/playlist_core.h
        api/libprojectM/playlist_filter.h
        api/libprojectM/playlist_items.h
        api/libprojectM/playlist_memory.h
        api/libprojectM/playlist_playback.h
        api/libprojectM/playlist_types.h
        )

target_include_directories(projectM_playlist_main
        PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/api>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
        )

target_link_libraries(projectM_playlist_main
        PRIVATE
        libprojectM::API
        )

add_library(projectM_playlist
        ${PROJECTM_DUMMY_SOURCE_FILE} # CMake needs at least one "real" source file.
        $<TARGET_OBJECTS:projectM_playlist_main>
        )

set_target_properties(projectM_playlist PROPERTIES
        VERSION "${PROJECTM_LIB_VERSION}"
        SOVERSION "${PROJECTM_SO_VERSION}"
        EXPORT_NAME playlist
        FOLDER libprojectM
        )

target_include_directories(projectM_playlist
        PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/api>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
        "$<INSTALL_INTERFACE:${PROJECTM_INCLUDE_DIR}>"
        )

target_link_libraries(projectM_playlist
        PUBLIC
        libprojectM::projectM
        )

if(BUILD_SHARED_LIBS)
    target_compile_definitions(projectM_playlist
            PRIVATE
            projectM_api_EXPORTS
            )

    target_link_libraries(projectM_playlist
            PUBLIC
            ${CMAKE_DL_LIBS}
            )
else()
    target_compile_definitions(projectM_playlist
            PUBLIC
            PROJECTM_STATIC_DEFINE
            )

    set_target_properties(projectM_playlist PROPERTIES
            OUTPUT_NAME $<IF:$<PLATFORM_ID:Windows>,libprojectM_playlist,projectM_playlist>
            FOLDER libprojectM
            )
endif()

include(GenerateExportHeader)

set(PROJECTM_PLAYLIST_EXPORT_HEADER "${CMAKE_CURRENT_BINARY_DIR}/include/libprojectM/projectM_playlist_export.h")

generate_export_header(projectM_playlist
        BASE_NAME projectM_playlist
        EXPORT_FILE_NAME "${PROJECTM_PLAYLIST_EXPORT_HEADER}"
        )

# Adds fallback support to boost if std::filesystem is unavailable.
include(FilesystemSupport.cmake)

add_library(libprojectM::playlist ALIAS projectM_playlist)

install(TARGETS projectM_playlist
        EXPORT libprojectMPlaylist
        LIBRARY DESTINATION "${PROJECTM_LIB_DIR}" COMPONENT Runtime
        RUNTIME DESTINATION "${PROJECTM_LIB_DIR}" COMPONENT Runtime
        ARCHIVE DESTINATION "${PROJECTM_LIB_DIR}" COMPONENT Devel
        PUBLIC_HEADER DESTINATION "${PROJECTM_INCLUDE_DIR}/libprojectM" COMPONENT Devel
        )

install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/include/libprojectM/projectM_playlist_export.h"
        DESTINATION "${PROJECTM_INCLUDE_DIR}/libprojectM"
        COMPONENT Devel
        )

install(DIRECTORY api/libprojectM
        DESTINATION "${PROJECTM_INCLUDE_DIR}"
        COMPONENT Devel
        )


# For use from an installed package (system install, vcpkg, homebrew etc.)
include(CMakePackageConfigHelpers)

write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/libprojectMPlaylist/libprojectMPlaylistConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY AnyNewerVersion
)

configure_package_config_file(libprojectMPlaylistConfig.cmake.in
        "${CMAKE_CURRENT_BINARY_DIR}/libprojectMPlaylist/libprojectMPlaylistConfig.cmake"
        INSTALL_DESTINATION "${PROJECTM_LIB_DIR}/cmake/libprojectMPlaylist"
        PATH_VARS PROJECTM_BIN_DIR PROJECTM_INCLUDE_DIR
        )

install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/libprojectMPlaylist/libprojectMPlaylistConfigVersion.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/libprojectMPlaylist/libprojectMPlaylistConfig.cmake"
        DESTINATION "${PROJECTM_LIB_DIR}/cmake/libprojectMPlaylist"
        COMPONENT Devel
        )

install(EXPORT libprojectMPlaylist
        FILE libprojectMPlaylistTargets.cmake
        NAMESPACE libprojectM::
        DESTINATION "${PROJECTM_LIB_DIR}/cmake/libprojectMPlaylist"
        COMPONENT Devel
        )

# pkg-config export, only supported on UNIX systems.
if(UNIX)
    include(GeneratePkgConfigFiles)

    set(PKGCONFIG_PACKAGE_NAME "libprojectM-playlist")
    set(PKGCONFIG_PACKAGE_DESCRIPTION "projectM Playlist Library")
    set(PKGCONFIG_PACKAGE_REQUIREMENTS_RELEASE "libprojectM")
    set(PKGCONFIG_PACKAGE_REQUIREMENTS_DEBUG "libprojectM-debug")

    generate_pkg_config_files(projectM_playlist libprojectM-playlist)
endif()
