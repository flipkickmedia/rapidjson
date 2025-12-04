cmake_minimum_required(VERSION 3.23)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules)

set(LIB_MAJOR_VERSION "1")
set(LIB_MINOR_VERSION "1")
set(LIB_PATCH_VERSION "0")
set(LIB_VERSION_STRING "${LIB_MAJOR_VERSION}.${LIB_MINOR_VERSION}.${LIB_PATCH_VERSION}")

project(RapidJSON VERSION "${LIB_VERSION_STRING}" LANGUAGES CXX)

# If the parent project hasn't specified a C++ standard, default to C++11.
# Consumers can override this by setting CMAKE_CXX_STANDARD before adding this project.
if(NOT CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 11)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_CXX_EXTENSIONS OFF)
endif()

# Compile in release-with-debug-info mode by default
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING
            "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel."
            FORCE)
endif()

# Build all binaries in a separate directory
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

option(RAPIDJSON_BUILD_DOC "Build rapidjson documentation." ON)
option(RAPIDJSON_BUILD_EXAMPLES "Build rapidjson examples." ON)
option(RAPIDJSON_BUILD_TESTS "Build rapidjson perftests and unittests." ON)
option(RAPIDJSON_BUILD_THIRDPARTY_GTEST
        "Use gtest installation in `thirdparty/gtest` by default if available" OFF)

option(RAPIDJSON_BUILD_ASAN "Build rapidjson with address sanitizer (gcc/clang)" OFF)
option(RAPIDJSON_BUILD_UBSAN "Build rapidjson with undefined behavior sanitizer (gcc/clang)" OFF)

option(RAPIDJSON_ENABLE_INSTRUMENTATION_OPT "Build rapidjson with -march or -mcpu options" ON)

option(RAPIDJSON_HAS_STDSTRING "" OFF)
if(RAPIDJSON_HAS_STDSTRING)
    add_definitions(-DRAPIDJSON_HAS_STDSTRING)
endif()

option(RAPIDJSON_USE_MEMBERSMAP "" OFF)
if(RAPIDJSON_USE_MEMBERSMAP)
    add_definitions(-DRAPIDJSON_USE_MEMBERSMAP=1)
endif()

find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
    set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
    if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Qunused-arguments -fcolor-diagnostics")
    endif()
endif()

find_program(VALGRIND_FOUND valgrind)

if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if(RAPIDJSON_ENABLE_INSTRUMENTATION_OPT AND NOT CMAKE_CROSSCOMPILING)
        if(CMAKE_SYSTEM_PROCESSOR STREQUAL "powerpc"
                OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc"
                OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64"
                OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64le")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mcpu=native")
        else()
            # NOTE: x86 uses -march=native; other archs may differ.
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
        endif()
    endif()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Werror")
    set(EXTRA_CXX_FLAGS -Weffc++ -Wswitch-default -Wfloat-equal -Wconversion -Wsign-conversion)

    if (RAPIDJSON_BUILD_ASAN)
        if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS "4.8.0")
            message(FATAL_ERROR "GCC < 4.8 doesn't support the address sanitizer")
        else()
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
        endif()
    endif()
    if (RAPIDJSON_BUILD_UBSAN)
        if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS "4.9.0")
            message(FATAL_ERROR "GCC < 4.9 doesn't support the undefined behavior sanitizer")
        else()
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined")
        endif()
    endif()
