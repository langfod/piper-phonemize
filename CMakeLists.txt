cmake_minimum_required(VERSION 3.14)

set(CMAKE_VERBOSE_MAKEFILE off)

project(
    piper_phonemize
    VERSION 1.2.0
    DESCRIPTION "Phonemization library for Piper text to speech system"
    HOMEPAGE_URL "https://github.com/rhasspy/piper-phonemize"
    LANGUAGES CXX
)

if(MSVC)
    # Force compiler to use UTF-8 for IPA constants
    add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")

elseif(NOT APPLE)
    # Linux flags
    string(APPEND CMAKE_CXX_FLAGS " -Wall -Wextra -Wl,-rpath,'$ORIGIN'")
    string(APPEND CMAKE_C_FLAGS " -Wall -Wextra")
endif()

add_library(
    piper_phonemize STATIC
    src/phonemize.cpp
    src/phoneme_ids.cpp
    src/tashkeel.cpp
    src/shared.cpp
)

set_target_properties(piper_phonemize PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
)

# ---- onnxruntime ---

# Look for onnxruntime files in <root>/lib
if(NOT DEFINED ONNXRUNTIME_DIR)
    if(NOT DEFINED ONNXRUNTIME_VERSION)
        set(ONNXRUNTIME_VERSION "1.14.1")
    endif()

    if(WIN32)
        # Windows x86-64
        set(ONNXRUNTIME_PREFIX "onnxruntime-win-x64-${ONNXRUNTIME_VERSION}")
        set(ONNXRUNTIME_EXT "zip")
    elseif (APPLE)
        if(CMAKE_SYSTEM_PROCESSOR STREQUAL x86_64)
            # MacOS x86-64
            set(ONNXRUNTIME_PREFIX "onnxruntime-osx-x86_64-${ONNXRUNTIME_VERSION}")
        elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL arm64)
            # MacOS Apple Silicon
            set(ONNXRUNTIME_PREFIX "onnxruntime-osx-arm64-${ONNXRUNTIME_VERSION}")
        else()
            message(FATAL_ERROR "Unsupported architecture for onnxruntime")
        endif()

        set(ONNXRUNTIME_EXT "tgz")
    else()
        if(CMAKE_SYSTEM_PROCESSOR STREQUAL x86_64)
            # Linux x86-64
            set(ONNXRUNTIME_PREFIX "onnxruntime-linux-x64-${ONNXRUNTIME_VERSION}")
        elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL aarch64)
            # Linux ARM 64-bit
            set(ONNXRUNTIME_PREFIX "onnxruntime-linux-aarch64-${ONNXRUNTIME_VERSION}")
        elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL armv7l)
            # Linux ARM 32-bit
            set(ONNXRUNTIME_PREFIX "onnxruntime-linux-arm32-${ONNXRUNTIME_VERSION}")
            set(ONNXRUNTIME_URL "https://github.com/synesthesiam/prebuilt-apps/releases/download/v1.0/onnxruntime-linux-arm32-${ONNXRUNTIME_VERSION}.tgz")
        else()
            message(FATAL_ERROR "Unsupported architecture for onnxruntime")
        endif()

        set(ONNXRUNTIME_EXT "tgz")
    endif()

    if(NOT DEFINED ONNXRUNTIME_URL)
        set(ONNXRUNTIME_URL "https://github.com/microsoft/onnxruntime/releases/download/v${ONNXRUNTIME_VERSION}/${ONNXRUNTIME_PREFIX}.${ONNXRUNTIME_EXT}")
    endif()

    set(ONNXRUNTIME_FILENAME "${ONNXRUNTIME_PREFIX}.${ONNXRUNTIME_EXT}")
    set(ONNXRUNTIME_DIR "${CMAKE_CURRENT_LIST_DIR}/lib/${ONNXRUNTIME_PREFIX}")

    if(NOT EXISTS "${ONNXRUNTIME_DIR}")
        if(NOT EXISTS "download/${ONNXRUNTIME_FILENAME}")
            # Download onnxruntime release
            message("Downloading ${ONNXRUNTIME_URL}")
            file(DOWNLOAD "${ONNXRUNTIME_URL}" "download/${ONNXRUNTIME_FILENAME}")
        endif()

        # Extract .zip or .tgz to a directory like lib/onnxruntime-linux-x64-1.14.1/
        file(ARCHIVE_EXTRACT INPUT "download/${ONNXRUNTIME_FILENAME}" DESTINATION "${CMAKE_CURRENT_LIST_DIR}/lib")
    endif()
endif()

# ---- espeak-ng ---

