cmake_minimum_required(VERSION 3.10)
PROJECT(qucs-suite CXX)

set(QUCS_NAME "qucs-s")

# Prohibit in-source builds
if("${CMAKE_BINARY_DIR}" STREQUAL "${CMAKE_SOURCE_DIR}")
  message(
    FATAL_ERROR
      "QUCS-S: You cannot build in a source directory (or any directory with "
      "CMakeLists.txt file). Please make a build subdirectory. Feel free to "
      "remove CMakeCache.txt and CMakeFiles.")
endif()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

file (STRINGS "${qucs-suite_SOURCE_DIR}/VERSION" QUCS_VERSION)

if(DEFINED CI_VERSION)
    set(PROJECT_VERSION "${CI_VERSION}")
else()
    set(PROJECT_VERSION "${QUCS_VERSION}")
endif()

message(STATUS "Configuring Qucs: VERSION ${PROJECT_VERSION}")

set(GIT "")
if(EXISTS ${CMAKE_SOURCE_DIR}/.git )
  find_package(Git)
  # Get the latest abbreviated commit hash of the working branch
  execute_process(
    COMMAND ${GIT_EXECUTABLE} log --pretty=format:%h -n 1
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_COMMIT_HASH
  )
  set(GIT ${GIT_COMMIT_HASH})
  message(STATUS "Found Git repository, last commit hash: ${GIT}")
endif()

message(STATUS "${PROJECT_NAME} ${CMAKE_INSTALL_PREFIX} ${qucs-suite_BINARY_DIR}" )

find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets LinguistTools)
set(QT_VERSION ${Qt6Core_VERSION})

message(STATUS "Qt Version: " ${QT_VERSION})

add_definitions(${QT_DEFINITIONS})
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_compile_definitions(QT_NO_DEBUG_OUTPUT)
endif()

# Function to configure embind linkage for WebAssembly executables
# This ensures all Qt applications link against embind for JavaScript interop
# IMPORTANT: This must be called AFTER qt_finalize_target() to avoid CMake signature conflicts
function(configure_wasm_executable target_name)
    if(EMSCRIPTEN)
        message(STATUS "Configuring WebAssembly embind linkage for ${target_name}")

        # Enable embind support via --bind compiler/linker flag
        # This activates Emscripten's C++ to JavaScript binding system
        target_compile_options(${target_name} PRIVATE "SHELL:--bind")
        target_link_options(${target_name} PRIVATE "SHELL:--bind")

        # Enable asyncify to support blocking calls like QDialog::exec()
        # This allows synchronous Qt APIs to work in WebAssembly
        target_link_options(${target_name} PRIVATE "SHELL:-sASYNCIFY")
        target_link_options(${target_name} PRIVATE "SHELL:-sASYNCIFY_STACK_SIZE=65536")

        # Enable assertions for better error messages during development
        target_link_options(${target_name} PRIVATE "SHELL:-sASSERTIONS")

        # Disable optimization for faster builds during development
        target_compile_options(${target_name} PRIVATE -O0)
        target_link_options(${target_name} PRIVATE -O0)

        # Force inclusion of all symbols from Qt6::Core to ensure embind EMSCRIPTEN_BINDINGS
        # static constructors are linked. Qt's embind bindings are in qstdweb.cpp and other
        # WebAssembly platform files, and without --whole-archive these static constructors
        # that register types with embind won't be included.
        target_link_options(${target_name} PRIVATE
            "SHELL:-Wl,--whole-archive"
            "$<TARGET_FILE:Qt6::Core>"
            "SHELL:-Wl,--no-whole-archive"
        )
    endif()
endfunction()

# Simulator backend options
option(WITH_NGSPICE_SHARED "Enable ngspice shared library support" OFF)

# External process categories
option(WITH_EXTERNAL_SIMULATORS "Enable external SPICE simulators (ngspice, xyce)" ON)
option(WITH_EXTERNAL_DATA_TOOLS "Enable external data conversion tools" ON)
option(WITH_EXTERNAL_POSTPROCESSING "Enable Octave/external post-processing" ON)
option(WITH_EXTERNAL_DESIGN_TOOLS "Enable external design utility tools" ON)
option(WITH_EXTERNAL_BUILD_TOOLS "Enable Verilog-A compilation tools" ON)
option(WITH_EXTERNAL_EDITORS "Enable external text editor support" ON)