elseif (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    if(NOT CMAKE_CROSSCOMPILING)
        if(CMAKE_SYSTEM_PROCESSOR STREQUAL "powerpc"
                OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc"
                OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64"
                OR CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64le")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mcpu=native")
        else()
            # NOTE: x86 uses -march=native; other archs may differ.
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
        endif()
    endif()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Werror -Wno-missing-field-initializers")
    set(EXTRA_CXX_FLAGS -Weffc++ -Wswitch-default -Wfloat-equal -Wconversion -Wimplicit-fallthrough)

    if (RAPIDJSON_BUILD_ASAN)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
    endif()
    if (RAPIDJSON_BUILD_UBSAN)
        if (CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined-trap -fsanitize-undefined-trap-on-error")
        else()
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined")
        endif()
    endif()
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    add_definitions(-D_CRT_SECURE_NO_WARNINGS=1)
    add_definitions(-DNOMINMAX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc")

    # Always compile with /WX
    if(CMAKE_CXX_FLAGS MATCHES "/WX-")
        string(REGEX REPLACE "/WX-" "/WX" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /WX")
    endif()
elseif (CMAKE_CXX_COMPILER_ID MATCHES "XL")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -qarch=auto")
endif()

# ===== Install layout (now honors CMAKE_INSTALL_PREFIX) =====

include(GNUInstallDirs)

# Relative install dirs (all prefixed by CMAKE_INSTALL_PREFIX at install time)
set(INCLUDE_INSTALL_DIR "${CMAKE_INSTALL_INCLUDEDIR}" CACHE PATH
        "The directory the headers are installed in")
set(LIB_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}" CACHE PATH
        "Directory where lib will install")
set(DOC_INSTALL_DIR "${CMAKE_INSTALL_DATAROOTDIR}/doc/${PROJECT_NAME}" CACHE PATH
        "Path to the documentation")

# Where to install CMake config files, relative to prefix
set(CMAKECONFIG_INSTALL_DIR
        "${LIB_INSTALL_DIR}/cmake/${PROJECT_NAME}"
        CACHE PATH "The directory cmake files are installed in")

# ------------------------------------------------------------

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

if(RAPIDJSON_BUILD_DOC)
    add_subdirectory(doc)
endif()

add_custom_target(travis_doc)
add_custom_command(TARGET travis_doc
        COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/travis-doxygen.sh
        POST_BUILD)

if(RAPIDJSON_BUILD_EXAMPLES)
    add_subdirectory(example)
endif()

if(RAPIDJSON_BUILD_TESTS)
    if(MSVC11)
        # required for VS2012 due to missing support for variadic templates
        add_definitions(-D_VARIADIC_MAX=10)
    endif()
    add_subdirectory(test)
    include(CTest)
endif()

# pkg-config
if(UNIX OR CYGWIN)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}.pc.in
            ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
            @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.pc
            DESTINATION "${LIB_INSTALL_DIR}/pkgconfig"
            COMPONENT pkgconfig)
endif()

install(FILES readme.md
        DESTINATION "${DOC_INSTALL_DIR}"
        COMPONENT doc)

# Add an interface target to export it
add_library(RapidJSON INTERFACE)

target_include_directories(RapidJSON INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${INCLUDE_INSTALL_DIR}>)

install(DIRECTORY include/rapidjson
        DESTINATION "${INCLUDE_INSTALL_DIR}"
        COMPONENT dev)

install(DIRECTORY example/
        DESTINATION "${DOC_INSTALL_DIR}/examples"
        COMPONENT examples
        # Exclude build-system junk from in-source builds
        PATTERN "CMakeFiles" EXCLUDE
        PATTERN "Makefile" EXCLUDE
        PATTERN "cmake_install.cmake" EXCLUDE)

# Provide config and version files to be used by other applications
# ===============================

# Export package for use from the build tree
export(PACKAGE ${PROJECT_NAME})

# --- Build tree config files ---
set(CONFIG_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(CONFIG_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(${PROJECT_NAME}_INCLUDE_DIR "\${${PROJECT_NAME}_SOURCE_DIR}/include")

include(CMakePackageConfigHelpers)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}ConfigVersion.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake @ONLY)

# --- Install tree config files ---

# Compute include-dir path relative to the CMake config dir, both under the
# same CMAKE_INSTALL_PREFIX.
file(RELATIVE_PATH REL_INCLUDE_DIR
        "${CMAKE_INSTALL_PREFIX}/${CMAKECONFIG_INSTALL_DIR}"
        "${CMAKE_INSTALL_PREFIX}/${INCLUDE_INSTALL_DIR}")

set(${PROJECT_NAME}_INCLUDE_DIR "\${${PROJECT_NAME}_CMAKE_DIR}/${REL_INCLUDE_DIR}")
set(CONFIG_SOURCE_DIR)
set(CONFIG_DIR)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}Config.cmake @ONLY)

# Install config + version + export files under CMAKECONFIG_INSTALL_DIR
install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}Config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
        DESTINATION "${CMAKECONFIG_INSTALL_DIR}"
        COMPONENT dev)

install(TARGETS RapidJSON EXPORT RapidJSON-targets)

install(EXPORT RapidJSON-targets
        DESTINATION "${CMAKECONFIG_INSTALL_DIR}"
        COMPONENT dev)