# Require local espeak-ng submodule. Do not download or fallback to the
# older archive; fail if it's missing. If you intend to use a prebuilt
# espeak-ng install, set ESPEAK_NG_DIR accordingly.
if(NOT DEFINED ESPEAK_NG_DIR)
    if(EXISTS "${PROJECT_SOURCE_DIR}/external/espeak-ng/CMakeLists.txt")
        set(ESPEAK_NG_DIR "${PROJECT_SOURCE_DIR}/external/espeak-ng/src")
        add_subdirectory(${PROJECT_SOURCE_DIR}/external/espeak-ng ${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build)
    else()
        message(FATAL_ERROR "espeak-ng submodule not found. Initialize the submodule or set -DESPEAK_NG_DIR to the install directory of espeak-ng.")
    endif()
endif()


# ---- Declare library ----

target_include_directories(
    piper_phonemize PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
    ${ESPEAK_NG_DIR}/include
    ${ONNXRUNTIME_DIR}/include
)

target_link_directories(
    piper_phonemize PUBLIC
    ${ESPEAK_NG_DIR}/lib
    ${ONNXRUNTIME_DIR}/lib
)

# ensure linker finds external espeak-ng internal libraries (ucd) - Release paths
link_directories(${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Release)

if(TARGET espeak-ng)
    if(MSVC)
        # Silence CRT deprecation warnings for in-tree espeak-ng builds
        target_compile_definitions(espeak-ng PRIVATE _CRT_SECURE_NO_WARNINGS)
    endif()
    # Ensure consumers see the same export macro (propagate to INTERFACE)
    target_compile_definitions(espeak-ng INTERFACE LIBESPEAK_NG_EXPORT=1)
    if(TARGET ucd)
        # If espeak-ng is built in-tree and the ucd target exists, link it publicly
        # so that the dependency propagates to executables that link piper_phonemize.
        target_link_libraries(piper_phonemize PUBLIC ucd)
    endif()
    # ensure link path finds espeak-ng internal sibling libraries (ucd, etc.)
    target_link_directories(piper_phonemize PUBLIC
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Release>"
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Debug>"
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Release>"
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Debug>"
        )
endif()

if(NOT TARGET espeak-ng)
    # If espeak-ng was not built in-tree (e.g., using an external install),
    # the espeak-ng INTERFACE compile definition may be unavailable; define
    # the export macro so headers don't default to __declspec(dllimport).
    target_compile_definitions(piper_phonemize PUBLIC LIBESPEAK_NG_EXPORT=1)
endif()

if(WIN32 AND TARGET espeak-ng)
    # try to find ucd.lib built by espeak-ng in build tree and add if found
    find_library(UCD_LIB_RELEASE NAMES ucd
        PATHS "${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Release"
        NO_DEFAULT_PATH)
    find_library(UCD_LIB_DEBUG NAMES ucd
        PATHS "${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Debug"
        NO_DEFAULT_PATH)
    # no alt external download paths supported

    # If find_library didn't work at configure time, try linking directly to expected build paths
    if(UCD_LIB_RELEASE)
        target_link_libraries(piper_phonemize PUBLIC ${UCD_LIB_RELEASE})
    elseif(UCD_LIB_DEBUG)
        target_link_libraries(piper_phonemize PUBLIC ${UCD_LIB_DEBUG})
    else()
        message(WARNING "ucd lib not found by find_library; attempting path-based linking to ucd.lib in espeak-ng build directories")
        # try generator-expressions for config-specific paths
        target_link_libraries(piper_phonemize PUBLIC
            $<IF:$<CONFIG:Debug>,${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Debug/ucd.lib,${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Release/ucd.lib>
            $<IF:$<CONFIG:Debug>,${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Debug/ucd.lib,${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Release/ucd.lib>)
    endif()
endif()

if(WIN32 AND NOT TARGET espeak-ng)
    message(FATAL_ERROR "espeak-ng is not configured as an in-tree target; please initialize `external/espeak-ng` submodule and re-run CMake or set -DESPEAK_NG_DIR to a valid install.")
endif()

target_link_libraries(
    piper_phonemize PUBLIC
    espeak-ng
    onnxruntime
)

target_compile_features(piper_phonemize PUBLIC cxx_std_17)

# ---- Declare executable ----

add_executable(piper_phonemize_exe src/main.cpp src/phoneme_ids.cpp)

if(NOT WIN32)
    set_property(TARGET piper_phonemize_exe PROPERTY OUTPUT_NAME piper_phonemize)
endif()

target_compile_features(piper_phonemize_exe PUBLIC cxx_std_17)

target_include_directories(
    piper_phonemize_exe PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
    ${ESPEAK_NG_DIR}/include
)

target_link_directories(
    piper_phonemize_exe PUBLIC
    ${ESPEAK_NG_DIR}/lib
)

target_link_libraries(piper_phonemize_exe PUBLIC
    piper_phonemize
    espeak-ng
)

if(TARGET ucd)
    # If espeak-ng is built in-tree and ucd target exists, ensure the exe links ucd
    target_link_libraries(piper_phonemize_exe PRIVATE ucd)
endif()

if(TARGET espeak-ng)
    target_link_directories(piper_phonemize_exe PUBLIC
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Release>"
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Debug>"
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Release>"
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/external_espeak_ng_build/src/ucd-tools/Debug>"
        )
endif()

# ---- Declare test ----



# ---- Declare install targets ----

include(GNUInstallDirs)

install(
    TARGETS piper_phonemize
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(
    DIRECTORY ${CMAKE_SOURCE_DIR}/src/
    DESTINATION include/piper-phonemize
    FILES_MATCHING
    PATTERN "*.h"
    PATTERN "*.hpp")

install(
    TARGETS piper_phonemize_exe
    ARCHIVE DESTINATION ${CMAKE_INSTALL_BINDIR})

install(
    FILES ${CMAKE_SOURCE_DIR}/etc/libtashkeel_model.ort
    TYPE DATA)

# Dependencies
install(
    DIRECTORY ${ESPEAK_NG_DIR}/
    DESTINATION ${CMAKE_INSTALL_PREFIX})

install(
    DIRECTORY ${ONNXRUNTIME_DIR}/include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(
    DIRECTORY ${ONNXRUNTIME_DIR}/lib/
    DESTINATION ${CMAKE_INSTALL_LIBDIR})