if(WITH_NGSPICE_SHARED)
    find_library(NGSPICE_LIBRARY NAMES ngspice libngspice)
    find_path(NGSPICE_INCLUDE_DIR ngspice/sharedspice.h)

    if(NGSPICE_LIBRARY AND NGSPICE_INCLUDE_DIR)
        message(STATUS "Found ngspice shared library: ${NGSPICE_LIBRARY}")
        message(STATUS "Found ngspice headers: ${NGSPICE_INCLUDE_DIR}")
        set(NGSPICE_SHARED 1)
    else()
        message(WARNING "Ngspice shared library not found. WITH_NGSPICE_SHARED disabled.")
        set(WITH_NGSPICE_SHARED OFF)
        set(NGSPICE_SHARED 0)
    endif()
else()
    set(NGSPICE_SHARED 0)
endif()

# Configure external process support flags
if(WITH_EXTERNAL_SIMULATORS)
    set(EXTERNAL_SIMULATORS 1)
    message(STATUS "External simulator support enabled")
else()
    set(EXTERNAL_SIMULATORS 0)
    message(STATUS "External simulator support disabled")
endif()

if(WITH_EXTERNAL_DATA_TOOLS)
    set(EXTERNAL_DATA_TOOLS 1)
    message(STATUS "External data tools enabled")
else()
    set(EXTERNAL_DATA_TOOLS 0)
    message(STATUS "External data tools disabled")
endif()

if(WITH_EXTERNAL_POSTPROCESSING)
    set(EXTERNAL_POSTPROCESSING 1)
    message(STATUS "External post-processing tools enabled")
else()
    set(EXTERNAL_POSTPROCESSING 0)
    message(STATUS "External post-processing tools disabled")
endif()

if(WITH_EXTERNAL_DESIGN_TOOLS)
    set(EXTERNAL_DESIGN_TOOLS 1)
    message(STATUS "External design tools enabled")
else()
    set(EXTERNAL_DESIGN_TOOLS 0)
    message(STATUS "External design tools disabled")
endif()

if(WITH_EXTERNAL_BUILD_TOOLS)
    set(EXTERNAL_BUILD_TOOLS 1)
    message(STATUS "External build tools enabled")
else()
    set(EXTERNAL_BUILD_TOOLS 0)
    message(STATUS "External build tools disabled")
endif()

if(WITH_EXTERNAL_EDITORS)
    set(EXTERNAL_EDITORS 1)
    message(STATUS "External editor support enabled")
else()
    set(EXTERNAL_EDITORS 0)
    message(STATUS "External editor support disabled")
endif()

enable_testing()

add_subdirectory( qucs )
#add_subdirectory( converter )
add_subdirectory( qucs-activefilter )
add_subdirectory( qucs-attenuator )
#add_subdirectory( qucs-doc )
add_subdirectory( qucs-filter )
add_subdirectory( library )
add_subdirectory( qucs-transcalc )
add_subdirectory( qucs-powercombining )
add_subdirectory( qucs-s-spar-viewer )
#add_subdirectory( examples )
if(EXISTS ${CMAKE_SOURCE_DIR}/qucsator_rf/CMakeLists.txt)
    add_subdirectory(qucsator_rf)
else()
    message("Qucsator RF submodule not found. Please update submodules")
endif()

# Add test suite
add_subdirectory(tests)

IF (UPDATE_TRANSLATIONS)
  file(GLOB_RECURSE FILES_TO_TRANSLATE "*.cpp" "*.h" "*.ui")
ENDIF (UPDATE_TRANSLATIONS)
add_subdirectory( translations )

install(DIRECTORY "examples" DESTINATION "share/${QUCS_NAME}")

install(FILES contrib/io.github.ra3xdh.qucs_s.metainfo.xml DESTINATION ${CMAKE_INSTALL_PREFIX}/share/metainfo)

#
# Custom uninstall target
#
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/uninstall.cmake"
    IMMEDIATE @ONLY)

add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/uninstall.cmake)


