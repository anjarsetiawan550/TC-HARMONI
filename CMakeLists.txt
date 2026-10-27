cmake_minimum_required(VERSION 3.22)

project(HarmonySingerByLR VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.13" CACHE STRING "Minimum macOS deployment target")

# ---------------------------------------------------------------------------
# JUCE is fetched automatically at CI/build time (needs internet access).
# This is why the project is built on GitHub Actions rather than offline.
# ---------------------------------------------------------------------------
include(FetchContent)
FetchContent_Declare(
    JUCE
    GIT_REPOSITORY https://github.com/juce-framework/JUCE.git
    GIT_TAG        7.0.12
    GIT_SHALLOW    TRUE
)
FetchContent_MakeAvailable(JUCE)


# ---------------------------------------------------------------------------
# ASIO (Windows) - supaya driver ASIO muncul di menu Options (Standalone):
# Options > Audio device type > ASIO.
# SDK ASIO dari Steinberg tidak boleh disertakan di repo, jadi diunduh saat
# konfigurasi (di GitHub Actions). Bisa juga memakai SDK lokal:
#   cmake -B build -DLR_ASIO_SDK_DIR=C:/path/ke/asiosdk
# Kalau unduhan gagal: build tetap jalan tanpa ASIO, kecuali
# -DLR_REQUIRE_ASIO=ON (dipakai di workflow) -> build berhenti dengan error.
# ---------------------------------------------------------------------------
option(LR_ENABLE_ASIO  "Aktifkan dukungan ASIO (Windows)" ON)
option(LR_REQUIRE_ASIO "Gagalkan build jika SDK ASIO tidak ditemukan" OFF)
set(LR_ASIO_SDK_URL "https://download.steinberg.net/sdk_downloads/asiosdk_2.3.3_2019-06-14.zip"
    CACHE STRING "URL unduhan SDK ASIO Steinberg")
set(LR_ASIO_SDK_DIR "" CACHE PATH "Folder SDK ASIO (berisi common/ dan host/), kosong = unduh otomatis")

set(LR_ASIO_INCLUDE_DIRS "")
if(WIN32 AND LR_ENABLE_ASIO)
    set(_asio_root "${LR_ASIO_SDK_DIR}")

    if(NOT _asio_root)
        set(_asio_zip  "${CMAKE_BINARY_DIR}/asiosdk.zip")
        set(_asio_dest "${CMAKE_BINARY_DIR}/asiosdk")
        file(GLOB _asio_found "${_asio_dest}/*/common/iasiodrv.h")
        if(NOT _asio_found)
            message(STATUS "Mengunduh SDK ASIO: ${LR_ASIO_SDK_URL}")
            file(DOWNLOAD "${LR_ASIO_SDK_URL}" "${_asio_zip}"
                 STATUS _asio_dl_status TLS_VERIFY ON)
            list(GET _asio_dl_status 0 _asio_dl_code)
            if(_asio_dl_code EQUAL 0)
                file(ARCHIVE_EXTRACT INPUT "${_asio_zip}" DESTINATION "${_asio_dest}")
            else()
                message(WARNING "Gagal mengunduh SDK ASIO: ${_asio_dl_status}")
            endif()
            file(GLOB _asio_found "${_asio_dest}/*/common/iasiodrv.h")
        endif()
        if(_asio_found)
            list(GET _asio_found 0 _asio_header)
            get_filename_component(_asio_common "${_asio_header}" DIRECTORY)
            get_filename_component(_asio_root   "${_asio_common}" DIRECTORY)
        endif()
    endif()

    if(_asio_root AND EXISTS "${_asio_root}/common/iasiodrv.h")
        set(LR_ASIO_INCLUDE_DIRS
            "${_asio_root}/common" "${_asio_root}/host" "${_asio_root}/host/pc")
        message(STATUS "ASIO aktif, SDK: ${_asio_root}")
    elseif(LR_REQUIRE_ASIO)
        message(FATAL_ERROR "SDK ASIO tidak ditemukan/gagal diunduh (LR_REQUIRE_ASIO=ON).")
    else()
        message(WARNING "SDK ASIO tidak ditemukan - build tanpa ASIO.")
    endif()
endif()

# ---------------------------------------------------------------------------
# Embed the GUI assets (knobs, buttons, LEDs, background) as BinaryData
# ---------------------------------------------------------------------------
file(GLOB RESOURCE_FILES ${CMAKE_SOURCE_DIR}/Resources/*.png)

juce_add_binary_data(HarmonySingerAssets
    SOURCES
        ${RESOURCE_FILES}
)
set_target_properties(HarmonySingerAssets PROPERTIES POSITION_INDEPENDENT_CODE TRUE)

# ---------------------------------------------------------------------------
# The plugin target
# ---------------------------------------------------------------------------
juce_add_plugin(HarmonySingerByLR
    COMPANY_NAME                "LR"
    PLUGIN_MANUFACTURER_CODE    LRpl
    PLUGIN_CODE                 HmSg
    FORMATS                     VST3 Standalone
    PRODUCT_NAME                "TC HELICON by LR"
    IS_SYNTH                    FALSE
    NEEDS_MIDI_INPUT            FALSE
    NEEDS_MIDI_OUTPUT           FALSE
    IS_MIDI_EFFECT              FALSE
    EDITOR_WANTS_KEYBOARD_FOCUS FALSE
    COPY_PLUGIN_AFTER_BUILD     FALSE
    VST3_CATEGORIES             Fx "Pitch Shift"
    ICON_BIG                    "${CMAKE_SOURCE_DIR}/Icons/icon_big.png"
    ICON_SMALL                  "${CMAKE_SOURCE_DIR}/Icons/icon_small.png"
)

juce_generate_juce_header(HarmonySingerByLR)

target_sources(HarmonySingerByLR
    PRIVATE
        Source/PluginProcessor.cpp
        Source/PluginEditor.cpp
        Source/dsp/YinPitchDetector.cpp
        Source/dsp/PsolaPitchShifter.cpp
        Source/dsp/ChordDetector.cpp
        Source/dsp/HarmonyEngine.cpp
        Source/dsp/AdaptiveTone.cpp
)

target_compile_definitions(HarmonySingerByLR
    PUBLIC
        JUCE_WEB_BROWSER=0
        JUCE_USE_CURL=0
        JUCE_VST3_CAN_REPLACE_VST2=0
        JUCE_DISPLAY_SPLASH_SCREEN=0
)

if(LR_ASIO_INCLUDE_DIRS)
    target_compile_definitions(HarmonySingerByLR PUBLIC JUCE_ASIO=1)
    target_include_directories(HarmonySingerByLR PUBLIC ${LR_ASIO_INCLUDE_DIRS})
endif()

target_link_libraries(HarmonySingerByLR
    PRIVATE
        HarmonySingerAssets
        juce::juce_audio_utils
        juce::juce_dsp
    PUBLIC
        juce::juce_recommended_config_flags
        juce::juce_recommended_lto_flags
        juce::juce_recommended_warning_flags
)
